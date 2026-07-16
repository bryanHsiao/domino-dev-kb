# DQL view 查詢報錯 4854 與默默回 0 筆：診斷階梯與 design catalog 生命週期

> 案例日期：2026-07-16
> 來源：[DQL view 日期直欄型別案例](2026-07-16-dql-view-date-column.md) 單點測試過程中的副產物——六輪測試踩出一條完整的故障診斷路徑
> 環境：Domino 伺服器（ap02/TheNet），LotusScript `NotesDominoQuery`

---

## 結論（TL;DR）

1. `'view'.column` 語法失敗時**錯誤碼一律是 4854**，但**錯誤訊息分三層**，每層對症下藥的方法不同（見下方診斷階梯）。
2. **view 設計變更後 `updall -d` 可能直接失敗**（`Named Object corrupt`），catalog 半殘時查詢行為會變得**不一致**：有的 view 報錯、有的默默回 0 筆——正是「有些抓得到有些沒抓到，蠻怪的」的另一種成因。
3. 查詢**回 0 筆且無錯誤**時不要急著怪語法，照下面的檢查清單排查：資料是否存在 → view index 是否需要 refresh → catalog 是否過期。

---

## 4854 三層診斷階梯（實測原文）

| 層 | 4854 錯誤訊息（runtime 原文） | 病因 | 解法 |
|---|---|---|---|
| 1 | `Error validating view column name - ['view'.$123]`<br>`.. invalid view name or database needs to be cataloged via updall -e` | DB 沒有 design catalog | `load updall <db> -e` |
| 2 | `View never built [viewName] - query term will fail, aborting` | view 建好後**從未被開啟過**、index 不存在——DQL 不會代建 | 開一次 view，或 `load updall <db> -t <view>` |
| 3 | `Partial TIMEDATEs NOT supported with view column lookup, specify a full TIMEDATE` | `@dt` 只給日期、沒給完整 timestamp | `@dt('2024-01-01T00:00:00+08:00')` 完整值 |

三層都是**硬報錯**、查詢直接中止，不會默默 fallback。錯誤在驗證階段逐層檢查：catalog → 查詢語法 → view index，所以修掉一層才會看到下一層。

## Design catalog 生命週期的坑

實測時間軸（六輪測試濃縮）：

| 時間點 | 操作 | 結果 |
|---|---|---|
| 15:44 | `load updall <db> -e` | ✅ 建 catalog 成功（17 view designs cataloged） |
| 16:2x | Designer 從 DXL 重新 sync 5 個 view 設計 | catalog 未同步更新（過期） |
| 16:29 | `load updall <db> -d`（官方指定的 refresh 指令） | ❌ **`Design Catalog refresh/rebuild FAILED ... DesignCatalog::LoadColumnData ... Named Object corrupt`** |
| 16:30 | catalog 半殘狀態下查詢 | **行為不一致**：某個 view 報「needs to be cataloged」、其他 view 默默回 0 筆 |
| 16:33 | `load updall <db> -e` 重建 | ✅ 修復（無需 compact；若 `-e` 也失敗再 `load compact <db> -c` 後重試） |

**要點**：

- 官方文件說 view 設計變更後跑 `updall -d` 即可，但實測 `-d` 在 catalog 已過期的狀態下**可能直接損毀失敗**，此時要用 `-e` 重建。
- catalog 半殘時**不是全部報錯**——有的 view 查得到、有的報錯、有的回 0 筆。正式環境若曾在建 catalog 後改過 view 設計（很常見），這就是「時好時壞」的高嫌疑成因。
- **維運建議**：把「改 view 設計 → 跑 `updall -d`、失敗就 `-e`」納入部署 SOP；DQL 有用到 `'view'.column` 語法的 DB 尤其要注意。

## 查詢回 0 筆且無錯誤時的檢查清單

實測中還遇過「查詢語法正確、無錯誤、就是 0 筆」的狀況，照這個順序排查（每步都用得上 Explain，判讀方式見下一節）：

1. **資料真的存在嗎？** 跑一個不經 view 的純欄位查詢當對照組（如 `Form = 'frmDoc'`）——0 筆的話問題在資料層，不在 view。
2. **view index 是新的嗎？** LotusScript 設 `dq.RefreshViews = True` 讓 DQL 查詢時強制重整 view index；或先開一次 view / F9。實測曾出現 view 有資料但 `in('view')` 回 `Entries 0`，開啟 RefreshViews 後恢復正常。
3. **catalog 過期了嗎？** view 設計改過之後 catalog 舊了，跑 `updall -d`（失敗就 `-e`）。
4. **view selection 篩掉了嗎？** 非 `SELECT @All` 的 view，被 selection 排除的文件無聲消失（見[第一篇案例](2026-07-16-dql-view-date-column.md)）。
5. **型別/時區對了嗎？** 混型直欄與 @dt 時區是最後兩個嫌疑犯（同見第一篇）。

## 用 Explain 看成本與行為

`NotesDominoQuery.Explain(query)` 回傳執行計畫，除錯與效能分析都靠它：

```
0. AND    (childct 3) Prep 0.0 msecs, Exec 2.670 msecs, ScannedDocs 0, Entries 6, FoundDocs 6
   1. IN  ... 'vwTestTyped' View NoteID Extract estimated cost = 20
   1. 'vwTestTyped'.$123 >= 2023-12-31T16:00:00.00Z View Column Search estimated cost = 10
      Prep 0.500 msecs, Exec 1.7 msecs, ScannedDocs 0, Entries 6, FoundDocs 6
```

| 欄位 | 意義 |
|---|---|
| `estimated cost` | 優化器對該節點的估算成本（決定執行順序） |
| `Prep / Exec msecs` | 實際準備/執行耗時 |
| `ScannedDocs` | **NSF 全庫掃描的文件數——0 代表純走索引**，數字大代表 fallback 掃描、效能警訊 |
| `Entries` | view index 掃過的 entry 數 |
| `FoundDocs` | 該節點命中的文件數 |
| `View Column Search` / `View NoteID Extract` | 節點型態：走 view 索引搜尋 / 抽 view 的 NoteID 清單 |

另外可見 `@dt('...+08:00')` 在計畫中被正規化成 UTC（`2023-12-31T16:00:00Z`）——時區換算行為一目了然。

## LotusScript 陷阱備忘（測試中實際踩到）

- `doc.GetItemValueString("field")` 遇**非文字型** item 丟 error 13 Type Mismatch——混型資料的 DB 裡列印欄位值一律用 `Cstr(doc.GetItemValue("field")(0))` + `HasItem` 防呆。
- 錯誤 handler 一定要印 `Erl`（行號），Domino runtime 錯誤 modal 不講哪行。
- 從零建的 view DXL 要帶 `bgcolor='white'` 等外觀屬性，否則 Notes client 顯示全黑。

## 參考資料

- [Design catalog — HCL Domino Designer 14.5.0](https://help.hcl-software.com/dom_designer/14.5.0/basic/dql_design_catalog.html)
- [DQL syntax — HCL Domino Designer 12.0.2](https://help.hcl-software.com/dom_designer/12.0.2/basic/dql_syntax.html)
- [第一篇案例：DQL view 日期直欄型別查證](2026-07-16-dql-view-date-column.md)
- 測試素材：repo 的 `odp-assets/`（4 views + 2 agents，可重複部署驗證）
