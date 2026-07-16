# DQL view 日期直欄 — 單點測試包

驗證 [docs/cases/2026-07-16-dql-view-date-column.md](../docs/cases/2026-07-16-dql-view-date-column.md) 的結論：
DQL 無自動轉型、混型直欄會部分命中、@dt 的 partial date / 時區行為。

## 內容物

| 檔案 | 說明 |
|---|---|
| `Views/vwTestProd.view` | `$123` = 現行公式 `@If(@IsText(DocDate);DocDate;@Text(@Date(DocDate)))`（地區格式文字） |
| `Views/vwTestTextPadded.view` | `$123` = 補零 `yyyy/mm/dd` 文字（**方案 A**） |
| `Views/vwTestTyped.view` | `$123` = `@If(@IsText(DocDate);@TextToTime(DocDate);DocDate)`（**方案 B**，統一日期型） |
| `Views/vwTestMixed.view` | `$123` = `DocDate` 原樣（**混型，重現症狀**） |
| `Views/vwTestRealClone.view` | 仿真實來文簽辦單 view：selection 非 @All（刻意排除 T6 以實證無聲篩掉）、分類最左欄、`$123` 遞減排序（`resort='ascending'`）、現行文字公式 |
| `Code/Agents/agtSeedTestDocs.lsa` | 種 8 份測試文件（可重複執行，會先清掉舊的） |
| `Code/Agents/agtTestDQL.lsa` | 跑 A~G 七組查詢，結果寫到 `%TEMP%\dql-test-results.txt` |

所有 view 的 selection 都是 `SELECT @All`（官方 view column requirements 的前置條件），
`$123` 直欄皆為最左、遞增排序（primary sorted column 條件）。

## 測試文件（T1~T8）

| # | DocDate | 型別 | 定位 |
|---|---|---|---|
| T1 | 2024/01/01 00:00 | 日期 | 起日邊界（**時區陷阱指標**：UTC 查詢會漏掉它） |
| T2 | 2024/01/15 12:00 | 日期 | 區間中間 |
| T3 | 2024/01/31 23:30 | 日期 | 訖日邊界內 |
| T4 | 2024/02/01 00:00 | 日期 | 區間外（UTC 查詢會誤抓它） |
| T5 | 2023/12/31 20:00 | 日期 | 區間外對照組 |
| T6 | 2024/01/01 09:00 | 日期 | 區間內 |
| T7 | `"2024/01/10"` | **文字** | 混型資料（補零） |
| T8 | `"2024/1/5"` | **文字** | 混型資料（未補零） |

理想命中（2024/01/01 ≤ x < 2024/02/01 本機時間）：**T1, T2, T3, T6, T7, T8 共 6 筆**。

## 預期結果矩陣

| 查詢 | 打在哪個 view | 預期 | 驗證的論點 |
|---|---|---|---|
| A. 文字比較 | vwTestProd（現行） | 視伺服器地區格式而定；T8 幾乎必漏 | @Text 地區格式不可靠、未補零文字範圍比較會錯 |
| B. 文字比較 | vwTestTextPadded | ✅ 6 筆全中 | **方案 A 成立** |
| C. @dt 只給日期 | vwTestTyped | ❌ 報錯 | 官方：view 搜尋不支援 partial date |
| D. @dt 完整但無時區 | vwTestTyped | T1 漏、T4 誤入（5 筆但內容錯） | @dt 無時差 = UTC，差 8 小時 |
| E. @dt 完整 +08:00 | vwTestTyped | ✅ 6 筆全中 | **方案 B 成立**（直欄已統一型別） |
| F. 同 E 的查詢 | vwTestMixed | T7、T8 消失（只剩 4 筆） | **重現工程師症狀**：混型 → 部分命中 |
| H. 文字比較 | vwTestRealClone | 同 A 但 **T6 無聲消失**、不報錯 | **仿真實 view**：selection 非 @All（刻意排除 T6）。官方文件說會報錯，[實測](https://bryanhsiao.github.io/domino-news/posts/dql-pitfalls/)是以 view selection 為查詢範圍 |
| G. Explain | vwTestTyped | 查詢計畫顯示走 view index | 確認不是 fallback 全庫掃描 |

## 執行步驟

1. 建一個空 NSF、在 Designer 開 ODP 同步（source control）
2. 把本資料夾的 `Views/`、`Code/Agents/` 複製進 ODP 對應目錄
   - ⚠️ 兩個 `.lsa` 內的 `replicaid='REPLACE_WITH_DB_REPLICAID'` 要換成該 DB 的 replica ID（抄 ODP 內任一既有 design element 檔的 `replicaid` 屬性）——把 ODP 路徑給 Claude 代做即可
3. Designer 內 sync 推回 NSF、簽署 agent
4. Notes client 開該 DB → 動作選單 → `agtSeedTestDocs`（種資料）→ F9 重整

### 第一輪：不建 design catalog（對照實驗）

5. **先不要跑 updall**，直接執行 `agtTestDQL`，把 `%TEMP%\dql-test-results.txt` 另存為第一輪結果
   - 官方文件說法：catalog 不存在時 "Only the `'View or folder name'>.<Columnname` syntax will fail"（A~F、H 應全部報錯，只有 G 的 explain 也會錯）
   - 實測目的：驗證是真的報錯、還是像 `@All` 一樣官方說法與實際行為有落差（例如 fallback 成 NSF 掃描）

### 第二輪：建 catalog 後重跑

6. **若 DB 在伺服器上**：console 跑 `load updall <db path> -e` 建 design catalog（之後改過 view 設計要再跑 `-d`）
7. 再跑一次 `agtTestDQL`，另存第二輪結果
8. 兩輪結果都貼回給 Claude 對照預期矩陣分析

## 已知注意事項

- 本機（非伺服器）DB 沒有 design catalog，`'view'.column` 語法可能整批報錯——錯誤訊息也是有效的測試結果，一樣貼回來
- Designer sync 時若把 `$123` 改名（programmatic name 衝突），以實際名稱修改 `agtTestDQL` 內的查詢字串
- `@TextToTime("2024/1/5")` 的解析依伺服器/client 地區設定，zh-TW 環境應可正常解析
