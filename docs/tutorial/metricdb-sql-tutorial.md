# metricdb SQL 教程（對照 Metric/ 程式碼）

> 給下次的 Claude：先讀「進度」這一節，從「下一課」繼續。每課教完就更新進度、補上學生的疑問。
> 教法：一次只教一課 → 先看 Python func → 寫出等價 SQL → 用 dump 裡的真實資料解釋 → 留 1~2 題小練習。

## 進度

| 課 | 主題 | 狀態 |
|----|------|------|
| 1 | 三張核心表：`metric_batches` → `metric_queries` → `metric_url` | ✅ 已教 (2026-10-03)，複習 (2026-10-04)；小練習 1.5 尚未作答 |
| 2 | `DatabaseRawDataReader`：讀快取 batch / 寫入新 batch | ✅ 已教 (2026-10-04) |
| 3 | `get_latest_batch_id`（measure.py） | ✅ 併入第 2 課 2.5 |
| 4 | `Head/RandomQueryStrategy.getGoldenSet`：SELECT → UPDATE tags → DELETE → INSERT | ✅ 已教 (2026-10-04) |
| 5 | `_update_batch_stats`：JSONB `@>` + JOIN + COUNT | ✅ 已教 (2026-10-04) |
| 6 | `CrawlerStatusMeasure._scan_shard`：`COUNT(*) FILTER` | ✅ 已教 (2026-10-04) |
| 7 | `_get_daily_summary_stats`：1/7/30 天滾動統計（Python 版 vs SQL 版） | ✅ 已教 (2026-10-04) |
| 8 | UPSERT：`INSERT ... ON CONFLICT (stat_date) DO UPDATE` → `crawler_stat_*` | ✅ 已教 (2026-10-04) |
| 9 | `CrawlerAllMetricMeasure`：Golden URL → 跨 DB 查狀態 → bulk UPDATE → `metric_headset_*/randomset_*` | ⏳ 下一課（主流程已在附錄 C，補 bulk_update_mappings 與 shard 對應） |
| 10 | 用 SQL 讀 dashboard / 抓資料異常（綜合練習） | |

學生的疑問 / 需要加強的地方：
- [第1課] `1 ──<` 符號看不懂 → 已解釋（見 1.6）。對「一對多 / FK 在多的那一邊」還不熟，後面 JOIN 時再強調。
- [提前問] Superset 圖「Total Overview Volumn」「Crawled (Daily/Weekly/Monthly)」怎麼算 → 附錄 A。教第 6/7 課時直接接著講。
- [誤解已澄清] 以為 crawled weekly 會去 metric_batches/metric_url 撈 → 錯，crawler_stat_* 和 metric_url 是兩條獨立管線（附錄 A.6）。之後講第 9 課時再對比一次。
- [提前問] metric_url 網站類型 + 是否混入廣告/知識面板 → 附錄 B（發現 batch 9 有 179 筆 /goto 假網址）。
- [第1課] 同 keyword 出現在多個 batch 時取哪個？→ 見 1.6。

- [複習小考 2026-10-04] 第1課 5 題：②④ 對；① 錯（以為 metric_url 有 batch_id）；③ 混淆 coverage 與 daily/weekly/monthly；⑤ 不懂 tags=[] 的意義 → 下次先抽問 ①③⑤。
- [問] shard 是什麼 → 見附錄 D。

## 待處理問題

> 學生的任務是 **investigation of current metrics**（調查現有 metric 是否正確、可信），以下皆為調查發現，全部位於 `Metric/`，不屬於 IndexSelection。

- [ ] **metric_url 混入 `/goto` 假網址（2026-10-04 發現，詳見附錄 B.2）**
  - 影響最大的是 `/goto` 那 179 筆。它們不是合法的網址，crawler 永遠不可能抓到，所以會一直拉低 batch 9 的 discovered rate 和 crawled rate。原因可能是當時 Google 或 SerpApi 的回傳格式改了，這只是推測，沒有確認。
  - 建議的修正：在 `QueryStrategy.getQuery` 只保留 `link.startswith("http")` 的網址，也可以順便排除 `google.*` 網域。（尚未修改程式，等學生決定）
- [ ] **coverage 比對 URL 沒有 normalize → discovered/crawled/indexed 被低估（2026-10-04 發現，詳見附錄 C.4 第 1 點）**
  - 發生位置：`CrawlerAllMetricMeasure` 拿 golden URL 跟 crawlerdb `url_state.url`（discovered/crawled）和 selectdb `selected_urls_current.url`（indexed）做**逐字字串比對**。同一頁不同寫法（www、http/https、結尾 /、大小寫、utm/#）就會比對不到。
  - dump 證據：同頁不同寫法 47 組，其中 18 組 discovered 一 t 一 f（例：`ligamx.net/` vs `www.ligamx.net/`）。實際低估總量未知（golden set 只出現一種寫法時看不出來）。
  - 附帶影響：golden set 去重也是逐字，同頁兩種寫法算 2 個 → 分母 `total` 多算。crawler_stat Overview 不受影響（只做 COUNT）。
  - 建議的修正：比對前兩邊用同一個 normalize 函式，最好沿用 crawler 本身的 normalize 邏輯（需先確認 crawler 存 URL 時怎麼 normalize）。（尚未修改程式，等學生決定）
- [ ] **SerpApi 搜尋沒有指定國家 / 語言（2026-10-04 提出，詳見附錄 A.6）**
  - 發生位置：`Metric/Query/QueryStrategy.py:getQuery` 只傳 `q`、`num=10`，沒有傳 `gl`（國家）、`hl`（語言）、`location` → 全部用 SerpApi 預設地區搜尋。
  - 影響：keyword 來自 10 國的 trending（`metric_queries.geo` 記錄了國家），但拿回的 URL 可能不是當地使用者看到的結果 → golden set 偏向預設地區的網站。（dump 中仍有 ja.wikipedia 1,480、news.yahoo.co.jp 657 等在地網站，推測是 keyword 本身的語言帶出來的，未驗證）
  - 設計抉擇：一個 keyword 可能對應多國（geo = `["IN","JP","GB",...]`）→ 要選一國（例如第一個 / frequency 最高的）還是每國各搜一次（SerpApi 額度 × 國家數）。
  - [2026-10-04 討論] **keyword 不需要翻譯**：trending 詞本來就是當地人搜尋的原文（dump 實算，單一國家的 keyword：JP 94% 日文/漢字、TW 84% 漢字、IN 40% 印度文字、DE/FR/BR 為當地語言）。翻譯會改變 query 本身，就不是當地使用者真正搜的東西。要做的是**用原文、加上該國的 `gl`（國家）+ `hl`（介面語言）**。
    - 建議對照（hl 代碼需再對照 SerpApi 文件確認）：US en / IN en（或 hi）/ JP ja / GB en / CA en（法語詞可用 fr）/ DE de / AU en / BR pt / FR fr / TW zh-TW。
    - 多國 keyword 佔 11%（15,347 / 135,590）。目前 merge 時各國 frequency 被加總，**沒保留各國分量**，也無法得知哪國最熱；geo list 的順序只是抓取順序（US, IN, JP…）。選項：(a) 只用 geo[0]；(b) 每國各搜一次（額度 × 國家數，可只對 head 做）；(c) 改 `_fetch_api_process_and_store` 保留各國 frequency，再挑最大者。
  - [2026-10-04 討論] 要不要多記「語言」？→ 不用記 keyword 的語言（hl 可由 geo 對照表推出，偵測語言也不可靠）。要記的是「**實際用什麼條件搜的**」，否則事後無法重現或解釋結果：
    - 做法 (a)/(c)（每個 keyword 只搜一次）：`metric_queries` 加 `search_gl`、`search_hl`（varchar）。
    - 做法 (b)（每國各搜一次）：條件要記在 `metric_url`（加 `gl` 欄位），因為同一 query 會有多組結果；去重單位也要變成 (url, gl) 或維持 url。
    - 做法 (c) 另需 `metric_queries` 加 `geo_freq jsonb`（例：`{"IN": 5000000, "JP": 100000}`）保留各國搜尋量。
    - 舊資料：沒有這些欄位的列 = SerpApi 預設地區搜尋（可補 NULL 表示「未指定」）。
  - 建議的修正：`getQuery` 加 `gl` / `hl` 參數，由 `getGoldenSet` 依 `s['geo']` 傳入；若要每國各搜一次，metric_url 可能需要記錄是哪個國家的結果。（尚未修改程式，等學生決定）
- [ ] **失敗時安靜地吞掉錯誤，產生看似正常的值（2026-10-04；共同方向：錯誤被 except 吃掉 → 回傳空值 / 0 → 被當成正常結果寫入）**
  - 共同修正原則：失敗要**看得見**（logging 記錄位置與例外內容）、**不要寫入假值**（不寫或寫 NULL）、**記錄失敗次數**。
  - **4-A｜SerpApi 失敗 → 空 batch / 0 個 URL 的 query（詳見第 2 課 2.3、2.7）**
    - 程式位置：`Metric/RawDataReader/DatabaseRawDataReader.py:_fetch_trending_now`（重試失敗回傳 `[]`）→ `_save_to_database` 照樣建 batch；`Metric/Query/QueryStrategy.py:getQuery`（失敗回傳 `[]`）。
    - 管線 / 影響指標：管線 B（coverage）→ metric_headset_* / metric_randomset_*。
    - 現象：10 國全失敗 → 空 batch（batch 10~17）→ 該輪沒有 coverage，快取期間不重抓；單一 keyword 失敗 → 有 tag 但 0 個 URL（dump：1,416 個，batch 1: 364、batch 5: 23、batch 9: 1,029）→ golden set 變小。
    - 待確認：10 月起是否仍會發生。
    - 建議修正：rawData 為空時不建 batch；`getQuery` 失敗時丟例外或跳過該 keyword；用快取前檢查 `meta_total_queries > 0`。
    - 相關：重跑時清掉舊 URL → 第 5 項。
  - **4-B｜crawlerdb shard 查詢失敗 → 該 shard 當成 0（詳見第 6 課 6.4）**
    - 程式位置：`Metric/Measure/CrawlerStatusMeasure.py:_scan_shard`（`except Exception: return shard_id, 0, 0, 0`，**完全沒有 log**）；同類：`Metric/Measure/CrawlerAllMetricMeasure.py:_scan_url_shard` / `_scan_domain_shard`（只 print 固定字串，沒有 shard 編號與例外內容）。
    - 管線 / 影響指標：管線 A（crawler_stat_* → Overview 圖的 discovered / crawled）；coverage 端則會讓該 shard 的 golden URL 被判為 discovered = false。
    - 現象：任一 shard 失敗 → 總數少約 1/256，無錯誤訊息；每小時 UPSERT 會蓋掉當天先前的正確值。判讀：discovered 和 crawled **同時**下降、之後回升（V 字缺口）；與修剪（只降 discovered、crawled 照升）不同。
    - 可能原因：修剪時表被替換或鎖住、磁碟滿（1TB）、連線數不足（整點多個 cron）、schema / 權限變更、連線中斷。
    - 建議修正：logging 記 shard 編號 + 例外；設 `statement_timeout`；重試；仍失敗則不寫入或寫 NULL，並記錄 `failed_shards`；log 寫到有 volume 的位置。
  - （尚未修改程式，等學生決定）
- [ ] **重跑同一 batch 時，SerpApi 失敗會清掉原本的 URL（2026-10-04 發現，詳見第 2 課 2.7）**
  - 發生位置：`Head/RandomQueryStrategy.getGoldenSet` C 段：`DELETE FROM metric_url WHERE query_id = ?` → `getQuery` 失敗回傳 `[]` → 插入 0 筆 → `commit`。
  - 影響：原本好好的 URL（以及 measure 回填的 is_* 狀態）被永久刪除；該 query 變成「有 tag、0 個 URL」→ golden set 變小。也會發生在 head/random 都選中同一 keyword、後跑的那次 SerpApi 失敗時。
  - 建議的修正：`url_list` 為空時跳過該 keyword（不 DELETE、不 commit），或讓 `getQuery` 失敗時丟例外讓該 keyword rollback。（尚未修改程式，等學生決定）
- [ ] **重跑 random 時 keyword 越選越多（2026-10-04 發現，詳見第 4 課 4.4）**
  - 發生位置：`RandomQueryStrategy.getGoldenSet` 每次重新 `random.sample`（無 seed）並補上 "random" tag，但**不會移除上一次的 "random" tag**。
  - dump 證據：batch 1 的 random query 有 1,313 個（> N=1000），`meta_tag_stats.random.queries` 也是 1313 → 推測 batch 1 的 random 跑過不只一次。
  - 影響：randomset 的分母變大、樣本不再是「一次隨機抽 N 個」，不同 batch 之間不好比較。head 無此問題（每次選同樣前 N 名）。
  - 建議的修正：跑 random 前先清掉該 batch 的 random tag（`UPDATE metric_queries SET tags = tags - 'random' WHERE batch_id = :b AND tags @> '["random"]'`），並一併處理那些 query 的舊 URL；或固定 seed / 已有 random 時直接跳過。（尚未修改程式，等學生決定）
- [ ] **Dashboard「Crawled (Daily/Weekly/Monthly)」其實是 fetch 次數，不是 URL 數（2026-10-04，詳見附錄 A.4、A.6）**
  - 位置：`superset/bootstrap.py` 的 `vw_crawled_rolling` 把 `fetch_ok / fetch_ok_7 / fetch_ok_30` 改名成 `crawled_daily / weekly / monthly`；來源是 crawlerdb `summary_daily.num_fetch_ok`（`CrawlerStatusMeasure._get_daily_summary_stats`）。
  - 影響：同一 URL 抓 N 次算 N 次，和「Total Overview Volumn」的 `crawled`（distinct URL）單位不同，名稱相同容易誤讀。另：當天值是每小時 UPSERT 的部分值（23:00 後不再更新）。
  - 補充（第 7 課 7.4 驗證）：存下來的 daily 是 23:00 那次的值，少了最後 1 小時 → **daily 系統性低估約 4%**（weekly − 7 天 daily 加總，中位數 3.5%）。
  - 補充（第 8 課 8.3）：**今天**那一列一直在被每小時 UPSERT 覆蓋，dashboard 最右邊的點是未完成值（dump 的 10-01 fetch_ok 只有平常的約 13%）。
  - 建議：改名為 `fetch_ok_daily` 等；若需要「期間內被抓過的 distinct URL 數」，用 `COUNT(*) FILTER (WHERE last_fetch_ok >= now() - interval '7 days')` 每天計算並存下。（尚未修改，等學生決定）
- [ ] **indexed 指標不可靠：crawler_stat 寫死 0；coverage 的 indexed 時有時無（2026-10-04，詳見附錄 A.2、C.4 第 6 點）**
  - crawler_stat：`CrawlerStatusMeasure._scan_shard` 直接回傳 indexed = 0 → `crawler_stat_*` 的 indexed 173 列全是 0，Overview 圖的 Indexed 線永遠是 0。
  - coverage（`metric_headset/randomset_total`，dump 實查）：
    - 06-05、06-12 有正常值（indexed_num 1,509 / 1,449 / 1,260，rate 16~21%）← `b0bea75 feat: add index coverage`（2026-06-03）之後。
    - **09-17 又變回 0**（功能已上線卻是 0）→ 待查：當次是否沒傳 `--select_db_url`、selectdb 連不上、或 `selected_urls_current` 為空。
    - 04-01 ~ 05-03 共 6 列 **indexed_num = 0 但 indexed_rate = 0.16~0.23**（自相矛盾；功能上線前）→ 推測為手動補值，來源待查。
  - 建議：crawler_stat 的 indexed 改從 selectdb 計數或明確標示「未實作」；coverage 在 selectDB 為 None 時寫 NULL 而非 0（區分「沒量」與「量到 0」）。（尚未修改，等學生決定）
- [ ] **rank 有存但沒用到；ranked 指標寫死 0（2026-10-04，詳見附錄 A.6、第 4 課）**
  - `metric_url.rank` 由 `getGoldenSet` 寫入（Google 名次 1~10），但沒有任何程式讀取；coverage 每個 URL 權重相同。
  - `metric_url.is_ranked` 永遠 false；`ranked_num / ranked_rate` 在 `CrawlerAllMetricMeasure` 寫死 0；`TypesenseRankMeasure.test()` 是空 stub（原邏輯被註解）；`measure.py --measure rank` 沒有對應分支。→ Dashboard 的 H/R - RankCov 線永遠是 0。
  - 建議：dashboard 移除或標示「未實作」；若要實作，恢復 Typesense 搜尋比對，並可依 rank 加權（例如 top-3 coverage）。（尚未修改，等學生決定）
  - 背景：rank 是上一屆想做、但沒想出合適演算法的指標。
  - **演算法方向（2026-10-04 討論，完整內容見附錄 E）**：
    - 漏斗定位：discovered → crawled → indexed → **ranked**；同時報 `ranked / total`（端到端）與 **`ranked / indexed`**（純排序能力），避免排序問題被低 coverage 蓋掉。
    - 候選：Recall@10、Hit@10、MRR、NDCG@10（Google 名次當相關度）、RBO（比較兩份排序清單）。
    - 推薦組合：dashboard 用 **Recall@10**；分析用**以 indexed 為條件的 NDCG@10**（IDCG 只用有 index 的 golden URL 計算 → coverage 與 ranking 分開）。
    - 前置條件：先處理 #2（URL normalize）、#3（gl/hl）；量測時間要接近 SerpApi 抓取時間；head / random 分開報。
    - 資料表：`metric_url` 加 `our_rank`（我們搜尋結果中的名次，沒出現 = NULL）；可另建 `metric_query_rank(query_id, stat_date, recall_10, ndcg_10, mrr)`。
    - 限制：Google 不是絕對正解，指標衡量的是「與 Google 的一致程度」。
- [ ] ⚠️ **【分散式必看】read-modify-write 造成 lost update（2026-10-04，詳見第 5 課 5.2）** —— 接下來要改成分散式系統（3 台主機），此項優先處理
  - 現況：`_update_batch_stats` 先讀 `meta_tag_stats` → Python 修改 → 整個寫回。head（12:00）與 random（18:00）分開跑，目前不會撞到。
  - 分散式後的風險：多台主機同時跑 → 兩邊都讀到舊值、各自寫回 → 後寫的蓋掉先寫的。
  - **同類型、影響更大的地方**（都是「先讀再寫」，分散式時都會出事）：
    1. `getGoldenSet` 的 tags：同時跑 head 和 random、選到同一 keyword → 各自讀到 `[]`、各自寫回 `["head"]` / `["random"]` → **其中一個 tag 消失** → 該 query 從某個 set 中不見，coverage 分母改變。
    2. `getGoldenSet` 的「先 SELECT 有沒有、沒有才 INSERT」：兩台同時查都沒有 → 都 INSERT → 同 batch 出現重複 keyword（DB 沒有 `UNIQUE(batch_id, keyword)`）。
    3. `getGoldenSet` 的 DELETE + INSERT URL：兩台同時處理同一 query → URL 被插兩份。
    4. `readData`：兩台同時發現快取過期 → 各建一個新 batch → 不同主機可能用到不同的「最新 batch」（id vs created_at，第 2 課 2.5）。
    5. 每小時的 `status` 若多台都跑：UPSERT 本身原子，但「最後寫的贏」→ 較舊的快照可能蓋掉較新的（第 8 課 8.5）；且 256 個 shard 被重複掃 N 次。→ 只讓一台跑，或加 `measured_at` + `DO UPDATE ... WHERE`。
  - 不受影響：`crawler_stat_*` / coverage 的 UPSERT（`INSERT ... ON CONFLICT` 在 DB 內原子完成）。
  - 建議修正方向：
    - 改用 DB 端原子操作：`meta_tag_stats = meta_tag_stats || jsonb_build_object(...)`、`tags = tags || '["head"]' WHERE NOT tags @> '["head"]'`。
    - 加約束：`UNIQUE(batch_id, keyword)`，搭配 `INSERT ... ON CONFLICT DO UPDATE`。
    - 需要「先讀再寫」時用 `SELECT ... FOR UPDATE` 鎖住該列，或用 PostgreSQL advisory lock 確保同一 batch 的 `--create` 只有一台主機在跑。
    - 或在架構上規定：dataset 建立（`--create`）只由一台主機負責，其他主機只跑 measure。
  - （尚未修改程式，等學生決定）
- [ ] **資料保留與可回溯：哪些資料要留、怎麼留，才能重算歷史 metric（2026-10-04 提出）**
  - 起因：batch 9 random 在 06-12 量測時 total = 7,738 個 distinct URL，現在 metric_url 只剩 7,139 個 → 量測後 URL 被刪，**舊 coverage 無法重現**（第 5 課 5.4）。
  - 目前會「覆蓋 / 刪除」歷史的地方：
    1. `getGoldenSet`：DELETE + INSERT URL（重跑就換掉舊結果）。
    2. `CrawlerAllMetricMeasure`：`metric_url.is_*`、`shard_id` 每次量測直接覆蓋，只剩最後一次狀態。
    3. coverage / crawler_stat：PK = `stat_date` + UPSERT → 同一天重跑會蓋掉前一次；看不出是哪一次執行、用哪個 batch 算的。
    4. `metric_batches.meta_*`：摘要會過時（第 5 課）。
    5. SerpApi 原始回應、trending 各國 frequency、`started`、random 抽樣結果：沒存或只存加工後的。
  - 建議保留的資料（能「重算」的最小集合）：
    - **輸入**：trending 原始回應（各國分開）、SerpApi 原始 JSON（含 q / gl / hl / 時間）、選擇參數（strategy、N、seed）。
    - **golden set 快照**：(run_id, query_id, url, rank)，不可修改。
    - **量測證據**：(measure_run_id, url, discovered, crawled, last_fetch_ok, shard_id, indexed)，每次量測新增，不覆蓋 metric_url。
    - **執行紀錄** `metric_runs`：run_id、類型（create / measure）、batch_id、git commit、CLI 參數、主機名稱（分散式）、開始 / 結束時間、成功 / 失敗。
    - **結果**：coverage / crawler_stat 加 `run_id` 外鍵；PK 改成 (stat_date, run_id) 或另存歷史表。
  - 建議保留方式：**append-only（只新增、不 UPDATE / DELETE）+ run_id 版本化**；「目前值」用 VIEW 取每個 key 最新的 run。好處：可重算、可比較兩次執行、分散式下大多只有 INSERT，順便消除第 10 項的 lost update。
  - 可選：每次 run 存一個「輸入指紋」= sha256(排序後的 golden URL 清單)，量測結果也記錄這個 hash → 之後可以驗證「這筆 coverage 是用哪一組 URL 算的、資料有沒有被改過」。
  - 關於 EIP-8288：它是 Ethereum 把後量子簽章 / STARK 證明「每個區塊聚合成一個證明」的提案（Vitalik，2026-09 草案），**解決的是鏈上驗證成本，和資料保留無關**，不適用。可借用的區塊鏈概念是更基本的「不可改的 append-only 紀錄 + 雜湊承諾（commitment）」，即上面的 run_id + hash 指紋；不需要真的上鏈或用 STARK。
  - 容量估計：每次 measure 約 7~15k 列證據、每次 create 約 2k 個 SerpApi JSON → PostgreSQL 完全可負擔；可設保留期限（例如原始 JSON 保留 1 年）。
  - （尚未修改程式，等學生決定）
- [ ] **crawler 停機時「沒資料」NULL 與 0 混用，且無停機標記（2026-10-04，詳見第 7 課 7.4 (b)）**
  - 程式位置：`Metric/Measure/CrawlerStatusMeasure.py:_get_daily_summary_stats`（`stats_data = defaultdict(int)`）。
  - 管線 / 影響指標：管線 A，`crawler_stat_*` 的 fetch / error 欄位 → Superset「Crawled (Daily/Weekly/Monthly)」、「Request Effectiveness」、「Fetch Total/OK/Fail」。
  - 現象：summary_daily 沒有「今天」的列時（crawler 停機），`fetch_ok` 等沒被設定 → UPSERT 未提供 → **NULL**；但 `day_total = stats_data["fetch_total"]` 讀取 defaultdict 時自動建立 key → `fetch_total` = **0**。同一列裡「沒資料」有的是 NULL、有的是 0。
  - dump 證據：2026-06-06 ~ 08-07 共 63 列 `fetch_ok` = NULL、`fetch_total` = 0，且 discovered / crawled 63 天完全不變（5,056,751,243）→ crawler 停機，但 measure 持續寫入；weekly / monthly 逐日滑落到 0，圖上看起來像「爬得越來越少」。
  - 建議修正：讀 dict 用 `.get(key)`；沒有當天資料時 fetch 欄位一律寫 NULL；加 `crawler_active`（或 `summary_row_exists`）欄位，dashboard 可標示停機區間。
  - （尚未修改程式，等學生決定）
- [ ] **A / B 的 fetch 流量其實可以拆，目前全寫 0（2026-10-04，詳見第 7 課 7.4 (c)）**
  - 程式位置：`Metric/Measure/CrawlerStatusMeasure.py:test()`（A / B 的 `keys_to_zero` 全填 0、`request_success_rate = None`）。
  - 管線 / 影響指標：管線 A，`crawler_stat_a` / `crawler_stat_b` 的 fetch / error / 成功率欄位 → 兩隊的抓取效率無法比較（dashboard 若顯示 A/B 會是 0）。
  - 現況理由：`summary_daily` 是全域資料，沒有 shard。
  - 發現：crawlerdb 的 `domain_stats_daily`（`Database/CrawlerModels.py:DomainStatsDaily`）每個網域每天一列，**有 `shard_id`**、`num_fetch_ok`、`num_fetch_fail`、`fail_reasons` → 可用 `GROUP BY CASE WHEN shard_id < 128 THEN 'A' ELSE 'B' END` 算出兩隊的 daily / 7 / 30 天值。
  - 待確認：`domain_stats_daily` 加總是否等於 `summary_daily`（若不等，需了解差異來源）；表的大小與查詢成本。
  - 另：A/B「無法計算」應寫 NULL 而不是 0（同上一項的 NULL vs 0 問題）。
  - （尚未修改程式，等學生決定）
- [ ] **同 batch 的 head ∩ random keyword 被 SerpApi 搜兩次，浪費額度（2026-10-04，詳見附錄 D.1）** —— 成本問題，不影響 metric 正確性
  - 程式位置：`Metric/Query/HeadQueryStrategy.py` / `RandomQueryStrategy.py` 的 `getGoldenSet`：每個選中的 keyword 一律先 `getQuery()`（呼叫 SerpApi），之後才檢查 query 是否已存在。
  - 管線 / 影響：管線 B 的 dataset 建立；影響 SerpApi 額度（`Metric/getQuota.py`），不影響 coverage 數字。
  - dump 估算：同 batch 兩邊都選中的 keyword 共 760 個 → 多搜 760 次（約佔 19,969 次搜尋的 3.8%）；重跑與失敗重試未計入，實際更多。
  - 不算浪費的部分：不同 batch 出現同一 keyword 重搜 1,879 次（約 9.4%）→ 結果會隨時間變，屬於設計上合理。
  - 建議修正：先查 query 是否已有 URL（同 batch、已被另一個 strategy 搜過）→ 有就只補 tag、跳過 `getQuery()`。順便避開第 5 項（SerpApi 失敗時清掉舊 URL）。
  - 注意：若兩個 strategy 要求不同的搜尋條件（例如之後加 gl/hl，第 3 項），才需要各搜一次。
  - （尚未修改程式，等學生決定）

## 素材：dump 是什麼

- `copied_metric/metricdb_copy.dump` 是 **PostgreSQL custom format dump**（不是純 .sql），來源 PG 16.11，DB 名 `metricdb`，owner `metric`。
- 不需要 server 也能看內容：
  ```bash
  pg_restore -l copied_metric/metricdb_copy.dump            # 列出目錄 (TOC)
  pg_restore -s -f schema.sql copied_metric/metricdb_copy.dump  # 只匯出 DDL
  pg_restore -a -f data.sql   copied_metric/metricdb_copy.dump  # 只匯出資料 (COPY 格式)
  ```
- 本機目前沒有 postgres server / docker，無法直接跑 SQL；教學用的是 dump 裡抽出的真實資料。
- 表與資料量（dump 時間點）：

| 表 | 筆數 | 寫入者（func） |
|----|------|----------------|
| `metric_batches` | 18 | `DatabaseRawDataReader._save_to_database`、`*QueryStrategy._update_batch_stats` |
| `metric_queries` | 135,590 | `_save_to_database`（tags=`[]`）、`*QueryStrategy.getGoldenSet`（tags 加 head/random） |
| `metric_url` | 140,840 | `*QueryStrategy.getGoldenSet`（INSERT）、`CrawlerAllMetricMeasure.test`（UPDATE is_*） |
| `crawler_stat_{total,a,b}` | 173 each | `CrawlerStatusMeasure.test` |
| `metric_headset_{total,a,b}` | 11 each | `CrawlerAllMetricMeasure.test`（tag=head） |
| `metric_randomset_{total,a,b}` | 13 each | `CrawlerAllMetricMeasure.test`（tag=random） |

ORM 定義在 `Database/MetricModels.py`；動態產生的 `_total/_a/_b` 表由 `Database/ModelFactory/AppModelFactory.py` + `Database/utils.py:createAllMetricModel` 建立。

---

## 第 1 課：三張核心表的關係

### 1.1 一句話
**一個 batch（某次抓 trending 的快照）→ 很多 query（關鍵字）→ 每個 query 有很多 url（Google 前 10 名結果）**。

```
metric_batches (id)
   1 ──< metric_queries (batch_id → metric_batches.id)
              1 ──< metric_url (query_id → metric_queries.id)
```

### 1.2 DDL（從 dump 抽出）對照 ORM

```sql
CREATE TABLE metric_batches (
    id bigint NOT NULL,                 -- PK，預設 nextval('metric_batches_id_seq')
    created_at timestamp,               -- ORM default=datetime.now
    meta_total_queries integer,
    meta_total_urls integer,
    meta_tag_stats jsonb,               -- {"head": {"urls":7746,"queries":1000}, "random": {...}}
    meta_geo_counts jsonb               -- {"US": 2681, "JP": 2935, ...}
);

CREATE TABLE metric_queries (
    id bigint NOT NULL,
    batch_id bigint REFERENCES metric_batches(id),
    keyword varchar NOT NULL,
    geo jsonb,                          -- ["US","IN"]：在哪些國家上 trending
    frequency integer,                  -- search_volume（多國會加總）
    tags jsonb                          -- []、["head"]、["random"]、["head","random"]
);
CREATE INDEX ix_metric_queries_batch_id ON metric_queries USING btree (batch_id);
CREATE INDEX ix_metric_queries_tags     ON metric_queries USING gin (tags);

CREATE TABLE metric_url (
    id bigint NOT NULL,
    query_id bigint REFERENCES metric_queries(id),
    url text,
    rank integer,                       -- Google 排名 1..10
    is_discovered boolean, is_crawled boolean, is_indexed boolean, is_ranked boolean,
    shard_id integer                    -- 0-127 = Team A, 128-255 = Team B, -1 = 找不到
);
CREATE INDEX ix_metric_url_query_id ON metric_url USING btree (query_id);
```

| SQL 概念 | ORM 寫法（MetricModels.py） |
|----------|------------------------------|
| `bigint` PK + SEQUENCE | `Column(BigInteger, primary_key=True, autoincrement=True)` |
| `FOREIGN KEY ... REFERENCES` | `ForeignKey('metric_batches.id')` |
| btree index | `index=True` |
| GIN index on jsonb | `Index('ix_metric_queries_tags', tags, postgresql_using='gin')` |
| （不存在於 DB） | `relationship(...)`：只是 Python 端方便 `batch.queries` 這樣走，DB 裡沒有對應物 |

重點：
- **SEQUENCE**：`id` 沒有 `SERIAL` 關鍵字，是「建 sequence + `ALTER COLUMN id SET DEFAULT nextval(...)`」— 這就是 SERIAL 展開後的樣子。
- **為何 tags 要 GIN index**：之後的查詢都是 `tags @> '["head"]'`（「陣列包含 head」），btree 做不到，GIN 可以。
- `relationship()` 和 `default=` 都是 **Python 端**行為，dump 的 DDL 裡看不到 default（例如 `created_at` 沒有 `DEFAULT now()`）。

### 1.3 真實資料長相

```
metric_batches
id | created_at          | meta_total_queries | meta_total_urls | meta_tag_stats
 7 | 2026-05-30 00:00:58 | 15235              | 15113           | {"head":{"urls":7746,"queries":1000},"random":{"urls":7926,"queries":1000}}

metric_queries
id   | batch_id | keyword               | geo                                  | frequency | tags
2    | 1        | gold rate today       | ["IN"]                               | 10000000  | ["head"]
3    | 1        | south africa vs india | ["IN","JP","GB","DE","AU","FR"]      | 5167000   | ["head"]
2002 | 1        | alex de minaur        | ["US","IN"]                          | 5500      | ["random"]

metric_url
id    | query_id | url                                   | rank | disc | crawl | idx | shard_id
78132 | 53056    | https://en.wikipedia.org/wiki/Cuba    | 1    | t    | t     | f   | 0
78133 | 53056    | https://cri.fiu.edu/research/...      | 2    | t    | t     | f   | 57
```

### 1.4 從 dump 算出來的觀察（值得記住）

1. **tags 的分佈（batch 1）**：`[]` 8504、`["head"]` 893、`["random"]` 1206、`["head","random"]` 36、`["random","head"]` 71。
   - `[]` = 只是 trending 原始關鍵字，**沒被選進 golden set，所以沒有任何 metric_url**（metric_url 裡完全沒有 tags=`[]` 的 query）。
   - 同時有 head 和 random 的順序不同 → 代表哪個 strategy 先跑（`--strategy random head` vs `head random`）。查詢要用 `@>` 而不是 `=`，就是因為這個。
2. **batch 10~17 的 meta_total_queries = 0**（2026-06-19 ~ 08-07，每週一個空 batch）：`_fetch_trending_now` 全部失敗回傳 `[]` 時，`_save_to_database` 仍會建一個空 batch。而 `readData` 只看「最新一筆 batch 是否過期」，所以空 batch 也會被當成有效快取。→ 第 2 課細講。
3. **metric_url.shard_id**：A 70,735、B 67,062、-1 有 3,043（URL 和 domain 都在 crawler DB 找不到）。

### 1.6 Q&A（學生問過的）

**Q：`1 ──< metric_queries (batch_id → metric_batches.id)` 是什麼意思？**
- `1 ──<` 是 ER 圖的「一對多」（`<` 是鳥爪/分岔 = 多）：**1 個 batch 對應多個 query**；每個 query 只屬於 1 個 batch。
- `batch_id → metric_batches.id`：外鍵放在「多」的那一邊（metric_queries.batch_id），值指向 metric_batches.id。
- FK 保證：不能插入 `batch_id = 999` 卻沒有 id=999 的 batch（DB 會報錯）。

**Q：同一個 keyword 在好幾個 batch 都有出現，是取最新的嗎？**
- 跨 batch **不去重、不合併**：每個 batch 各存一份獨立的 row（不同 id、各自的 frequency / tags）。DB 也沒有 `UNIQUE(batch_id, keyword)` 限制。
- 真實例子 `mexico`：出現在 10 個 batch，每個 batch 都有一個獨立的 row：
  ```
  batch 1  id=39      freq=570200  tags=["head"]
  batch 2  id=18962   freq=500     tags=[]
  ...
  batch 8  id=100115  freq=44200   tags=["head"]
  batch 18 id=122911  freq=10000   tags=[]
  ```
  dump 統計：91,960 個不同 keyword，其中 22,828 個出現在 ≥2 個 batch；**同一 batch 內重複 = 0**。
- 「取最新」發生在**選 batch** 的時候，不是選 keyword：
  - `measure.py:get_latest_batch_id` → `ORDER BY id DESC LIMIT 1` 選出最新 batch。
  - `*QueryStrategy` 和 `CrawlerAllMetricMeasure` 都只處理這個 batch_id → 舊 batch 的 row 只是歷史紀錄。
- 同一 batch 內不重複，是 Python 保證的，DB 沒有限制：
  - `_fetch_from_api_process_and_store` 用 `merged_dict[kw]` 合併多國（geo 合併、frequency 相加）。
  - `getGoldenSet` 先 `filter_by(batch_id, keyword).first()`，有就只補 tag，沒有才 INSERT。

**Q：所以 metric_queries 是 inverted table？可以查 keyword 最早在第幾個 batch 出現？**
- 不算 inverted index。它是正規化的「子表 / 事件表」：一個 row = 「keyword 在某 batch 出現一次」。
  - inverted index 的形狀是 `term → [doc1, doc2, ...]`，key 是 term，value 是一串 posting list。
  - metric_queries 的形狀是 `(batch_id, keyword, ...)` 一列一筆，是攤平的。
  - 可以「當成」inverted index 來查（GROUP BY keyword），只是沒有 keyword index，所以要掃全表。
- 這裡真正的 inverted index 是 **GIN index on tags**：PostgreSQL 的 GIN 就是倒排索引（tag 值 → 含有它的 row 列表），所以 `tags @> '["head"]'` 很快。
- 查詢「最早出現 / 出現幾次」：
  ```sql
  SELECT keyword,
         MIN(batch_id)                      AS first_batch,
         COUNT(DISTINCT batch_id)           AS n_batches,
         array_agg(batch_id ORDER BY batch_id) AS batches   -- ← 這行把它「轉成」inverted list
  FROM metric_queries
  GROUP BY keyword
  ORDER BY n_batches DESC;
  ```
  dump 結果：`brad pitt / china / donald trump ...` first_batch=1，10 個非空 batch 全出現。
  每個 batch「首次出現」的新 keyword 數：1:10710, 2:10188, 3:9672, 4:12939, 5:11209, 6:9801, 7:8479, 8:5320, 9:5068, 18:8574。
- 注意：用 `MIN(batch_id)` 當「最早」成立，是因為 id 由 sequence 遞增，跟 created_at 同序；更嚴謹是 JOIN metric_batches 取 `MIN(created_at)`。
- 若常查，可加 `CREATE INDEX ON metric_queries (keyword);`（目前沒有）。

**Q：不同 batch 的同一個 query 可能產生相同 URL，怎麼處理？**（也包括同一 batch 內不同 query 撞到同一 URL）
- 儲存層：`metric_url` **不去重**。每個 (query, rank) 一列；同一 URL 可在不同 query / 不同 batch 出現多次。dump 共 146,101 列，distinct URL 只有 125,278 個。
- 三種「重複」各自怎麼處理：

| 情境 | 程式怎麼處理 | 位置 |
|------|-------------|------|
| 跨 batch 同 URL | 不處理；measure 只讀最新 batch，舊 batch 的列不參與計算 | `measure.py:get_latest_batch_id` |
| 同 batch、不同 query 撞到同 URL（例如 wikipedia、fifa.com） | 計算時去重：`url_id_map = {url: [metric_url.id, ...]}`；統計 **以 distinct URL 計 1 次**；結果寫回所有 id | `CrawlerAllMetricMeasure.test` 步驟 1、3 |
| 同 batch、同 query 重跑（或同時 head+random） | 先 `DELETE FROM metric_url WHERE query_id=?` 再 INSERT；後跑的 strategy 會覆蓋前一次的 URL | `*QueryStrategy.getGoldenSet` C 段 |

- 驗證：batch 2 head 共 7,628 列、7,057 個 distinct URL；`metric_headset_total` 2026-03-16 的 `total = 7057` → coverage 的分母是 **distinct URL**，不是列數。
  而 `meta_tag_stats.head.urls`（7,628）是列數，兩者口徑不同。
- 等價 SQL（coverage 分母）：
  ```sql
  SELECT COUNT(DISTINCT mu.url)
  FROM metric_url mu JOIN metric_queries mq ON mq.id = mu.query_id
  WHERE mq.batch_id = 2 AND mq.tags @> '["head"]';
  ```
- 延伸觀察：相鄰 batch 的 URL 重疊很少（batch 8 vs 9 只有 223 個共同 URL）→ dashboard 上跨 batch 的 coverage 跳動，**主要是換了一組 URL**，不全是 crawler 進步。⚠️ 更正：coverage 並不是每天量，cron 只在建 batch 那天（每月 1、16 號）量一次，所以 dashboard 上每個點幾乎都對應不同 batch（見附錄 C.4）。
- 若想追蹤「某 URL 隨時間是否被爬到」：用 `GROUP BY url` 跨 batch 看 `bool_or(is_crawled)`；或另建 `url` 維度表（url UNIQUE），metric_url 改存 url_id（正規化）。

**Q：上面說的是 measure 階段的處理。那 rank 呢？rank 會影響 URL 有沒有被挑出來嗎？**
- 「挑選」只發生在 **keyword 層**（head 依 frequency 取前 N 名，random 隨機抽 N 個）。**URL 層沒有挑選**：SerpApi 回傳的 organic results（`num=10`）全部存進去。
- `rank` 只是 `enumerate(url_list)` 的 `idx + 1`（Google 上的名次），**目前沒有任何程式讀它**：
  - coverage 計算時每個 URL 權重相同，rank 1 和 rank 10 一樣算 1 個。
  - 同一 URL 在不同 query 有不同 rank，`url_id_map` 去重時直接忽略 rank。
  - `is_ranked`、`ranked_num`、`ranked_rate` 寫死為 0；`TypesenseRankMeasure.test()` 是空的 stub（舊邏輯被註解掉，原本是去 Typesense 搜 keyword、看 golden URL 有沒有在前 10 名）。
- dump 裡 rank 的分佈：rank 1~5 約 17.7k 筆，rank 9 只有 6k、rank 10 只有 1.8k → 很多 query 拿不到 10 筆（organic 結果被廣告、新聞框擠掉）。每個 query 平均 7~9 個 URL；同一 query 內沒有重複 URL。
- 可以延伸的分析：`GROUP BY rank` 看「Google 排越前面的 URL，是否越容易被我們的 crawler 爬到」。

**Q：head + random 是什麼意思？**
- 兩種**挑 keyword 的策略**，從同一個 batch 的 trending 清單（rawData）挑：
  - **head**：依 frequency 排序，取前 N 名 → 最熱門的詞（`HeadQueryStrategy`）。
  - **random**：`random.sample` 隨機抽 N 個 → 代表一般的詞（`RandomQueryStrategy`）。
- random 從**全部**詞裡抽，所以也可能抽到熱門詞 → 同一 keyword 被兩邊都選中 → **同一個 metric_queries row** 的 tags 變成兩個（不會建第二列，因為先 `filter_by(batch_id, keyword).first()`）。
- tags 的順序 = 誰先跑。Dockerfile cron：每月 1、16 號 12:00 跑 random，18:00 跑 head → 正常會是 `["random","head"]`；batch 1 也有 `["head","random"]`，可能是手動跑的。
- 後跑的會 `DELETE` 這個 query 的 URL 再重新呼叫 SerpApi 插入 → 兩個 set 共用**同一組 metric_url 列**（後跑那次的結果）。
- 之後 measure 用 `tags @> '["head"]'` 和 `tags @> '["random"]'` 分別算，所以這些 URL 會同時算進 headset 和 randomset。
- 觀察：Dockerfile 寫 `--keywordNums 50`，但 dump 裡每個 batch head 的 queries 都是 1000 → 實際部署的參數和 repo 的 Dockerfile 不一樣（待確認）。

### 1.5 小練習（下次上課先問）
1. 寫 SQL：列出 batch 7 中「同時」被選進 head 和 random 的關鍵字。（提示：`@>` 可以放兩個元素）
2. 為什麼 `metric_url` 沒有 `batch_id` 欄位，卻還能查「batch 7 的所有 head URL」？寫出 SQL。

<details><summary>參考答案</summary>

```sql
-- 1
SELECT keyword FROM metric_queries
WHERE batch_id = 7 AND tags @> '["head","random"]'::jsonb;

-- 2 透過 query_id JOIN 回 metric_queries 拿 batch_id（正規化設計）
SELECT mu.*
FROM metric_url mu
JOIN metric_queries mq ON mq.id = mu.query_id
WHERE mq.batch_id = 7 AND mq.tags @> '["head"]'::jsonb;
```
</details>

---

## 第 2 課：`DatabaseRawDataReader` —— 用快取還是重抓？

檔案：`Metric/RawDataReader/DatabaseRawDataReader.py`。由 `measure.py:createDataset` 呼叫（只有加 `--create` 才會跑）。
任務：產生「這一輪的 trending keyword 清單」(rawData)，交給第 4 課的 strategy 挑選。

### 2.1 流程

```
readData()
 ├─ ① 取最新 batch：ORDER BY created_at DESC LIMIT 1
 ├─ 有，且 (now - created_at).days < update_day (預設 14)
 │     └─ ② _fetch_from_db(batch_id)  → 回傳該 batch 所有 query（快取，不花 SerpApi 額度）
 └─ 沒有 / 過期
       ├─ ③ _fetch_trending_now(geo) × 10 國（SerpApi google_trends_trending_now, hours=168）
       ├─ ④ 合併同 keyword（geo 併成 list、frequency 相加）→ 依 frequency 排序
       └─ ⑤ _save_to_database：INSERT 1 個 batch + N 個 query（tags = []）
```

### 2.2 等價 SQL

```sql
-- ① 最新 batch（created_at 沒有 index → 全表掃描；只有 18 列所以無所謂）
SELECT * FROM metric_batches ORDER BY created_at DESC LIMIT 1;

-- 判斷是否過期（Python: delta.days < self.update_day）
SELECT id, now() - created_at AS age,
       EXTRACT(day FROM now() - created_at) < 14 AS use_cache
FROM metric_batches ORDER BY created_at DESC LIMIT 1;

-- ② 讀快取（走 ix_metric_queries_batch_id；Python 端再依 frequency 排序）
SELECT keyword, frequency, geo
FROM metric_queries
WHERE batch_id = :batch_id
ORDER BY frequency DESC;

-- ⑤ 寫入（同一個 transaction）
BEGIN;
INSERT INTO metric_batches (created_at, meta_total_queries, meta_total_urls, meta_tag_stats, meta_geo_counts)
VALUES (now(), 10003, 0, '{}', '{"US": 2237, "JP": 2218, ...}')
RETURNING id;                       -- ← session.flush() 就是為了拿到這個 id
INSERT INTO metric_queries (batch_id, keyword, geo, frequency, tags)
VALUES (:id, 'gold rate today', '["IN"]', 10000000, '[]'),
       (:id, ...), ...;             -- 每個 keyword 一列
COMMIT;
```

重點：
- `session.flush()` = 把 INSERT 先送到 DB（拿到 sequence 產生的 id），但還沒 COMMIT；後面的 query 才能填 `batch_id`。
- 全部在同一個 transaction：中途出錯 → `Database.session()` 會 `rollback()` → 不會留下「有 batch 但只有一半 query」的狀態。
- `meta_geo_counts` 在這裡算好；`meta_total_urls = 0`、`meta_tag_stats = {}`，要等第 4/5 課的 strategy 跑完才會更新。
- 快取的意義：random（12:00）和 head（18:00）是兩次獨立執行；第二次讀到同一個 batch → 兩個 strategy 共用同一份 keyword 清單、同一個 batch_id。

### 2.3 坑：空 batch（batch 10~17）

dump 實際資料：

| id | created_at | meta_total_queries |
|----|-----------|-------------------|
| 9 | 2026-06-12 14:00 | 10003 |
| 10~17 | 2026-06-19 ~ 08-07，每週五 14:15 | **0** |
| 18 | 2026-09-17 06:09 | 14761 |

成因（讀程式推出來的）：
1. `_fetch_trending_now` 重試 4 次都失敗時**回傳 `[]`，不丟例外**（例如 API key 失效、額度用完）。
2. 10 國都失敗 → `all_data = []` → `_save_to_database([])` 照樣建一個 0 query 的 batch。
3. strategy 拿到空清單 → 什麼都不做 → 也沒有 URL、沒有 coverage。
4. 更糟的是：這個空 batch 是「新的」→ 接下來 update_day 天內都會被當成有效快取 → 即使 API 恢復了也不會重抓。

修正方向：rawData 為空（或太少）時不要建 batch、直接報錯或結束；或 `readData` 用快取前檢查 `meta_total_queries > 0`。

另外的觀察：空 batch 每 7 天出現一次、時間是週五 14:15 → 實際部署的排程與 `--update` 參數和 repo 的 Dockerfile（每月 1、16 號，預設 14 天）不一樣（與 1.6 的 keywordNums 1000 vs 50 是同一類問題）。

**影響範圍（學生補充：batch 週期調整過；只需要 2026-10 起的數據）**
- 對歷史資料：空 batch 不會產生錯誤數字，只會讓 coverage **斷層**（headset/randomset 從 06-05/06-12 直接跳到 09-17）。crawler_stat 不受影響（不用 batch）。→ 10 月起的分析可以忽略 batch 10~17。
- 對 10 月起的資料：程式的問題仍在。只要某次 SerpApi 失敗，就會建空 batch → `get_latest_batch_id` 選到它 → `CrawlerAllMetricMeasure` 印 "No URLs found" 直接結束 → 那一輪**沒有 coverage**，而且快取期間內不會自動重抓。
- dump 現況：coverage 最後一筆是 2026-09-17（batch 18）；**10 月還沒有任何 coverage 資料**，crawler_stat 只有 10-01 一筆。

### 2.4 其他細節
- `created_at` 用 `datetime.now()`（無時區）→ 取決於執行機器的時區。
- `started`（trending 開始時間）有抓、合併時取最早，但**沒存進 DB** → 從快取讀回來就沒了。
- 合併時各國 frequency 直接相加，沒保留各國分量（見待處理第 3 項）。
- `hours=168` = 抓過去 7 天的 trending。

### 2.7 Q&A：發生 rollback 時，是全空還是保留已完成的部分？（學生問，2026-10-04）

先分清楚：**rollback 只在「丟出例外」時發生**（DB 錯誤、程式 bug、連線中斷；程序被 kill 時，未 commit 的 transaction 也會被 DB 自動丟棄，效果相同）。SerpApi 失敗**不算**，它只回傳 `[]`。

| 階段 | commit 粒度 | 中途出例外的結果 |
|------|------------|----------------|
| `_save_to_database`（batch + 全部 query） | 最後 commit 1 次 | **全空**：batch 和 query 都不會留下 |
| `getGoldenSet`（第 4 課） | **每個 keyword commit 1 次** | **保留已完成的 keyword**（它們的 tag 和 URL 都在），只丟掉出錯的那一個；後面的 keyword 沒跑；`_update_batch_stats` 沒跑 → `meta_tag_stats` 數字過時 |
| `CrawlerAllMetricMeasure`（回填 is_* + 寫 coverage 3 列） | 最後 commit 1 次 | 全空：metric_url 狀態不更新、coverage 不寫 |
| `CrawlerStatusMeasure`（crawler_stat 3 列） | 最後 commit 1 次 | 全空：3 列都不寫（Total/A/B 不會只寫一半） |

SerpApi 失敗時（沒有例外、會 commit）：
- 抓 trending 失敗 → 空 batch 被 commit（2.3）。
- 某 keyword 搜尋失敗 → 該 query 照樣加上 tag、URL 0 筆，commit。dump 中有 1,416 個這種 query（batch 9 就有 1,029 個，推測當時額度用完）。

### 2.5（原第 3 課）`get_latest_batch_id` vs `readData` 的「最新」

```sql
-- measure.py:get_latest_batch_id   → 給 strategy / coverage 用
SELECT id FROM metric_batches ORDER BY id DESC LIMIT 1;          -- 走 PK index，只取 id
-- readData                          → 決定用快取還是重抓
SELECT * FROM metric_batches ORDER BY created_at DESC LIMIT 1;   -- 無 index
```
- 兩個「最新」定義不同（id vs created_at），正常情況下一致（id 由 sequence 遞增，created_at 也遞增）。
- 若有人手動插入舊時間的 batch，兩者會不一致 → strategy 可能寫到錯的 batch。比較穩的寫法：readData 回傳它用的 batch_id，createDataset 直接沿用，不要再查一次。

### 2.6 小練習
1. 寫 SQL 找出所有空 batch（沒有任何 query 的 batch）。注意：不要只看 `meta_total_queries`，要真的去數 metric_queries。
2. 假設今天是 2026-09-25，`--update 14`，執行 `measure.py --create` 時會用快取還是重抓？用的是哪個 batch？
3. （複習第 1 課）batch 18 的 `tags = []` 的 query 有幾個 URL？為什麼？

<details><summary>參考答案</summary>

```sql
-- 1  LEFT JOIN + IS NULL = 找「沒有對應子資料」的母資料
SELECT b.id, b.created_at
FROM metric_batches b
LEFT JOIN metric_queries q ON q.batch_id = b.id
WHERE q.id IS NULL
ORDER BY b.id;
-- 或 NOT EXISTS 寫法
SELECT id FROM metric_batches b
WHERE NOT EXISTS (SELECT 1 FROM metric_queries q WHERE q.batch_id = b.id);
```
2. 最新 batch 是 18（2026-09-17 06:09），到 09-25 約 8 天，8 < 14 → 用快取，batch 18。
3. 0 個。tags=[] 表示沒被 head/random 選中，沒呼叫 SerpApi，所以沒有 URL。
</details>

---

## 第 4 課：`getGoldenSet` —— 挑 keyword、搜 Google、存 URL

檔案：`Metric/Query/HeadQueryStrategy.py`、`RandomQueryStrategy.py`（兩檔幾乎一樣，只差「怎麼挑」和 tag 名稱）。
輸入：第 2 課的 rawData（該 batch 全部 trending keyword）+ `batch_id`（第 2 課 2.5 的最新 batch）。

### 4.1 第一步：挑 keyword（在 Python 做，不是 SQL）

| | 程式 | 等價 SQL |
|--|------|---------|
| head | `sorted(rawData, key=frequency, reverse=True)[:N]` | `SELECT * FROM metric_queries WHERE batch_id=:b ORDER BY frequency DESC LIMIT :N;` |
| random | `random.sample(rawData, min(len, N))` | `SELECT * FROM metric_queries WHERE batch_id=:b ORDER BY random() LIMIT :N;` |

- random 沒有設 seed → 每次跑抽到的不一樣，無法重現。
- head 的 frequency 是多國加總（第 2 課 ④）→ 多國都熱的詞比較容易進 head。

### 4.2 第二步：每個 keyword 跑一次這個迴圈（一個 keyword = 一個 transaction）

```
for keyword in 選中的:
  A. url_list = getQuery(keyword)                    ← SerpApi，花額度；失敗回傳 []
  B. 這個 batch 已經有這個 keyword 嗎？
       有 → 在 tags 補上 "head"（UPDATE）
       沒有 → 新建一列 tags=["head"]（INSERT + flush 拿 id）
  C. DELETE 這個 query 的舊 URL → INSERT 新 URL（rank = 1, 2, 3…）
  D. COMMIT
最後：_update_batch_stats（第 5 課）
```

等價 SQL（一個 keyword）：
```sql
BEGIN;
-- B. 查有沒有（batch_id 有 index，keyword 沒有 → 先用 batch_id 縮小範圍再過濾）
SELECT * FROM metric_queries WHERE batch_id = :b AND keyword = :kw LIMIT 1;

-- B-1. 有 → 補 tag（Python 版：讀出 list、append、整個寫回）
UPDATE metric_queries SET tags = '["random","head"]' WHERE id = :id;
--      PostgreSQL 一句話版本（|| = 串接 jsonb 陣列）：
-- UPDATE metric_queries SET tags = tags || '["head"]'
-- WHERE id = :id AND NOT tags @> '["head"]';

-- B-2. 沒有 → 新增
INSERT INTO metric_queries (batch_id, keyword, geo, frequency, tags)
VALUES (:b, :kw, :geo, :freq, '["head"]') RETURNING id;

-- C. 先刪後插（防止重跑時 URL 重複）
DELETE FROM metric_url WHERE query_id = :id;
INSERT INTO metric_url (query_id, url, rank, is_discovered, is_crawled, is_indexed, is_ranked)
VALUES (:id, 'https://en.wikipedia.org/wiki/Cuba', 1, false, false, false, false),
       (:id, 'https://cri.fiu.edu/...',           2, false, false, false, false), ...;
COMMIT;
```

### 4.3 重點觀念

1. **幾乎都走「有 → UPDATE」**：rawData 本身就是從同一個 batch 讀出來的（或剛存進去的），所以 keyword 一定已經存在，INSERT 分支實務上幾乎不會發生。tags 從 `[]` → `["head"]` 就是這裡造成的。
2. **為什麼要 `current_tags = list(query_obj.tags)` 再整個指定回去？** SQLAlchemy 預設**偵測不到 JSONB 內部的修改**（`query_obj.tags.append("head")` 不會產生 UPDATE）。複製一份新 list 再指定回去，SQLAlchemy 才看得到「欄位被換掉了」→ 產生 UPDATE。（另一種做法：欄位用 `MutableList.as_mutable(JSONB)`）
3. **先 DELETE 再 INSERT** = 簡易版「覆蓋」。好處：重跑不會重複；壞處：SerpApi 失敗時會清掉舊資料（待處理第 5 項）。
4. **每個 keyword commit 一次**：跑到第 600 個掛掉，前 599 個保留（第 2 課 2.7）。代價是每個 keyword 都有多次 DB 往返，但比起 SerpApi 呼叫（秒級）可忽略。
5. 新 URL 的 `is_*` 都是 false、`shard_id` 是 NULL → 要等第 9 課的 `CrawlerAllMetricMeasure` 回填。

### 4.4 坑：重跑 random 會「越選越多」

- 重跑 random 時會**再抽一批不同的** keyword 加上 "random"，但**不會移除上一次的 "random" tag**。
- dump 證據：batch 1 的 random query 有 1,313 個（1206 + 36 + 71），比 N=1000 多 → 推測 random 在 batch 1 跑過不只一次。meta_tag_stats 也記錄 `"random": {"queries": 1313}`。
- head 重跑不會有這問題（每次都是同樣的前 N 名），只是重抓 URL。
- 修正方向：跑 random 前先清掉該 batch 的 random tag：
  ```sql
  UPDATE metric_queries SET tags = tags - 'random'   -- jsonb 的 - 運算子：移除陣列中的字串元素
  WHERE batch_id = :b AND tags @> '["random"]';
  ```

### 4.5 小練習
1. 寫 SQL：找出 batch 9 中「有 random tag 但 0 個 URL」的 keyword。（提示：第 2 課練習的 LEFT JOIN + IS NULL）
2. 用一句 SQL 算出每個 batch 的 random query 數，找出哪些 batch 超過 1000。
3. 如果把 `current_tags = list(query_obj.tags)` 改成直接 `query_obj.tags.append("head")`，會發生什麼事？

<details><summary>參考答案</summary>

```sql
-- 1
SELECT q.keyword
FROM metric_queries q
LEFT JOIN metric_url u ON u.query_id = q.id
WHERE q.batch_id = 9 AND q.tags @> '["random"]' AND u.id IS NULL;

-- 2
SELECT batch_id, COUNT(*) AS n_random
FROM metric_queries
WHERE tags @> '["random"]'
GROUP BY batch_id
HAVING COUNT(*) > 1000;   -- HAVING = 對 GROUP BY 之後的結果過濾（WHERE 是對分組前的列過濾）
```
3. Python 物件裡的 list 變了，但 SQLAlchemy 不知道 → commit 時不會送 UPDATE → DB 裡的 tags 沒變；之後 measure 用 `@> '["head"]'` 查不到這個 query → 它的 URL 不會進 coverage。
</details>

---

## 第 5 課：`_update_batch_stats` —— 把統計數字寫回 batch

位置：`HeadQueryStrategy._update_batch_stats` / `RandomQueryStrategy._update_batch_stats`，在 `getGoldenSet` 的迴圈全部跑完後呼叫一次。
目的：更新 `metric_batches` 上的 `meta_tag_stats`、`meta_total_queries`、`meta_total_urls`（給人看的摘要，measure 不會用到）。

### 5.1 四個 COUNT（以 head 為例）

| Python（ORM） | 等價 SQL |
|---------------|---------|
| `session.get(MetricBatch, id)` | `SELECT * FROM metric_batches WHERE id = :b;`（用 PK） |
| head query 數 | `SELECT COUNT(*) FROM metric_queries WHERE batch_id = :b AND tags @> '["head"]';` |
| head URL 數 | `SELECT COUNT(*) FROM metric_url mu JOIN metric_queries mq ON mq.id = mu.query_id WHERE mq.batch_id = :b AND mq.tags @> '["head"]';` |
| 全 batch query 數 | `SELECT COUNT(*) FROM metric_queries WHERE batch_id = :b;` |
| 全 batch URL 數 | `SELECT COUNT(*) FROM metric_url mu JOIN metric_queries mq ON mq.id = mu.query_id WHERE mq.batch_id = :b;` |

- 第 2 句會同時用到兩個 index：`batch_id`（btree）和 `tags`（GIN）→ PostgreSQL 可用 BitmapAnd 把兩個條件的結果交集。
- `session.query(...).count()` 實際送出的是 `SELECT count(*) FROM (SELECT ... ) AS anon_1`（包一層子查詢），結果相同。
- 一句話算完的寫法（FILTER，第 6 課會細講）：
  ```sql
  SELECT COUNT(*)                                   AS total_q,
         COUNT(*) FILTER (WHERE tags @> '["head"]') AS head_q,
         COUNT(*) FILTER (WHERE tags @> '["random"]') AS random_q
  FROM metric_queries WHERE batch_id = :b;
  ```

### 5.2 寫回 JSONB：read-modify-write

```python
current_stats = dict(batch.meta_tag_stats)          # 讀出 + 複製（同第 4 課：要換新物件 SQLAlchemy 才偵測得到）
current_stats['head'] = {"queries": ..., "urls": ...} # 只改自己的 key，保留 random 的
batch.meta_tag_stats = current_stats                  # 整個寫回 → UPDATE
```
等價 SQL（PostgreSQL 一句話版，`||` 合併兩個 jsonb 物件，同 key 會被右邊覆蓋）：
```sql
UPDATE metric_batches
SET meta_tag_stats = meta_tag_stats || jsonb_build_object('head', jsonb_build_object('queries', 1000, 'urls', 7746)),
    meta_total_queries = 15235,
    meta_total_urls = 15113
WHERE id = :b;
```
- read-modify-write 的風險：若 head 和 random **同時**跑，兩邊各讀到舊值再寫回 → 後寫的蓋掉先寫的（lost update）。目前 12:00 / 18:00 分開跑，所以沒事；`||` 寫法在 DB 內完成，不會有這問題。

### 5.3 三個口徑要分清楚

| 數字 | 算法 | batch 7 head |
|------|------|-------------|
| `meta_tag_stats.head.urls` | metric_url **列數** | 7,746 |
| coverage `total`（第 9 課 / 附錄 C） | **distinct URL** | 7,335 |
| `meta_total_urls` | 全 batch 列數；head+random 共用的 query 只算一次 | 15,113（≠ 7,746 + 7,926 = 15,672） |

- 15,672 − 15,113 = 559 = 同時是 head 和 random 的 query 的 URL 被算了兩次。

### 5.4 用 dump 驗證：meta 和實際數字對不上

把每個 batch 的 meta 跟實際重算比較：batch 7、10~18 完全一致；其他有落差：

| batch | 欄位 | meta | 實際 |
|------|------|------|------|
| 1 | head urls | 7,576 | 7,574 |
| 2 | random urls | 8,039 | 8,034 |
| 3 | random urls | 8,045 | 8,056 |
| 5 | random urls | 13,387 | 13,393 |
| 6 | random urls | 8,197 | 8,203 |
| 8 | random urls | 7,990 | 7,991 |
| **9** | **random urls** | **7,792** | **7,179**（少 613） |

- 小落差（個位數）的原因：random 12:00 先跑並寫好統計 → head 18:00 跑時，**兩邊都選中的 query 會被 DELETE + 重新搜尋**（第 4 課 C 段），URL 數可能變多或變少 → 但 head 的 `_update_batch_stats` **只更新 head 的 key** → random 的數字就過時了。（batch 1 是 head 落差，因為 batch 1 有 head 先跑的情況。）
- 這是「把統計結果另外存一份（denormalized / cached aggregate）」的典型問題：來源資料變了，存起來的摘要不會自動跟著變。
- **batch 9 少 613 筆無法用上面原因解釋**（batch 9 沒有 head）→ 統計寫入後，有 613 筆 random URL 被刪掉；可能是重跑 random 時 SerpApi 失敗（待處理 #5）但程式在 `_update_batch_stats` 前就中斷，或手動刪除。待查。
- **影響評估（學生問）**：
  - 不影響：coverage、crawler_stat、Superset dashboard（measure 與 bootstrap.py 都不讀 metric_batches 的 meta 欄位）。
  - 會影響：有人直接看 meta 寫報告、估 golden set 大小或 SerpApi 用量時，會拿到過時數字（多為個位數，影響小）。
  - **batch 9 才是重點**：randomset 06-12 記錄的 total = 7,738 個 distinct URL，但現在 metric_url 裡 batch 9 random 只剩 7,139 個 → URL 是在**量測之後**被刪的。當時的 coverage 數字沒錯，但**現在已無法用 metric_url 重現/驗證那筆 coverage**（歷史紀錄和原始資料不一致）。
- 結論：**`meta_tag_stats` 只能當參考，要精確數字就用 5.1 的 SQL 現算。** measure 本身不讀 meta，所以 coverage 不受影響。

### 5.5 小練習
1. 用一句 SQL，列出每個 batch 的 head query 數、random query 數、兩者都有的 query 數。
2. 寫 SQL：算出 batch 7 head 的「列數」和「distinct URL 數」，驗證 7,746 和 7,335。
3. 為什麼 `meta_total_urls` 不等於 head urls + random urls？

<details><summary>參考答案</summary>

```sql
-- 1
SELECT batch_id,
       COUNT(*) FILTER (WHERE tags @> '["head"]')          AS head_q,
       COUNT(*) FILTER (WHERE tags @> '["random"]')        AS random_q,
       COUNT(*) FILTER (WHERE tags @> '["head","random"]') AS both_q
FROM metric_queries
GROUP BY batch_id ORDER BY batch_id;

-- 2
SELECT COUNT(*) AS n_rows, COUNT(DISTINCT mu.url) AS n_distinct
FROM metric_url mu JOIN metric_queries mq ON mq.id = mu.query_id
WHERE mq.batch_id = 7 AND mq.tags @> '["head"]';
```
3. 同時有 head 和 random 的 query 共用同一組 metric_url 列：在 head urls、random urls 各算一次，但在 meta_total_urls 只算一次。
</details>

---

## 第 6 課：`CrawlerStatusMeasure._scan_shard` —— `COUNT(*) FILTER` 與 Overview 圖

檔案：`Metric/Measure/CrawlerStatusMeasure.py`。cron 每小時跑（`--measure status`），結果寫進 `crawler_stat_{total,a,b}`，就是 Superset「Total Overview Volumn」的資料來源（附錄 A.2）。
這是**第二條管線**：只讀 crawlerdb，不碰 batch / metric_url（附錄 A.6）。

### 6.1 一個 shard 的查詢

```python
stmt = select(
    func.count(),                                              # discovered
    func.count().filter(UrlState.last_fetch_ok.is_not(None))   # crawled
)
```
```sql
SELECT COUNT(*)                                          AS discovered,
       COUNT(*) FILTER (WHERE last_fetch_ok IS NOT NULL) AS crawled
FROM url_state_current_042;
```
→ 256 張表各跑一次（16 threads），shard 0-127 加到 A、128-255 加到 B，兩者都加到 Total。

### 6.2 `FILTER` 和它的三種等價寫法

| 寫法 | 說明 |
|------|------|
| `COUNT(*) FILTER (WHERE last_fetch_ok IS NOT NULL)` | PostgreSQL 9.4+ 標準語法，最清楚 |
| `SUM(CASE WHEN last_fetch_ok IS NOT NULL THEN 1 ELSE 0 END)` | 所有資料庫都能用的舊寫法 |
| `COUNT(CASE WHEN last_fetch_ok IS NOT NULL THEN 1 END)` | CASE 沒有 ELSE → NULL → COUNT 不算 |
| **`COUNT(last_fetch_ok)`** | 這題最簡單：`COUNT(欄位)` 本來就只數非 NULL |

觀念：**`COUNT(*)` 數列數；`COUNT(欄位)` 數該欄非 NULL 的列數**。FILTER 的價值在於條件複雜時（例如 `WHERE last_fetch_ok >= now() - interval '7 days'`），一次掃表就能算出多個不同條件的計數。

### 6.3 效能：每小時掃 63 億列

- discovered ≈ 63 億 / 256 ≈ 每張表 2,500 萬列；`COUNT(*)` 在 PostgreSQL 要掃整張表（MVCC：每列對每個交易的可見性不同，沒有現成的總數）。
- 每小時 × 256 張 → 對 crawlerdb 是持續的讀取壓力，會和 crawler 本身搶 I/O。
- 替代方案：
  - 只需要大概值：`SELECT reltuples FROM pg_class WHERE relname = 'url_state_current_042'`（統計估計值，瞬間回傳，但不精確，不能拆 crawled）。
  - 降低頻率：snapshot 一天量一次就夠（flow 的部分在第 7 課另算）。
  - partial index：`CREATE INDEX ... ON url_state_current_042 (url) WHERE last_fetch_ok IS NOT NULL` → 可走 index-only scan（仍需掃 index，但比整張表小）。

### 6.4 坑：失敗時「安靜地回傳 0」

```python
except Exception as e:
    return shard_id, 0, 0, 0
```
- 任何一張表查詢失敗（逾時、連線斷、表被鎖），該 shard 就被當成 0，**總數直接少掉約 1/256，沒有任何錯誤訊息**；而且 UPSERT 會把當天先前正確的值蓋掉。
- 建議：失敗就重試；仍失敗則整次不寫入（或寫 NULL）並記錄 log。
- 判讀技巧：shard 失敗時，**discovered 和 crawled 會「同時」掉**。
**Q：查詢失敗會留下 log 嗎？（學生問，2026-10-04）**
- `CrawlerStatusMeasure._scan_shard`：`except Exception as e: return shard_id, 0, 0, 0` → **完全沒有 print / log**，例外訊息 `e` 被丟掉。
- `CrawlerAllMetricMeasure._scan_url_shard` / `_scan_domain_shard`：只 print `"[Error] scan url_state_current failed"`，**沒有 shard 編號、沒有例外內容** → 有 log 也查不出原因。
- `_load_indexed_urls`（selectdb）沒有 try → 失敗會讓整次 coverage 中止、不寫入。
- 整個 repo 沒用 `logging` 模組；cron 把 stdout/stderr 導到容器內的 `/var/log/cron.log`（Dockerfile），沒有看到掛 volume、沒有 rotation → 容器重建就消失。是否保留要看實際部署。

**Q：查詢失敗可能的原因？**

| 原因 | 說明 | 和本專案的關聯 |
|------|------|---------------|
| 表不存在 / 被換掉 | 修剪若用「建新表 → DROP 舊表 → RENAME」，中間查詢會 `UndefinedTable` | 1TB 不夠、會刪 URL（6.5） |
| 鎖 | 修剪時 `TRUNCATE` / `VACUUM FULL` / `ALTER` 會拿排他鎖 → COUNT 等待；目前**沒設 statement_timeout**，比較可能是「卡很久」而不是失敗 → 下一個小時的 cron 疊上去 | 修剪 |
| 磁碟滿 | DB 無法寫 WAL / 暫存檔 → 錯誤或整個 DB 停擺 | 1TB 不夠 |
| 連線數不足 | 16 threads + 每個 engine pool_size 40 + max_overflow 40；status / coverage / migrate 都整點跑，加上 crawler 本身的連線 → 超過 `max_connections` | 整點同時跑多個 cron |
| 連線中斷 / DB 重啟 | 網路、OOM、維護重啟 | — |
| 讀 replica 時的衝突 | 若 crawlerdb 是 standby：`canceling statement due to conflict with recovery` | 未知部署 |
| 權限 / 帳號變更 | crawler 端改 schema 或權限（git 有 `align with new crawler schema` 的 commit） | 曾發生 schema 調整 |
| 執行緒安全（機率低） | 16 threads 同時動態建立 ORM class，修改同一個 SQLAlchemy registry | 程式設計 |

- 建議：用 `logging` 記錄 shard 編號 + 例外訊息；設 `statement_timeout`；失敗重試；仍失敗則不寫入（或寫 NULL）並記錄 `failed_shards` 數；log 寫到有 volume 的位置。


### 6.5 用 dump 驗證：2026-05-24 discovered 掉了 20%

| 日期 | A discovered | B discovered | A crawled | B crawled |
|------|-------------|-------------|-----------|-----------|
| 05-23 | 26.5 億 | 22.5 億 | 4.34 億 | 4.31 億 |
| 05-24 | **20.0 億** | **19.3 億** | 4.35 億 ↑ | 4.31 億 ↑ |
| 05-25 | 19.2 億 | 18.7 億 | 4.48 億 ↑ | 4.44 億 ↑ |

- Total discovered：48.97 億 → 39.22 億（−19.9%），隔天再 −3.2%。
- 但 crawled 持續上升 → **不是 shard 掃描失敗**（那樣 crawled 也會掉），比較像 crawler 端**清掉了一批沒抓過的 URL**（例如修剪 frontier）。待向 crawler 端確認；不在 10 月範圍內，但說明 discovered「不一定只增不減」。
- **學生確認原因（2026-10-04）**：crawler 儲存空間 1TB 不夠用，所以會刪 URL；由 crawler 的 rank（URL / domain 分數，例如 `url_state.url_score`、`domain_score`）決定保留哪些。→ 05-24 是**刻意修剪**，不是 bug。
  - 名詞注意：這個「crawler 的 rank（保留優先度）」≠ `metric_url.rank`（Google 名次）≠ `ranked_*`（我們搜尋引擎排序指標，待處理 #9）。三個 rank 意思不同。
  - 對 metric 的意涵：
    1. Overview 的 discovered = 「**目前還留著**的 URL 數」，不是「累計發現過」→ 會下降；定義要寫清楚。
    2. coverage 的 discovered 也會因為修剪而下降：golden URL 曾被發現、後來被刪 → 變成 false。coverage 下降不一定是 crawler 變差，可能是保留策略刪掉了它。
    3. 反過來，這讓 coverage 可以**評估保留策略**：golden set ≈「重要網頁」的代表，若修剪後 golden URL 的 discovered 明顯下降，表示保留演算法刪錯了。
    4. 建議記錄修剪量（例如 crawler_stat 加 `pruned` 欄位，或 crawler 端記 log），才能把「刪除」和「真的沒發現」分開。
- 其他檢查：173 天中 A + B = Total 全部成立（status 管線沒有 shard -1 的問題，和 coverage 不同）。

### 6.6 小練習
1. 用一句 SQL（對單一 shard），同時算出：總 URL 數、曾成功抓過的數量、最近 7 天成功抓過的數量、`should_crawl = false` 的數量。
2. 寫 SQL：在 `crawler_stat_total` 中找出 discovered 比前一天少的日期。（提示：`LAG()` 視窗函數）
3. 如果某天 shard 042 查詢逾時，Overview 圖會出現什麼形狀？怎麼和 05-24 的情況區分？

<details><summary>參考答案</summary>

```sql
-- 1
SELECT COUNT(*)                                                         AS discovered,
       COUNT(last_fetch_ok)                                             AS crawled,
       COUNT(*) FILTER (WHERE last_fetch_ok >= now() - interval '7 days') AS crawled_7d,
       COUNT(*) FILTER (WHERE NOT should_crawl)                          AS not_should_crawl
FROM url_state_current_042;

-- 2  LAG(欄位) = 取「前一列」的值（依 ORDER BY 排序）
SELECT stat_date, discovered, prev, discovered - prev AS diff
FROM (
  SELECT stat_date, discovered,
         LAG(discovered) OVER (ORDER BY stat_date) AS prev
  FROM crawler_stat_total
) t
WHERE discovered < prev;
```
3. 那一天（那個小時）discovered 和 crawled 會**同時**往下掉約 1/256（約 0.4%），下一次成功的執行又跳回來 → 呈現「V 字缺口」。05-24 是 discovered 掉、crawled 照樣上升，而且沒有回升 → 是資料真的被刪，不是查詢失敗。
</details>

---

## 第 7 課：`_get_daily_summary_stats` —— 1 / 7 / 30 天滾動統計

檔案：`Metric/Measure/CrawlerStatusMeasure.py`。基礎見附錄 A.3；這課補 SQL 進階寫法與 dump 驗證。
來源表：crawlerdb `summary_daily`（每天一列：`event_date`、`num_fetch_ok`、`num_fetch_fail`、`fail_reasons` jsonb…）。

### 7.1 Python 做法：一次撈 30 天，在記憶體分桶

```python
rows = session.query(SummaryDaily).filter(
    and_(SummaryDaily.event_date >= today - 29, SummaryDaily.event_date <= today)).all()
for row in rows:
    delta = (today - row.event_date).days
    if delta == 0: daily ...      # 當天
    if delta < 7:  weekly += ...  # 含今天共 7 天
    if delta < 30: monthly += ... # 含今天共 30 天
```

### 7.2 等價 SQL（含 JSONB 取值）

```sql
SELECT
  SUM(num_fetch_ok) FILTER (WHERE event_date = CURRENT_DATE)      AS fetch_ok,
  SUM(num_fetch_ok) FILTER (WHERE event_date >= CURRENT_DATE - 6) AS fetch_ok_7,
  SUM(num_fetch_ok)                                               AS fetch_ok_30,
  -- fail_reasons 是 jsonb：->> 取出文字，再轉 int；key 不存在時是 NULL，SUM 會略過
  SUM((fail_reasons->>'HttpError 404')::int) FILTER (WHERE event_date >= CURRENT_DATE - 6) AS http_error_404_7,
  -- 成功率：NULLIF 避免除以 0
  SUM(num_fetch_ok) FILTER (WHERE event_date = CURRENT_DATE)::float
    / NULLIF(SUM(num_fetch_ok + num_fetch_fail) FILTER (WHERE event_date = CURRENT_DATE), 0) AS request_success_rate
FROM summary_daily
WHERE event_date BETWEEN CURRENT_DATE - 29 AND CURRENT_DATE;
```
- `->` 取出 jsonb、`->>` 取出 text。`fail_reasons->>'HttpError 404'` = Python 的 `reasons.get("HttpError 404")`。

### 7.3 進階：用視窗函數一次算出「每一天」的滾動值

目前程式每次只算「今天」，歷史值靠每天存起來。若想直接從 summary_daily 重算整段歷史（對應待處理 #11 的可回溯）：
```sql
SELECT event_date,
       num_fetch_ok AS daily,
       SUM(num_fetch_ok) OVER w7  AS weekly,
       SUM(num_fetch_ok) OVER w30 AS monthly
FROM summary_daily
WINDOW w7  AS (ORDER BY event_date RANGE BETWEEN INTERVAL '6 days'  PRECEDING AND CURRENT ROW),
       w30 AS (ORDER BY event_date RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW)
ORDER BY event_date;
```
- **`RANGE` vs `ROWS`**：`ROWS BETWEEN 6 PRECEDING` = 前 6「列」；若中間缺日期（crawler 停機那天沒有列），就會往前多抓、窗口超過 7 天。`RANGE ... INTERVAL '6 days'` 是依「日期值」算，缺日期也正確。

### 7.4 用 dump 驗證

**(a) daily 系統性少約 4%**
- 對 95 個「前 7 天都有資料」的日子，比較 `fetch_ok_7` 和「存下來的 7 個 daily 加總」：weekly 比較大，差距中位數 **3.5%**（0 ~ 4.7%）。
- 原因：每小時 UPSERT，當天最後一次寫入是 23:00 → 存下來的 daily 少了 23:00~24:00 那一小時（1/24 ≈ 4.2%）；而 weekly 是當下從 summary_daily 重算，前幾天都是完整的。→ **Dashboard 的 Crawled Daily 每天都被低估約 4%**（併入待處理 #7）。

**(b) 2026-06-04 ~ 08-07：crawler 停機，但 measure 還在跑**

| 日期 | discovered | fetch_ok | fetch_ok_7 |
|------|-----------|----------|-----------|
| 06-04 | 5,056,751,243 | 29,367,333 | 193M |
| 06-05 | 5,056,751,243（不變） | 963,809 | 166M |
| 06-06 | 5,056,751,243（不變） | **NULL** | 136M ↓ |
| … 08-07 | 5,056,751,243（不變） | NULL | … |

- discovered / crawled 63 天完全不變 → url_state 沒有變化 = crawler 沒在跑。
- fetch_ok 是 NULL：summary_daily 沒有「今天」那一列 → `delta == 0` 分支沒執行 → dict 裡沒有 `fetch_ok` 這個 key → UPSERT 沒提供 → NULL。
- 但 `fetch_total` 卻是 **0** 不是 NULL：因為 `day_total = stats_data["fetch_total"]` 這行讀取 `defaultdict(int)` 時**自動建立了 key = 0**。→ 同一列裡「沒資料」有的寫 NULL、有的寫 0，語意不一致（`defaultdict` 讀取就會新增 key 的副作用）。
- weekly / monthly 慢慢下降到 0（窗口裡的舊資料逐日滑出），圖上看起來像「爬得越來越少」，其實是停機。
- 之後 08-07 ~ 09-17 完全沒有列（measure 也停了），09-24 ~ 09-29 又缺 4 天。

**(c) 其他**
- `request_success_rate` 在 02-27 ~ 05-27 共 90 列是 NULL（欄位後來才加，git `feat/crawler-request-success-rate`）。
- A / B 表的 fetch 欄位全寫 **0**、成功率寫 NULL：summary_daily 是全域資料，無法拆隊。
  但 crawlerdb 有 **`domain_stats_daily`**（每個網域每天一列，**有 `shard_id`**、`num_fetch_ok`、`fail_reasons`）→ 其實可以算出 A / B 的 flow：
  ```sql
  SELECT CASE WHEN shard_id < 128 THEN 'A' ELSE 'B' END AS team,
         SUM(num_fetch_ok) FILTER (WHERE event_date = CURRENT_DATE)      AS fetch_ok,
         SUM(num_fetch_ok) FILTER (WHERE event_date >= CURRENT_DATE - 6) AS fetch_ok_7
  FROM domain_stats_daily
  WHERE event_date >= CURRENT_DATE - 29
  GROUP BY 1;
  ```
  （需確認 domain_stats_daily 加總是否等於 summary_daily。）

### 7.5 小練習
1. 寫 SQL：從 summary_daily 算出「最近 7 天每天的成功率」，並標出成功率低於 80% 的日子。
2. 為什麼 7.3 要用 `RANGE` 而不是 `ROWS`？舉 06-06 ~ 08-07 的例子說明。
3. 在 `crawler_stat_total` 中，如何用 SQL 找出「crawler 停機」的日子？（提示：discovered 和前一天完全相同，或 fetch_ok IS NULL）

<details><summary>參考答案</summary>

```sql
-- 1
SELECT event_date,
       num_fetch_ok::float / NULLIF(num_fetch_ok + num_fetch_fail, 0) AS success_rate,
       num_fetch_ok::float / NULLIF(num_fetch_ok + num_fetch_fail, 0) < 0.8 AS low
FROM summary_daily
WHERE event_date >= CURRENT_DATE - 6
ORDER BY event_date;

-- 3
SELECT stat_date
FROM (SELECT stat_date, discovered, fetch_ok,
             LAG(discovered) OVER (ORDER BY stat_date) AS prev_disc
      FROM crawler_stat_total) t
WHERE discovered = prev_disc OR fetch_ok IS NULL
ORDER BY stat_date;
```
2. 06-06 ~ 08-07 summary_daily 沒有列。若用 `ROWS 6 PRECEDING`，08-08 的「7 天」會往前抓到 06-05 以前的 6 列 → 把兩個月前的資料算成「本週」。`RANGE INTERVAL '6 days'` 只看日期在 7 天內的列 → 正確得到 0。
</details>

---

## 第 8 課：UPSERT —— `INSERT ... ON CONFLICT (stat_date) DO UPDATE`

檔案：`Metric/Measure/CrawlerStatusMeasure.py:test()`（寫 `crawler_stat_*`）、`Metric/Measure/CrawlerAllMetricMeasure.py:test()`（寫 `metric_headset_* / metric_randomset_*`）。兩邊寫法一樣。

### 8.1 Python

```python
stmt = insert(ModelClass).values(row_data)          # sqlalchemy.dialects.postgresql.insert
stmt = stmt.on_conflict_do_update(
    index_elements=['stat_date'],                   # 用哪個唯一鍵判斷「已存在」
    set_=row_data)                                  # 已存在時要 SET 哪些欄位
session.execute(stmt)        # Total / A / B 三張表各一次
session.commit()             # 三張表在同一個 transaction
```

### 8.2 等價 SQL

```sql
INSERT INTO crawler_stat_total (stat_date, discovered, crawled, indexed, fetch_ok, ...)
VALUES ('2026-10-01', 6327505701, 1538967152, 0, 3526448, ...)
ON CONFLICT (stat_date) DO UPDATE
SET discovered = 6327505701, crawled = 1538967152, indexed = 0, fetch_ok = 3526448, ...;
```
- `ON CONFLICT (stat_date)` 需要 `stat_date` 上有 UNIQUE / PRIMARY KEY（ORM：`stat_date = Column(Date, primary_key=True)`）→ **一天只有一列**。
- 慣用寫法是 `SET discovered = EXCLUDED.discovered`：`EXCLUDED` = 「這次想插入但撞到的那一列」。SQLAlchemy 用 `stmt.excluded.discovered`。程式直接把值再寫一次，效果相同。
- 整句在 DB 內原子完成，不是「先 SELECT 再決定 INSERT 或 UPDATE」→ 兩個連線同時跑也不會插出兩列（對比第 5 課的 read-modify-write）。

### 8.3 一天之內發生什麼事：最後寫的贏

`status` 每小時整點跑（Dockerfile `0 * * * *`），`stat_date = datetime.now().date()`：

| 時間 | 動作 | 這一列的 fetch_ok |
|------|------|-------------------|
| 00:00 | 今天還沒有列 → **INSERT** | 今天第 1 小時的量 |
| 01:00 ~ 22:00 | 撞到 PK → **UPDATE**（整列蓋掉） | 越來越大 |
| 23:00 | 最後一次 UPDATE | **最終值**（少最後 1 小時 → 第 7 課 7.4(a) 的 4%） |

**dump 驗證**：最新一列 `2026-10-01` 的 `fetch_ok = 3,526,448`，前幾天都是 2,000~2,800 萬 → 只有正常的約 13%。不是 crawler 變慢，而是 dump 在 10-01 凌晨（約 03:00）拿的，這一列**還沒寫完**。
→ 讀 dashboard 時，**最右邊（今天）那個點的 flow 值一定偏低**，不能拿來和前幾天比。（你只看 10 月起的資料，所以 10-01 這一列正好落在範圍內。）

### 8.4 坑：`set_=row_data` 只更新「有給的 key」

- INSERT 時：`row_data` 沒有的欄位 → NULL（或 DB default）。
- UPDATE 時：`row_data` 沒有的欄位 → **不在 SET 裡，保留舊值**。
- 所以同一個「key 不存在」在 INSERT 和 UPDATE 下結果不一樣。結合第 7 課 7.4(b)：crawler 停機時 `fetch_ok` 等 key 不存在 → 00:00 那次 INSERT 寫 NULL；如果某天早上有資料、之後才沒有，UPDATE 就會**保留早上的舊值**，看不出來後面停了。
- 用 `EXCLUDED` 也一樣要注意：沒放進 INSERT 欄位清單的欄位，`EXCLUDED.x` 是 default（NULL）。最清楚的做法是**每次都給齊所有欄位**，沒資料就明確給 None。

### 8.5 進階：有條件的 UPSERT（`DO UPDATE ... WHERE`）

「最後寫的贏」有兩種情況會出錯：
1. 23:00 那次有 shard 失敗（第 6 課 6.4）→ 用偏小的值蓋掉 22:00 的正確值。
2. 分散式後多台主機都跑每小時的 `status` → 較慢的那台**較舊的快照**比較晚寫入 → 蓋掉較新的值。

PostgreSQL 可以在 DO UPDATE 後面加條件，條件不成立就**什麼都不做**：
```sql
INSERT INTO crawler_stat_total AS t (stat_date, fetch_ok, ...)
VALUES (...)
ON CONFLICT (stat_date) DO UPDATE
SET fetch_ok = EXCLUDED.fetch_ok, ...
WHERE EXCLUDED.fetch_ok >= t.fetch_ok;   -- 當天的 fetch_ok 只會增加，變小 = 這次量錯或較舊
```
- 對 `fetch_ok` 有效（一天內只增不減）；對 `discovered` **不能這樣用**，因為修剪會讓它合理地下降（memory：crawler 1TB 修剪）。
- 更通用的做法：加 `measured_at timestamptz` 欄位，`WHERE EXCLUDED.measured_at > t.measured_at`（只讓較新的快照覆蓋）。

### 8.6 冷知識：從 dump 的列順序看出「被 UPDATE 過」（推測）

PostgreSQL 的 UPDATE 不是原地修改，而是**寫一個新版本的列**（MVCC），舊版本之後才被清掉。所以 dump（照實體位置輸出）裡，被 UPDATE 過的列常會排到後面。
`metric_headset_total` 在 dump 的順序是：02-27、03-01、03-16、**05-12、04-01、04-16、05-03**、05-23、05-30…
→ 04-01、04-16、05-03 排在 05-12 後面，而這 3 列正好是 `indexed_num = 0 但 indexed_rate = 0.16~0.18` 的那幾列（待處理「indexed 指標不可靠」）。這和「它們在 05-12 之後、05-23 之前被手動補過值」一致。只是推測：PG 也可能把新列放到空出來的位置，順序不能當證據，只能當線索。

### 8.7 小練習
1. 用 `EXCLUDED` 改寫：把一列寫入 `metric_headset_total`（欄位 `stat_date, total, discovered_num, discovered_rate`），已存在就更新。
2. 某天 dashboard 的 Crawled Daily 最後一個點突然掉到平常的 1/8，你會先檢查什麼，才說 crawler 出問題？
3. 8.5 的 `WHERE EXCLUDED.fetch_ok >= t.fetch_ok`：如果 23:00 那次的 shard 失敗只影響 `discovered`，這個條件擋得住嗎？要怎麼改？

<details><summary>參考答案</summary>

```sql
-- 1
INSERT INTO metric_headset_total (stat_date, total, discovered_num, discovered_rate)
VALUES ('2026-10-04', 3724, 2008, 2008.0 / 3724)
ON CONFLICT (stat_date) DO UPDATE
SET total = EXCLUDED.total,
    discovered_num = EXCLUDED.discovered_num,
    discovered_rate = EXCLUDED.discovered_rate;
```
2. 先看那個點是不是**今天**（還沒過完，8.3）；再看是不是 23:00 之後才拍的資料；最後才看 fetch_total、discovered 有沒有一起變（停機 / shard 失敗）。
3. 擋不住：`fetch_ok` 來自 summary_daily，不受 shard 失敗影響，條件成立 → 偏小的 `discovered` 照樣寫進去。要擋 shard 失敗，應該在 Python 端記錄 `failed_shards`，`failed_shards > 0` 時不寫入 snapshot 欄位（或在 SQL 加 `WHERE EXCLUDED.failed_shards = 0`）。
</details>

---

## 附錄 A：Superset 圖表怎麼算（學生提前問）

圖表定義在 `superset/bootstrap.py`：每個圖 = 一個 virtual dataset（SQL）+ 幾條折線 metric。X 軸 = `stat_date`。

### A.1 三層對照

| 圖 | dataset SQL（bootstrap.py） | 欄位來源 | 寫入者 |
|----|---------------------------|---------|--------|
| **Total Overview Volumn** | `SELECT stat_date, discovered, crawled, indexed FROM crawler_stat_total` | 每個 shard 的 `url_state_current_###` **快照** | `CrawlerStatusMeasure._scan_shard` |
| **Crawled (Daily/Weekly/Monthly)** | `SELECT stat_date, fetch_ok AS crawled_daily, fetch_ok_7 AS crawled_weekly, fetch_ok_30 AS crawled_monthly FROM crawler_stat_total` | crawlerdb `summary_daily` **流量** | `CrawlerStatusMeasure._get_daily_summary_stats` |

chart metric 用 `MAX(...)`：因為 `stat_date` 是 PK，每天只有 1 列，MAX 只是 Superset 要求必須有聚合函式，值不變。

### A.2 Total Overview Volumn（存量 / snapshot）

對 256 個 shard 各跑一次（16 threads 平行），加總：
```sql
SELECT COUNT(*)                                       AS discovered,  -- 表裡有幾個 URL
       COUNT(*) FILTER (WHERE last_fetch_ok IS NOT NULL) AS crawled   -- 曾經成功抓過的 URL 數
FROM url_state_current_###;
```
- `discovered` = crawler 目前知道的 distinct URL 總數（存量，只會一直變大）。
- `crawled` = 其中**至少成功抓過一次**的 distinct URL 數。
- `indexed` = 寫死 0（`_scan_shard` 回傳 0；selectdb 只用在 coverage，沒用在這裡）。
- Team A/B 表：shard 0-127 加到 A、128-255 加到 B。

### A.3 Crawled Daily/Weekly/Monthly（流量 / flow）

一次撈 `summary_daily` 最近 30 天（`event_date` 在 [today-29, today]），Python 迴圈依 `delta = today - event_date` 分桶：
- daily = `delta == 0` 那天的 `num_fetch_ok`
- weekly = `delta < 7` 的 `num_fetch_ok` 加總（含今天共 7 天）
- monthly = `delta < 30` 的加總

等價 SQL：
```sql
SELECT
  SUM(num_fetch_ok) FILTER (WHERE event_date =  CURRENT_DATE)      AS crawled_daily,
  SUM(num_fetch_ok) FILTER (WHERE event_date >= CURRENT_DATE - 6)  AS crawled_weekly,
  SUM(num_fetch_ok)                                                AS crawled_monthly
FROM summary_daily
WHERE event_date BETWEEN CURRENT_DATE - 29 AND CURRENT_DATE;
```

### A.4 兩張圖的 "crawled" 單位不同 ⚠️
- Overview 的 `crawled` = **distinct URL 數**（某 URL 抓 100 次也只算 1）。
- Daily/Weekly/Monthly = **成功 fetch 的次數**（重抓同一 URL 每次都算）。
- 所以不能直接相減比較。例：2026-04-02 `crawled` 50.2M，`fetch_ok_30` 39.5M。

### A.5 讀圖要注意
- status cron **每小時**跑一次（Dockerfile `0 * * * *`），用 UPSERT 覆蓋當天那一列 → 今天的 daily 是「到目前為止」的部分值，最後一次是 23:00 的 run，所以 23:00~24:00 那段不會被算進當天。
- 滾動窗是 `summary_daily` 的加總，跟 crawler_stat 有沒有寫入無關。dump 裡 2026-09-25~28 沒有列（measure 沒跑），09-29 的 weekly 從 ~207M 掉到 77M → 那幾天 `summary_daily` 的 fetch 量本身很低（可能 crawler 停了，待確認）。
- `request_success_rate` 舊列是 NULL（欄位是後來加的）；dashboard 的 Effectiveness 圖是用 `fetch_ok / NULLIF(fetch_total,0)` 現算，所以舊資料也有值。

### A.6 Q&A（學生追問）

**Q：SerpApi 的邏輯？metric_url 的前 10 名，Google 是怎麼挑的？**
- SerpApi **不是 Google 官方 API**，是第三方付費服務：它代替你去 Google 搜尋，把結果頁（SERP）解析成 JSON 回傳（計次收費，`Metric/getQuota.py` 用來查剩餘額度）。
- 前 10 名是 Google 搜尋排名演算法決定的（黑盒：相關性、連結權重、新鮮度、地區、語言…）。我們沒有挑，只照抄。
- 程式只拿 `organic_results`（一般藍色連結），廣告、Top stories、影片框、知識面板都不算 → 這就是很多 query 不到 10 筆的原因。
- `getQuery` 只傳 `q`、`num=10`，**沒有傳 `gl`（國家）/`hl`（語言）/`location`** → 全部用 SerpApi 預設地區搜尋。來自 JP/TW/BR 的 trending 詞，拿到的可能不是當地使用者會看到的結果。（設計上的取捨，可以提出來討論）

**Q：crawled daily 會重複算同一個 URL，不就違反「有多少 URL 至少被抓過一次」的定義？**
- 那個定義是 **Overview 的 `crawled`**（distinct URL）。daily/weekly/monthly 的原始欄位叫 `fetch_ok`（成功抓取次數）= 吞吐量 / 工作量。是 dashboard 把它改名成 `crawled_daily` 才容易誤會 → 命名問題，不是算錯。
- 如果真的想要「過去 7 天內有被成功抓過的 distinct URL 數」，可以直接從 url_state 算（只要 7 天內抓過，最後一次成功時間一定在 7 天內）：
  ```sql
  SELECT COUNT(*) FILTER (WHERE last_fetch_ok >= now() - interval '1 day')   AS urls_daily,
         COUNT(*) FILTER (WHERE last_fetch_ok >= now() - interval '7 days')  AS urls_weekly,
         COUNT(*) FILTER (WHERE last_fetch_ok >= now() - interval '30 days') AS urls_monthly
  FROM url_state_current_###;   -- 256 個 shard 加總
  ```
  限制：只能算「到現在為止」，無法回推歷史上某一天的值（url_state 只存最新狀態）→ 必須每天跑、存起來。

**Q：weekly crawled 是不是先找一個月內的 batch，再去 metric_url 算？**
- **不是。兩條管線完全獨立：**

| | crawler_stat_*（Overview、Daily/Weekly/Monthly） | metric_headset_* / randomset_*（Coverage） |
|---|---|---|
| 問的問題 | 整個 crawler 規模多大、抓多快？ | Google 熱門搜尋結果，我們爬到幾 %？ |
| 範圍 | crawlerdb **全部** URL（數十億） | 只有 golden set（**最新 1 個 batch**，約 7 千個 URL） |
| 讀的表 | `url_state_current_###`、`summary_daily` | `metric_url` ⋈ `metric_queries`，再去 url_state / domain_state / selectdb 查狀態 |
| 程式 | `CrawlerStatusMeasure` | `CrawlerAllMetricMeasure` |
| 用到 batch？ | ❌ | ✅（只用最新的） |

- Coverage 的 `crawled_rate` 也是快照：量測當下，golden URL 在 url_state 的 `last_fetch_ok IS NOT NULL` 的比例，沒有 7 天 / 30 天的版本。

---

## 附錄 B：metric_url 是哪些網站？（學生要求的統計，2026-10-04）

做法：dump → `pg_restore -a` → Python 解析 `url` 的 hostname，用正則規則把網域歸類（啟發式，非精確）。全部 batch 共 140,840 列、23,760 個不同 host。

### B.1 類別分佈

| 類別 | 列數 | 佔比 | 代表網域 |
|------|------|------|---------|
| 其他（官網/品牌/部落格/工具…） | 47,340 | 33.6% | finance.yahoo、weather.com、statmuse、ticketmaster… |
| 體育 | 23,603 | 16.8% | espn、sofascore、flashscore、cricbuzz、nba/mlb/nhl |
| 社群/論壇 | 19,922 | 14.1% | instagram 8,263、facebook 5,051、reddit 2,751、x.com 2,721 |
| 百科/參考 | 18,339 | 13.0% | en.wikipedia 8,899、ja/zh.wikipedia、imdb、britannica |
| 影音 | 11,588 | 8.2% | youtube 9,808（第一名的單一網域）、spotify、tiktok |
| 新聞/媒體 | 11,221 | 8.0% | bbc、nytimes、guardian、news.yahoo.co.jp |
| 政府/教育 | 4,920 | 3.5% | .gov / .edu / .go.jp |
| 電商/購物 | 2,586 | 1.8% | amazon、play.google |
| 圖片庫 | 808 | 0.6% | gettyimages |
| 旅遊/地圖 | 513 | 0.4% | tripadvisor、google maps |

- rank 1 最常見：en.wikipedia（4,255 次）、espn、youtube、instagram、ja.wikipedia。
- TLD：.com 68%、.org 12%、.jp 5.8%。
- 意涵：trending 詞多是體育賽事 / 名人 / 影視 → golden set 偏重 **大型平台**（youtube、IG、FB、X 常擋 crawler 或需要登入）→ 會壓低 crawled coverage，解讀 coverage 時要記得。

### B.2 真的沒有廣告 / 知識面板嗎？ → 大致沒有，但有約 316 列「漏網」

| 類型 | 列數 | 例子 | 狀態 |
|------|------|------|------|
| **相對路徑 `/goto?url=CAES...`**（Google 內部跳轉連結，不是真網址） | **179** | query "cole young" | **全在 batch 9**；全部 is_discovered=f、shard_id=-1 |
| Google 自家頁面 | 137 | `google.com/maps/search/...`（28）、`support.google.com/...`（18）、`g.co/kgs/...`（1，知識圖譜短網址）、`google.com/search` | 分散在各 batch |
| 帶追蹤參數 `utm_source=` | 20 | open.spotify.com、discuss.com.hk | 是真實頁面，只是 URL 帶參數 |
| 廣告網址（googleadservices / doubleclick / aclk / gclid） | **0** | — | ✅ |

- 廣告：0 筆 → `organic_results` 確實有排除廣告。
- 知識面板：本身不會進來，但 `g.co/kgs` 這種知識圖譜連結、Maps 連結偶爾會以 organic 形式出現。
- `/goto` 那 179 筆是 **資料品質問題**：不是合法 URL，crawler 永遠找不到 → 永久拉低 batch 9 的 discovered/crawled rate。可能是當時 SerpApi / Google 結果格式改變（推測，未確認）。
- 建議修正（`QueryStrategy.getQuery`）：只收 `link.startswith("http")`，並可排除 `google.*` host。
- 也可以用 SQL 直接查：
  ```sql
  SELECT mq.batch_id, COUNT(*)
  FROM metric_url mu JOIN metric_queries mq ON mq.id = mu.query_id
  WHERE mu.url NOT LIKE 'http%'
     OR mu.url ~ '^https?://(www\.)?google\.[a-z.]+/'
     OR mu.url LIKE '%g.co/kgs%'
  GROUP BY mq.batch_id ORDER BY 1;
  ```

---

## 附錄 C：coverage 怎麼從 metric_url 算出來、怎麼去重（學生提前問，第 9 課核心）

程式：`Metric/Measure/CrawlerAllMetricMeasure.py:test()`。每跑一次 = 一個 (最新 batch, tag) → 寫 3 列（Total/A/B 各一列，stat_date = 今天）。

### C.1 五個步驟

| 步驟 | 做什麼 | 等價 SQL / 資料結構 |
|------|--------|---------------------|
| 1. 撈 golden URL | 最新 batch 中 tags 含該 tag 的 query 底下所有 URL 列 | `SELECT mu.* FROM metric_url mu JOIN metric_queries mq ON mq.id=mu.query_id WHERE mq.batch_id=:b AND mq.tags @> '["head"]'` |
| 2. **去重** | 用 dict 把相同 url 字串收成一組 | `url_id_map = {url: [metric_url.id, ...]}` ≈ `GROUP BY url` + `array_agg(id)` |
| 3. 查狀態（每個 distinct URL 查一次） | ① crawlerdb 256 個 `url_state_current_###`：`WHERE url IN (...)` → 有找到 = discovered；`last_fetch_ok IS NOT NULL` = crawled；找到的 shard 編號 = shard_id ② 找不到的話用 `domain_state` 依網域補 shard_id（只決定 A/B，discovered 仍是 False）③ selectdb `selected_urls_current WHERE url = ANY(:urls)` → indexed | 三個不同 DB，無法一句 SQL JOIN，所以在 Python 合併 |
| 4. 統計 | 以 **distinct URL** 為單位累加；shard 0-127 → A，128-255 → B，-1 → 只算 Total | 見 C.2 |
| 5. 寫回 | ① `bulk_update_mappings`：把狀態寫回**同組所有** metric_url.id ② UPSERT 到 `metric_{tag}set_{total,a,b}` | `UPDATE metric_url SET is_discovered=..., shard_id=... WHERE id=...` + `INSERT ... ON CONFLICT (stat_date) DO UPDATE` |

### C.2 等價 SQL（假設三個 DB 在同一個地方，方便理解）

```sql
WITH golden AS (                        -- 步驟 1+2：去重
  SELECT DISTINCT mu.url
  FROM metric_url mu JOIN metric_queries mq ON mq.id = mu.query_id
  WHERE mq.batch_id = :batch_id AND mq.tags @> '["head"]'
),
status AS (                             -- 步驟 3：查狀態
  SELECT g.url,
         us.url IS NOT NULL                    AS discovered,
         us.last_fetch_ok IS NOT NULL          AS crawled,
         su.url IS NOT NULL                    AS indexed,
         COALESCE(us.shard_id, ds.shard_id, -1) AS shard_id
  FROM golden g
  LEFT JOIN url_state_all us        ON us.url = g.url         -- 256 個 shard 的聯集
  LEFT JOIN domain_state ds         ON ds.domain = domain_of(g.url)
  LEFT JOIN selected_urls_current su ON su.url = g.url
)
SELECT CASE WHEN shard_id BETWEEN 0 AND 127 THEN 'A'
            WHEN shard_id BETWEEN 128 AND 255 THEN 'B' END AS team,
       COUNT(*)                                   AS total,
       COUNT(*) FILTER (WHERE discovered)         AS discovered_num,
       COUNT(*) FILTER (WHERE crawled)            AS crawled_num,
       COUNT(*) FILTER (WHERE indexed)            AS indexed_num,
       COUNT(*) FILTER (WHERE crawled)::float / COUNT(*) AS crawled_rate
FROM status
GROUP BY ROLLUP (team);   -- ROLLUP 多一列 NULL = Total（含 shard -1 的 URL）
```

### C.3 用 dump 驗證（batch 2 head，stat_date 2026-03-16）—— 完全對上

| | metric_url 列數 | distinct URL = total | discovered | crawled |
|--|--|--|--|--|
| Total | 7,628 | **7,057** | 1,708 (24.2%) | 390 (5.5%) |
| A | | 3,517 | 883 | 172 |
| B | | 3,349 | 825 | 218 |
| shard -1（只在 Total） | | 191 | 0 | 0 |

- 3,517 + 3,349 + 191 = 7,057 → **A + B ≠ Total**，差的是找不到 shard 的 URL。
- 從 metric_url 的 is_* 欄位以 distinct url 重算，數字與 metric_headset_* 完全一致。

### C.4 去重的細節與坑
1. 去重單位是「**完全相同的 URL 字串**」。`http://` vs `https://`、有沒有 `www.`、結尾 `/`、`utm_` 參數都算不同 URL（不做 normalize）。crawler 的 url_state 如果存的是 normalize 過的網址，就會比對不到 → discovered 被低估。
   - 白話（學生問「看不懂」後的解釋）：比對是**逐字比對字串**，同一個網頁可能有好幾種寫法（`https://ligamx.net/` vs `https://www.ligamx.net/`）。Google 給的寫法 ≠ crawler 存的寫法 → `WHERE url IN (...)` 找不到 → 判為 discovered=False，即使 crawler 其實已經有這頁。
   - dump 證據：同一頁不同寫法共 47 組，其中 18 組 discovered 一 t 一 f，而且 shard_id 相同（同網域）：
     - `instagram.com/banksy/` t vs `instagram.com/Banksy/` f（大小寫）
     - `vancouverattractions.com/` t vs `www.vancouverattractions.com/` f（www）
     - `mdr.de/in-aller-freundschaft` t vs `mdr.de/In-aller-freundschaft` f（大小寫）
     - 差異類型：大小寫 19、www 11、結尾 / 7、http/https 5、#/utm 5
   - 能看到的只是冰山一角：golden set 只出現一種寫法時，我們看不到 crawler 存的是哪種 → 低估的總量未知。
   - 範圍確認（學生問）：低估**只發生在 coverage 管線**拿 golden URL 去比對的時候：① 比 crawlerdb `url_state`（→ discovered/crawled）② 比 selectdb `selected_urls_current`（→ indexed）。domain fallback 用 tldextract 取網域，不受影響（但它只決定 A/B）。crawler_stat 的 Overview 只是 `COUNT(*)` 不做字串比對，不會低估；反而若 crawler 自己沒 normalize，同頁多種寫法會被算成多個 URL（高估）。golden set 這邊，同頁兩種寫法也會被算成 2 個 distinct URL（分母多算）。
   - 改善：比對前兩邊用同一個 normalize 函式（去 www、統一 https、去結尾 /、去 utm/#），最好用 crawler 本身的 normalize 邏輯。
2. 只在**同一 batch + 同一 tag** 內去重。head 和 random 各算各的，同一 URL 會同時算進兩邊。
3. 加權：每個 distinct URL 權重 1，與 rank、被幾個 query 搜到、query frequency 無關。
4. 量測頻率：cron 只在 `--create` 那次一起跑（每月 1、16 號），所以 headset 只有 11 列、randomset 13 列 → **每個點 ≈ 一個新 batch**，相鄰點的 URL 幾乎完全不同。
5. metric_url 的 is_* 是「最後一次量測時」的快照；舊 batch 不會再被更新。
6. indexed 依賴 selectdb；沒傳 `--select_db_url` 時全部是 False → indexed_rate = 0。（2026-04-01 headset_total 有 indexed_num=0 但 indexed_rate=0.184 的矛盾列，疑似手動修改，待查）

---

## 附錄 D：shard 是什麼？

- **shard（分片）= 把一張超大表切成很多張小表**。crawlerdb 的 URL 有幾十億筆，放一張表太大，所以切成 256 張：`url_state_current_000` ~ `url_state_current_255`（`AppModelFactory.create_url_state_current_model(idx)` 動態產生這些表的 ORM class）。
- 切法：**以網域為單位**分配。`domain_state` 表記錄每個網域被分到哪個 shard（`domain_state.shard_id`），同網域的 URL 都在同一個 shard。（實際分配規則在 crawler 端，本 repo 看不到，可能是 hash）
- **Team A / B**：shard 0-127 歸 Team A、128-255 歸 Team B → 兩組各負責一半網域的 crawler（`docs/04-database-schema.md` §5）。所以 dashboard 的 A/B 是在比較兩隊的成果。
- 在 metric 裡的用途：
  - `CrawlerStatusMeasure`：256 張表各 `COUNT(*)`，依 shard 編號加到 A 或 B。
  - `CrawlerAllMetricMeasure`：golden URL 要去 256 張表都找一次（不知道在哪張）；找到的表號 = `metric_url.shard_id`；找不到就用 `domain_state` 依網域補；兩者都找不到 = -1（只算 Total）。
- 為什麼 A + B ≠ Total：shard_id = -1 的 URL 不屬於任何一隊。

### C.5 Q&A：重複的 query / URL 有必要收集嗎？去掉能提升 coverage 嗎？（學生問，2026-10-04）
- **有必要存**：metric_url 一列 = 「某 query 的第 N 名是這個 URL」。去掉重複列會失去 query→URL 對應與 rank，之後無法做「每個 query 前 10 名爬到幾個」或「rank 越前面越容易被爬到嗎」的分析。成本很低（batch 2 head 只多 571 列）。
- **去掉也不會提升 coverage**：coverage 本來就以 distinct URL 計算，刪掉重複列數字完全不變。
- 刻意刪 URL 讓比例變好 = 改分母做數字（gaming the metric），coverage 應該只因 crawler 變好而上升。
- 真正的設計選擇是「怎麼加權」，batch 2 head 三種算法（dump 實算）：

| 算法 | 分母 | discovered | crawled |
|------|------|-----------|---------|
| distinct URL（目前做法） | 7,057 | 24.2% | 5.5% |
| 每列都算（被越多 query 搜到的 URL 權重越大） | 7,628 | 25.1% | 5.9% |
| 以 query 為單位：前 10 名至少 1 個被爬到 | 1,000 | — | 30.2% |

  - 被多個 query 搜到的 408 個 URL，crawled 8.6%，高於整體 5.5% → 熱門大站較容易被爬到。
  - 要提升 coverage 的正道：讓 crawler 優先爬 golden set 的網域 / URL（seed）、修正附錄 C.4 的 normalize 問題、清掉 /goto 假網址（這兩項是**修正低估**，不算作弊）。

### C.6 Q&A：把 query / URL 的標記保留給下一個 batch 用？（學生問，2026-10-04）
- 相鄰 batch 的重疊（dump 實算，selected = tags 非空）：

| batch | 選中 keyword 重疊 | URL 重疊 | 重疊 URL 中：上一批已 crawled → 這批 crawled（新爬到） |
|------|------|------|------|
| 1→2 | 60 / 1,915 | 303 / 14,204 | 50 → 70（+20） |
| 4→5 | 80 / 2,557 | 533 / 19,631 | 107 → 193（+86） |
| 7→8 | 113 / 1,911 | 754 / 14,431 | 294 → 346（+52） |
| 8→9 | 91 / 1,920 | 223 / 7,139 | 93 → 93（+0） |

- 結論：
  1. 重疊只有約 3~6%（keyword）、2~5%（URL）→ 沿用上一批的 SerpApi 結果，省下的額度很少；而且 Google 結果會變，沿用會拿到舊的 SERP。
  2. crawler 狀態（is_crawled 等）**不能沿用**，一定要重新查：上表顯示重疊 URL 在兩批之間會有一部分從「沒爬到」變成「爬到」。
  3. 真正有價值的用途是「**追蹤**」：建一張跨 batch 的 URL 歷史表（url UNIQUE, first_seen_batch, first_crawled_at），可以算「golden URL 從出現到被爬到要多久」（time-to-crawl），也可以把一直沒爬到的 URL 回饋給 crawler 當 seed。

---

## 附錄 E：Rank 指標的演算法方向（學生問，2026-10-04）

背景：rank 是上一屆想做、但沒想出合適演算法的指標（`TypesenseRankMeasure` 為空 stub，原註解邏輯 = 拿 keyword 搜 Typesense 前 10 名，看 golden URL 有沒有出現）。

### E.1 核心原則：把 rank 放在漏斗最後一層，並且「以 indexed 為條件」
```
golden URL → discovered → crawled → indexed → ranked（我們的搜尋結果前 K 名有它）
```
- 若 `ranked_rate = ranked / total`，數字低可能只是因為沒爬到 / 沒 index，**分不出是排序爛還是覆蓋率低**。
- 建議同時報兩個：`ranked_rate = ranked / total`（端到端）、`ranked_given_indexed = ranked / indexed`（純排序能力）。

### E.2 候選演算法（由簡到難）

| 指標 | 單位 | 定義 | 優點 | 缺點 |
|------|------|------|------|------|
| **Recall@K**（上一屆的構想） | query | 我們前 K 名中命中幾個 golden URL ÷ golden URL 數 | 簡單、好解釋 | 不看名次；Google 第 1 和第 10 同權 |
| **Hit@K** | query | 前 K 名至少命中 1 個 golden URL 的 query 比例 | 貼近使用者感受（有沒有找到好結果） | 太粗 |
| **MRR** | query | 1 ÷ 第一個命中 golden URL 的名次，取平均 | 看「最快多久找到」 | 只看第一個命中 |
| **NDCG@10（Google 名次當相關度）** | query | 相關度 rel = 11 − google_rank（或 1/log2(google_rank+1)），對我們的前 10 名算 DCG ÷ IDCG | 名次越前的 golden URL 越重要；業界標準 | 把 Google 當絕對正解 |
| **RBO（Rank-Biased Overlap）** | query | 比較兩個排序清單在各深度的重疊，深度越淺權重越大（參數 p≈0.9） | 專門比較「兩個搜尋引擎的排序像不像」；能處理兩邊 URL 不完全相同 | 較難向非技術人員解釋 |

推薦組合：**Recall@10（給 dashboard，容易懂）+ NDCG@10 以 indexed 為條件（給分析，看排序品質）**。

### E.3「以 indexed 為條件」的 NDCG（把覆蓋率和排序拆開）
- 一般 IDCG = 假設 10 個 golden URL 全部排在最前面。
- 改成 **IDCG_indexed = 只拿「我們有 index 到的 golden URL」做理想排序**。
- `NDCG_cond = DCG / IDCG_indexed` → 1.0 代表「有 index 的都排得跟 Google 一樣好」，低分才是排序問題；index 不到的部分由 indexed_rate 負責。

### E.4 實作要點
1. 比對要先 normalize URL（待處理 #2），否則 ranked 會跟 discovered 一樣被低估；可另外算「網域層級命中」（同網域任一頁出現就算）作為寬鬆版。
2. Google 結果要跟我們搜尋同條件（待處理 #3 的 gl/hl）。
3. 量測時間要接近 SerpApi 抓取時間：trending 變很快，隔兩週再比沒意義。
4. head / random 分開報；也可依 geo 分開。
5. 資料表建議：
   - `metric_url` 加 `our_rank int`（我們搜尋結果中的名次，沒出現 = NULL）→ `is_ranked = our_rank <= K`。
   - 新表 `metric_query_rank(query_id, stat_date, recall_10, ndcg_10, mrr)` 存每個 query 的分數；`ranked_num / ranked_rate` 再由它聚合。
6. 有了 `our_rank` 之後，Recall@10 一句 SQL：
   ```sql
   SELECT mq.id, mq.keyword,
          COUNT(*) FILTER (WHERE mu.our_rank <= 10)::float / COUNT(*) AS recall_10
   FROM metric_url mu JOIN metric_queries mq ON mq.id = mu.query_id
   WHERE mq.batch_id = :b AND mq.tags @> '["head"]'
   GROUP BY mq.id, mq.keyword;
   ```

### E.5 限制（報告時要說明）
- Google 不是絕對正解：我們排得不一樣不一定是錯。RBO / NDCG 衡量的是「和 Google 的一致程度」。
- 我們的 index 遠小於 Google，端到端 ranked_rate 預期會很低 → 一定要搭配 E.1 的條件版本。

### D.1 Q&A（學生問，2026-10-04）

**Q：A / B 是 head / random 嗎？** → 不是，兩者是**正交的兩個維度**：
- A / B = **這個 URL 歸哪一隊的 crawler 負責**（shard 0-127 = A、128-255 = B）。
- head / random = **keyword 是怎麼挑的**（熱門前 N vs 隨機 N）。
- coverage 有 2 × 3 = 6 張表：`metric_{head,random}set_{total,a,b}`。例如 `metric_headset_a` = 「head 挑出來的 golden URL 中，屬於 A 隊網域的那些」的覆蓋率。crawler_stat 只有 A / B / Total，沒有 head / random（它不用 golden set）。

**Q：比較 A / B 成功率有什麼意義？**
- 若兩隊各自實作 / 調整 crawler（排程、重試、禮貌延遲、URL 挑選），成功率 = 「送出的請求有多少成功」，可比較**兩隊 crawler 的效率**（例如 A 隊 404 特別多 → 它挑了很多失效 URL）。
- 也能抓**只發生在一半 shard 的故障**（某隊的機器掛了，Total 只會小降，拆開才看得出來）。
- 限制：兩隊負責的網域不同。若 shard 是依網域 hash 分配，大數量下網域組成接近隨機、可比；若是依其他規則分配，差異可能來自網域本身（某些網站本來就常擋爬蟲），不是 crawler 好壞。
- 若兩隊跑同一套 crawler、沒有競爭關係，A / B 比較的價值主要剩「偵測局部故障」。

**Q：送 SerpApi 的 query 會重複嗎？**（dump 估算；不含重跑與失敗重試，實際更多）
| 情況 | 是否重複 | 數量 |
|------|---------|------|
| 同一次 strategy 執行內 | 不會（同 batch 的 keyword 已去重） | — |
| 同 batch、head 和 random 都選中 | **會**：兩次執行各搜一次，後跑的 DELETE + 重搜 | 760 次（約 3.8%） |
| 同 batch 重跑 | **會**：全部重搜 | 未知 |
| 不同 batch 出現同一 keyword | **會**（設計上合理：結果會隨時間變） | 1,879 次（約 9.4%）；例：`weather tomorrow`、`f1`、`alexander zverev` 各在 7 個 batch 被搜 |
- 同 batch 的 head ∩ random 重搜是純浪費：後跑的 strategy 若發現該 query 已有 URL，可直接加 tag、跳過搜尋。
