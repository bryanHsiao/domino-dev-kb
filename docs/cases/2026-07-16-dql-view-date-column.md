# DQL 用 View 直欄（日期時間）做範圍搜尋：要不要轉型？

> 案例日期：2026-07-16
> 症狀回報：「直欄轉成日期反而有些抓得到有些沒抓到，蠻怪的」
> 相關直欄：`$123`（來文日期），直欄公式：
> `@If(@IsText(DocDate); DocDate; @Text(@Date(DocDate)))`
> 查詢寫法（LotusScript 組字串）：
> `in ('viewName') and 'viewName'.$123 >= '<dtStart>' and 'viewName'.$123 < '<dtEnd>'`

---

## 結論（TL;DR）

1. **DQL 沒有自動轉型**。官方文件明講：Domino 是無型別（typeless）資料模型，**「查詢詞裡給的值是什麼型別，就搜尋什麼型別」**。型別不一致的文件**不會報錯，只是默默查不到** —— 這就是「有些抓得到有些沒抓到」的直接原因。
2. `'view'.$123` 這種 view 直欄搜尋，比對的是 **view index 裡該直欄算出來的值**（不是文件欄位本身）。直欄公式若對不同文件輸出不同型別（有的文字、有的日期），就必然出現部分命中。
3. 本案的直欄公式本身就洩露了病根：`@IsText(DocDate)` 的存在，代表**底層欄位 DocDate 在部分文件裡是文字、部分是日期時間**（歷史資料混型）。把直欄「轉成日期」後，文字那批文件的直欄值仍是文字，用 `@dt(...)` 查就漏掉它們。
4. 另外兩個官方明定的地雷：
   - **View 搜尋不能用「只有日期」的 @dt 值**，必須給完整 timestamp（含時間）。
   - **@dt 不加時區位移一律當 UTC(GMT)**。台灣是 +08:00，忘了加會讓邊界日期前後 8 小時的文件跑錯邊。

---

## 官方文件佐證

### 1. 查詢值的型別決定搜尋型別（無自動轉型）

> "Domino features a typeless data model where any type of data can be stored in any field. In a DQL query, **the type of data searched for is defined by the type of data specified in the query term**."
>
> "Use a single quote (') to force DQL processing to interpret arguments as **text rather than date/times or numerals**."

— [DQL syntax（HCL Domino Designer 12.0.2）](https://help.hcl-software.com/dom_designer/12.0.2/basic/dql_syntax.html)

意思是：`'2024/01/01'`（單引號）= 搜文字；`@dt('2024-01-01T00:00:00+08:00')` = 搜日期時間。存的值型別跟查詢詞型別不同，就不會被比對到。

### 2. View 搜尋不接受「只有日期」的部分值

> "**Partial date values cannot be found using view searching.**"
>
> （解法：view 搜尋要給完整值，例如 `@dt('2018-09-01T00:00:00.000Z')`）

— [Date and time values（HCL Domino Designer 12.0.2）](https://help.hcl-software.com/dom_designer/12.0.2/basic/dql_date_and_time.html)

### 3. @dt 不加時區 = UTC

> "Without timezone modifiers (+ or – hh:mm suffixes) **all times are taken as UTC values (GMT)**."

— 同上頁。台灣環境要寫 `+08:00`，例如 `@dt('2024-01-01T00:00:00+08:00')`。

### 4. View 直欄查詢的前置條件

> "Columnname is the programmatic name of any **primary sorted column** in the view or folder specified."（直欄必須是有排序的直欄；`$123` 是 programmatic name，來文日期直欄有設遞增排序，符合）

— [DQL syntax](https://help.hcl-software.com/dom_designer/12.0.2/basic/dql_syntax.html)、[View column requirements](https://help.hcl-software.com/dom_designer/11.0.1/basic/dql_view_column.html)

> Design catalog 過期時："Only the `<'View or folder name'>.<Columnname>` syntax will fail."

— [Design catalog](https://help.hcl-software.com/dom_designer/14.5.0/basic/dql_design_catalog.html)

### 5. `SELECT @All` 條件：官方文件 vs 實測（重要修正）

官方文件（[View column requirements](https://help.hcl-software.com/dom_designer/14.5.0/basic/dql_view_column.html)）寫 `'view'.column` 語法要求：

> "the view must use Select @All as its selection criteria."（不符時 "the query fails with a return code"）

但 **Domino 12 實測行為不是報錯**（見 [DQL 踩坑筆記](https://bryanhsiao.github.io/domino-news/posts/dql-pitfalls/)）：

> "runtime 不會擋下「view 不是 Select @All」的查詢，沒有錯誤訊息 —— 但 **DQL 會以該 view selection 篩出來的 doc 為範圍**做查詢"

原因是 DQL 底層直接用 view 既有的 column index，而 index 建立時就只含符合 selection 的文件。**實務影響**：view selection 公式篩掉的文件，查詢時無聲消失、不會有任何錯誤——這在 selection 複雜的正式 view 上是另一個「有些抓得到有些沒抓到」的來源。解法（依該文）：改 view 為 `SELECT @All`、改用 `in('viewname') and field = value` 語法、或建專用的 `($DQL)` 查詢 view。

**改過直欄公式之後**，記得讓 view index 重建，並更新 design catalog：
`load updall <db path> -d`（初次建立用 `-e`）。沒更新的話，view 直欄語法可能失敗或用到舊的直欄定義。

> ✅ **實測（Domino，2026-07-16 單點測試）**：`'view'.column` 查詢失敗時都是錯誤碼 4854，但訊息分層、可直接對症下藥——
>
> | 層 | 4854 錯誤訊息（原文） | 病因 / 解法 |
> |---|---|---|
> | 1. 無 design catalog | `Error validating view column name - ['view'.$123] .. invalid view name or database needs to be cataloged via updall -e` | 跑 `load updall <db> -e`（view 設計變更後跑 `-d`） |
> | 2. view index 沒建過 | `View never built [viewName] - query term will fail, aborting` | view 建好後從未被開啟過；開一次 view 或 `load updall <db> -t <view>` 建 index，DQL 不會代建 |
> | 3. @dt 只給日期 | `Partial TIMEDATEs NOT supported with view column lookup, specify a full TIMEDATE` | 官方規則的 runtime 實證；給完整 timestamp（含時間與時區） |
>
> 三層都是**硬報錯、不會默默 fallback 或給部分結果**。**排查判斷式**：查詢「有結果但部分缺漏」＝ 上面三層都過了，問題出在型別混雜/時區/view selection 篩範圍。

---

## 為什麼「轉成日期反而有些抓得到有些沒抓到」

把三件事疊起來就完全解釋得通：

| 原因 | 機制 | 症狀 |
|---|---|---|
| 底層 DocDate 混型（部分文件存文字） | 直欄改成回傳日期後，`@IsText` 分支的文件直欄值仍是文字；`@dt` 查詢詞只比對日期型的值 | 日期型的那批抓得到，文字型的那批默默消失 |
| `@dt` 沒帶 `+08:00` | 查詢邊界被當成 UTC，與台灣時間差 8 小時 | 邊界日（起訖日）附近的文件跑錯邊 |
| `@dt` 只給日期沒給時間 | view 搜尋不支援 partial date | 查詢報錯或行為不符預期 |
| view selection 不是 `SELECT @All` | DQL 以 view index 為範圍，被 selection 篩掉的文件不在 index 內（實測不報錯） | selection 排除的文件無聲消失 |

---

## 正確做法（二選一，重點是「直欄型別 = 查詢詞型別，且全部文件一致」）

### 方案 A（建議）：直欄固定輸出「補零文字」，查詢用文字比較

最穩、完全避開時區與混型問題，也最接近現行程式寫法。直欄公式改成強制正規化為 `yyyy/mm/dd` 補零格式：

```
_d := @If(@IsText(DocDate); @TextToTime(DocDate); DocDate);
@Text(@Year(_d)) + "/" +
@Right("0" + @Text(@Month(_d)); 2) + "/" +
@Right("0" + @Text(@Day(_d)); 2)
```

查詢維持文字比較（注意 dtStart / dtEnd 也要是 `yyyy/mm/dd` 補零格式）：

```
in ('viewName') and 'viewName'.$123 >= '2024/01/01' and 'viewName'.$123 < '2024/02/01'
```

> ⚠️ 現行公式 `@Text(@Date(DocDate))` 的輸出格式跟著**伺服器地區設定**走（zh-TW Windows 可能吐 `2024/1/5` 不補零），不補零的文字做範圍比較會錯（`2024/1/15` < `2024/1/2`）。所以一定要用上面的公式強制補零，不能依賴 @Text 預設格式。

### 方案 B：直欄統一輸出日期時間，查詢用 @dt 完整 timestamp

直欄公式讓所有文件都輸出 datetime：

```
@If(@IsText(DocDate); @TextToTime(DocDate); DocDate)
```

查詢必須（1）完整 timestamp（2）帶 +08:00 時區：

```
in ('viewName') and
'viewName'.$123 >= @dt('2024-01-01T00:00:00+08:00') and
'viewName'.$123 <  @dt('2024-02-01T00:00:00+08:00')
```

> ⚠️ 若有文件的 DocDate 文字內容不是 @TextToTime 能解析的格式，該文件直欄會是錯誤值，仍會漏抓。方案 B 建議搭配資料清整（agent 把文字 DocDate 批次轉回真正的日期欄位），一勞永逸。

### 兩個方案共同的收尾步驟

1. 改完直欄公式後重建 view index（Shift+F9 或 `load updall <db> -r`）。
2. 更新 design catalog：`load updall <db> -d`。
3. 用邊界資料驗證：挑「起日當天 00:00 附近」「訖日當天」「DocDate 是文字」的文件各一份，確認查詢結果數量正確。可用 `NotesDominoQuery.Explain` 看查詢計畫確認有走 view index。

---

## 單點測試實錄（2026-07-16，Domino 伺服器實測）

**環境**：測試 DB 含 412 份文件 = 404 份真實收文（2023~2026，DocDate **本來就大量混型**，抽樣可見數十筆 STRING）+ 8 份控制組測試文件 T1~T8。同一個 `$123` 直欄做了 5 種 view 變體，查詢區間 2024/01/01（含）~ 2024/02/01（不含）本機時間，理想命中 = T1,T2,T3,T6,T7,T8 共 6 筆。

測試文件：T1=2024/01/01 00:00（起日邊界）、T2=01/15 12:00、T3=01/31 23:30、T4=02/01 00:00（區間外）、T5=2023/12/31 20:00（區間外）、T6=01/01 09:00，以上日期型；T7=文字`"2024/01/10"`、T8=文字`"2024/1/5"`（未補零）。

| # | 直欄公式 | 查詢 | 命中 | 結果分析 |
|---|---|---|---|---|
| A | 現行 `@Text(@Date())` 文字 | 文字比較 | 5/6 | ❌ **T8 遺失**：未補零文字 `2024/1/5` 範圍比較錯位 |
| B | **方案A** 補零 yyyy/mm/dd 文字 | 文字比較 | **6/6** | ✅ 全中 |
| C | 方案B 日期型 | `@dt('2024-01-01')` 只給日期 | 報錯 | ✅ `Partial TIMEDATEs NOT supported with view column lookup` |
| D | 方案B 日期型 | `@dt(...T00:00:00Z)` 無時區 | 6 筆但**成員錯** | ❌ **T1 漏抓、T4 誤入**——筆數看起來對、內容錯，最危險 |
| E | **方案B** 日期型 | `@dt(...T00:00:00+08:00)` | **6/6** | ✅ 全中；Explain 證實走 view index（ScannedDocs 0） |
| F | 混型（DocDate 原樣） | 同 E 的 @dt 查詢 | 4/6 | ❌ **T7/T8 文字型無聲消失——工程師症狀完整重現** |
| H | 仿真實 view（非 @All、分類欄、遞減排序） | 文字比較 | 4 | selection 排除的 T6 **無聲消失、不報錯**（實證 [DQL 踩坑筆記](https://bryanhsiao.github.io/domino-news/posts/dql-pitfalls/)）；分類最左欄 + `$123` 遞減（`resort` 升冪）可正常查詢 |

**真實 view 最終驗證**：對正式的來文簽辦單 view（`$123` = 現行文字公式）直接下查詢——用 `@dt` 查 0 筆、用文字比較查中目標文件（1 筆）。**判斷法則：看直欄公式最終輸出的型別，決定查詢詞型別**；現行公式 `@If(@IsText(DocDate);DocDate;@Text(@Date(DocDate)))` 兩個分支都輸出文字，所以是文字直欄、要用文字比較。

**三個最重要的實證**：

1. **F 完整重現「有些抓得到有些沒抓到」**：混型直欄 + `@dt` 查詢 → 日期型命中、文字型無聲消失，無任何錯誤或警告。
2. **D 是最危險的陷阱**：`@dt` 忘了帶 `+08:00` 時，T1 被漏掉、T4 被誤抓——**總筆數還是 6、看起來完全正常**，只有逐筆核對才會發現內容錯了。
3. **真實資料的混型比想像嚴重**：404 份真實收文中抽樣就有數十筆 DocDate 是文字型（且橫跨 2023~2026 都有），代表某個輸入路徑持續寫入文字日期——**資料清整是治本，直欄正規化是治標**，兩者都該做。

> DQL 不會自動轉型：`'...'` 查文字、`@dt(...)` 查日期，型別對不上的文件不報錯、直接查不到。這個 view 的底層欄位 DocDate 本來就有文字/日期混型（直欄公式裡的 @IsText 就是證據，實測 404 份真實文件抽樣有數十筆文字型），所以「直欄轉日期」後只剩日期型的那批查得到——**單點測試已完整重現**（混型直欄 @dt 查詢 6 筆只中 4 筆，文字型無聲消失）。解法是讓直欄對**所有文件輸出同一種型別**，再用同型別的查詢詞：**方案 A（補零 yyyy/mm/dd 文字 + 文字比較）與方案 B（@TextToTime 統一日期型 + 完整 timestamp 帶 +08:00 的 @dt）實測都 6/6 全中**。@dt 忘記帶時區是最陰的坑：實測筆數一樣是 6、但邊界文件一漏一誤，逐筆核對才看得出來。另外真實 view 的 selection 不是 @All 也會讓被篩掉的文件無聲消失（實測不報錯）。

## 參考資料

- [DQL syntax — HCL Domino Designer 12.0.2](https://help.hcl-software.com/dom_designer/12.0.2/basic/dql_syntax.html)
- [Date and time values — HCL Domino Designer 12.0.2](https://help.hcl-software.com/dom_designer/12.0.2/basic/dql_date_and_time.html)
- [View column requirements — HCL Domino Designer 11.0.1](https://help.hcl-software.com/dom_designer/11.0.1/basic/dql_view_column.html)
- [Design catalog — HCL Domino Designer 14.5.0](https://help.hcl-software.com/dom_designer/14.5.0/basic/dql_design_catalog.html)
- [View column requirements — HCL Domino Designer 14.5.0](https://help.hcl-software.com/dom_designer/14.5.0/basic/dql_view_column.html)
- [DQL 踩坑筆記（Domino 12 實測）— domino-news](https://bryanhsiao.github.io/domino-news/posts/dql-pitfalls/)
