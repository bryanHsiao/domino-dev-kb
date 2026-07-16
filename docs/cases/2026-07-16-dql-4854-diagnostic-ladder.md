# DQL view 查詢「Domino Query 執行時錯誤」與默默回 0 筆：診斷階梯與 design catalog 生命週期

> 案例日期：2026-07-16
> 來源：[DQL view 日期直欄型別案例](2026-07-16-dql-view-date-column.md) 單點測試過程中的副產物——六輪測試踩出一條完整的故障診斷路徑
> 環境：Domino 伺服器（ap02/TheNet），LotusScript `NotesDominoQuery`

---

## 結論（TL;DR）

1. `'view'.column` 語法失敗時，畫面/日誌上看到的是「**Domino Query 執行時錯誤:**」開頭的訊息文字——**診斷靠訊息內容分層**，每層對症下藥的方法不同（見下方診斷階梯）。程式端捕捉到的錯誤碼一律是 4854，但那只有印出 `Err` 的 handler 才看得到、且不帶診斷資訊。
2. **view 設計變更後 `updall -d` 可能直接失敗**（`Named Object corrupt`），catalog 半殘時查詢行為會變得**不一致**：有的 view 報錯、有的默默回 0 筆——正是「有些抓得到有些沒抓到，蠻怪的」的另一種成因。
3. 查詢**回 0 筆且無錯誤**時不要急著怪語法，照下面的檢查清單排查：資料是否存在 → view index 是否需要 refresh → catalog 是否過期。

---

## 你看到的是訊息文字，不是 4854

查詢失敗時，**使用者實際看到的**是「Domino Query 執行時錯誤:」開頭的多行文字（無 handler 時 Notes 跳的 modal、或 handler 印出的 `Error$`）。「4854」這個數字**只存在於程式端的 `Err`**——是 LotusScript 層的統包錯誤碼，定義在 Notes/Domino 安裝目錄的 `lsxbeerr.lss`（實機 12.0.2 摘錄）：

```lotusscript
Public Const lsERR_NOTES_DQUERY_INTERNAL  = 4852   ' DQL 內部錯誤
Public Const lsERR_NOTES_DQUERY_PLANNING  = 4853   ' 查詢規劃階段錯誤
Public Const lsERR_NOTES_DQUERY_EXECUTION = 4854   ' 查詢執行錯誤（最常見）
Public Const lsERR_NOTES_DQUERY_TIMEOUT   = 4855   ' 查詢逾時
Public Const lsERR_NOTES_DQUERY_NOQUERY   = 4856   ' 沒有查詢字串
```

`NotesDominoQuery.Execute / Explain` 失敗時 `Err` 一律回 4854，**錯誤碼本身幾乎不帶診斷資訊**；真正的細節在錯誤文字（`Error$`）裡，由 DQL 引擎組成固定四段：

```
Domino Query 執行時錯誤:                          ← LS 對 4854 的訊息前綴
Query is not understandable - 語法錯誤            ← ① 引擎分類行
Partial TIMEDATEs NOT supported with view...      ← ② 詳細原因（診斷階梯看這行）
in ('vwTestTyped') and ...                        ← ③ 原查詢字串回顯
(Call hint: NSFCalls::NSFItemInfo, Core call #0)  ← ④ 內部出錯的 API 呼叫點
```

**實務守則**：error handler 一定要印 `Error$` 全文（只印 `Err` 等於什麼都沒印）；下方診斷階梯的分層依據就是②那一行。

**重現實驗（agtRepro4854，2026-07-16 實測）**——五種刻意錯誤的 `Err` 值：

| 情境 | Err | 訊息重點 |
|---|---|---|
| `@dt` 只給日期 | 4854 | `Partial TIMEDATEs NOT supported...` |
| 不存在的 view | 4854 | `invalid view name or database needs to be cataloged`——**view 名打錯跟缺 catalog 是同一句話**，先檢查名字 |
| 語法錯誤（`and and`） | 4854（**不是** 4853 PLANNING） | `Query is not understandable`，且附插入符號指標行 `.....^.....` 指出錯誤字元位置 |
| 空查詢字串 | 4854（**不是** 4856 NOQUERY） | `Query null or length error or illegal flags` |
| 正確查詢（對照組） | 無錯誤 | 命中 6 筆 |

**結論：LotusScript 實測中所有 DQL 失敗一律回 4854**——4852/4853/4855/4856 常數雖然存在，但一般查詢錯誤根本用不到。錯誤分類只能靠訊息文字，再次印證本節標題。

## 錯誤訊息三層診斷階梯（實測原文）

| 層 | 錯誤訊息第②行（runtime 原文） | 病因 | 解法 |
|---|---|---|---|
| 1 | `Error validating view column name - ['view'.$123]`<br>`.. invalid view name or database needs to be cataloged via updall -e` | DB 沒有 design catalog，**或 view 名稱/直欄名打錯**（實測兩者同一句話） | 先檢查 view 名與直欄名拼字，再 `load updall <db> -e` |
| 2 | `View never built [viewName] - query term will fail, aborting` | view 建好後**從未被開啟過**、index 不存在——DQL 不會代建 | 開一次 view，或 `load updall <db> -t <view>` |
| 3 | `Partial TIMEDATEs NOT supported with view column lookup, specify a full TIMEDATE` | `@dt` 只給日期、沒給完整 timestamp | `@dt('2024-01-01T00:00:00+08:00')` 完整值 |

三層都是**硬報錯**、查詢直接中止，不會默默 fallback。錯誤在驗證階段逐層檢查：catalog → 查詢語法 → view index，所以修掉一層才會看到下一層。

## Design catalog 生命週期的坑

先搞清楚 catalog 是什麼（依 [DQL 正式環境筆記](https://bryanhsiao.github.io/domino-news/posts/dql-production/)）：

- **Domino 11+ 的 catalog 是 NSF 內的持久 state**——「Design Catalog 移到 NSF 內部，存在隱藏的設計元素裡」，「不是 in-memory state。Server 重啟、HTTP task 重啟、JVM 重啟都還在」，「一個 NSF 一輩子只需要建立 catalog 一次，之後永久存在」。（Domino 10.x 是資料目錄下的獨立檔 `GQFdsgn.cat`）
- 因為是 NSF 內的持久物件，**損毀也是持久的**——本案 `updall -d` 失敗後的 `Named Object corrupt` 狀態不會因重啟消失，必須明確重建才會好。

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
- **維運建議（程式內自動處理，優於人工 SOP）**：`NotesDominoQuery` 有 `RefreshDesignCatalog` / `RebuildDesignCatalog` 屬性，照 [DQL 正式環境筆記](https://bryanhsiao.github.io/domino-news/posts/dql-production/)的模式——查詢前設 `RefreshDesignCatalog = True`（catalog 是新的時近乎零成本），若吃到含 `needs to be cataloged via updall -e` 的 4854，改設 `RebuildDesignCatalog = True` 重試一次。注意 **Refresh 無法 bootstrap 全新 NSF**，缺 catalog 時必須用 Rebuild。這樣就不依賴人工 updall；console 的 `updall -d` / `-e` 留作手動維運備援。

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

- [Design catalog — HCL Domino Designer 14.5.1](https://help.hcl-software.com/dom_designer/14.5.1/basic/dql_design_catalog.html)
- [DQL syntax — HCL Domino Designer 14.5.1](https://help.hcl-software.com/dom_designer/14.5.1/basic/dql_syntax.html)
- [DQL 正式環境筆記（catalog 儲存機制與 Refresh/Rebuild 模式）— domino-news](https://bryanhsiao.github.io/domino-news/posts/dql-production/)
- [第一篇案例：DQL view 日期直欄型別查證](2026-07-16-dql-view-date-column.md)
- 測試素材：repo 的 `odp-assets/`（4 views + 2 agents，可重複部署驗證）
