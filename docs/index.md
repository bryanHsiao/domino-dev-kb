# Domino 開發案例知識庫

記錄 HCL Domino / DQL / LotusScript 開發過程中踩過的坑、查證過的官方文件結論，供團隊回饋與日後查閱。

## 案例索引

| 日期 | 案例 | 一句話結論 |
|---|---|---|
| 2026-07-16 | [DQL 用 view 日期直欄做範圍搜尋要不要轉型](cases/2026-07-16-dql-view-date-column.md) | DQL 不自動轉型；直欄輸出型別必須全部文件一致，且與查詢詞型別相同。`@dt` 要完整 timestamp + `+08:00`。已單點測試全數實證 |
| 2026-07-16 | [DQL 4854 診斷階梯與 design catalog 生命週期](cases/2026-07-16-dql-4854-diagnostic-ladder.md) | 4854 有三層錯誤訊息各有解法；view 設計變更後 `updall -d` 可能損毀 catalog 造成行為不一致；0 筆無錯誤時照檢查清單排查 |

## 新增案例

在 `docs/cases/` 下新增 `YYYY-MM-DD-簡短主題.md`，格式：**症狀 → 官方文件佐證（附連結與原文引用）→ 根因 → 正確做法 → 一句話總結**，並在 `mkdocs.yml` 的 `nav` 與上面的索引表各加一行。
