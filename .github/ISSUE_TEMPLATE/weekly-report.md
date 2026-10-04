---
name: Weekly Report
about: 每週進度報告
title: "Weekly Report — YYYY-MM-DD"
labels: weekly-report
---

## 摘要
<!-- 一兩句話說明本週重點 -->
調查現有 metric pipeline（`Metric/`、`measure.py`、metricdb、Superset dashboard）的正確性。用 metricdb dump 對照程式碼，共找到 14 個問題：會讓數字失真的 9 項、dashboard 判讀問題 2 項、分散式風險 1 項、資料保留 1 項、成本 1 項。程式碼都**尚未修改**，修正方向需要討論。

## 本週完成
- 讀懂 metricdb 三張核心表（`metric_batches` → `metric_queries` → `metric_url`），以及 `crawler_stat_*`、`metric_headset_*`、`metric_randomset_*` 的寫入流程
- 用 dump 驗證 coverage 的計算方式（batch 2 head 完全對得上）
- 整理 Superset 各張圖的計算來源
- 教程與詳細證據：`docs/tutorial/metricdb-sql-tutorial.md`（下面的「§」都指這份檔案的章節）

## 指標變化
| 指標 | 上週 | 本週 | 變化 |
|---|---|---|---|
| discovery coverage | | | |
| crawl coverage | | | |

## 實驗與發現
<!-- 做了什麼實驗、結果、學到什麼 -->

### A. 會讓 metric 數字失真
1. **coverage 比對 URL 沒有 normalize → 低估**（§附錄 C.4）
   golden URL 和 crawlerdb / selectdb 用逐字比對，所以 `www`、http/https、結尾 `/` 不同就比對不到。dump 中有 47 組同一頁的不同寫法，其中 18 組 discovered 一個 t、一個 f。
2. **metric_url 混入 `/goto` 假網址**（§附錄 B.2）
   batch 9 有 179 筆，永遠抓不到，所以會拉低 discovered / crawled rate。
3. **SerpApi 沒有指定 `gl` / `hl`**（§附錄 A.6）
   keyword 來自 10 個國家的 trending，卻都用 SerpApi 預設地區搜尋，所以 golden set 會偏向預設地區的網站。keyword 不需要翻譯，只要用原文加上該國的 `gl`/`hl` 搜尋，並記錄實際用了哪些搜尋條件。
4. **失敗時默默回傳 0 / 空值，被當成正常結果寫入**（§2.3、§2.7、§6.4）
   - SerpApi 失敗會產生空 batch（batch 10~17），以及「有 tag、但 0 個 URL」的 query（1,416 個）。
   - crawlerdb 的 shard 失敗會被當成 0，而且完全沒有 log，總數會少約 1/256。
5. **重跑同一個 batch 時，SerpApi 失敗會把原本的 URL 刪掉**（§2.7）
   流程是 DELETE，接著 `getQuery` 回傳 `[]`、插入 0 筆後 commit，所以原本的 URL 和 is_* 狀態會被永久刪除。
6. **重跑 random 時 keyword 越選越多**（§4.4）
   舊的 random tag 沒有移除。batch 1 有 1,313 個 random query，超過 N = 1000。
7. **indexed 指標不可靠**（§附錄 A.2、C.4）
   - `crawler_stat_*` 的 indexed 寫死為 0。
   - coverage 的 indexed 在 09-17 又變回 0。
   - 04-01 ~ 05-03 有 6 列是 `indexed_num = 0`，但 `indexed_rate` 卻有值。
8. **rank 存了但沒用到；ranked 指標寫死 0**（§附錄 E）
   dashboard 的 RankCov 線永遠是 0。演算法方向已整理在附錄 E（Recall@10，以及以 indexed 為條件的 NDCG@10）。
9. **crawler 停機時，NULL 和 0 混用**（§7.4(b)）
   2026-06-06 ~ 08-07 有 63 列 `fetch_ok = NULL`、`fetch_total = 0`，weekly / monthly 會逐日滑落，看起來像「爬得越來越少」。

### B. Dashboard 判讀問題
10. **「Crawled (Daily/Weekly/Monthly)」其實是 fetch 次數，不是 URL 數**（§附錄 A.4、§7.4、§8.3）
    - 和 Overview 的 `crawled`（distinct URL）單位不同，名稱卻一樣。
    - 存下來的 daily 是 23:00 那次的值，少了最後 1 小時，所以低估約 4%。
    - 今天那一點還沒寫完，一定偏低（dump 的 10-01 只有平常的約 13%）。
11. **A / B 隊的 fetch 欄位全寫 0**（§7.4(c)）
    其實可以用 crawlerdb 的 `domain_stats_daily`（有 `shard_id`）分隊計算；算不出來時應該寫 NULL，不是 0。

### C. 分散式 / 資料保留 / 成本
12. ⚠️ **【分散式必看】read-modify-write 會造成 lost update**（§5.2、§8.5）
    `meta_tag_stats`、`getGoldenSet` 的 tags，以及「先 SELECT 再 INSERT」，在多台主機同時執行時都會互相覆蓋或產生重複列。每小時的 `status` 如果多台都跑，較舊的快照可能蓋掉較新的。
13. **歷史 metric 無法重算**（§5.4）
    URL、is_* 欄位、coverage 都會被覆蓋或刪除。batch 9 random 在 06-12 量測時有 7,738 個 URL，現在只剩 7,139 個。
14. **同一個 batch 中 head ∩ random 的 keyword 被 SerpApi 搜了兩次**（§附錄 D.1）
    多搜了 760 次（約 3.8%），只影響成本，不影響正確性。

## 遇到的問題
<!-- 可以連結相關 issue，例如 #42 -->
- 本機沒有 postgres server，所有驗證都是從 dump 抽資料分析，無法直接查 crawlerdb / selectdb（例如 #1 實際低估多少、#7 的 09-17 為什麼是 0）。
- 部署的 cron 設定和 repo 的 `Dockerfile` 不一致（keywordNums 1000 vs 50、batch 週期曾經調整過），需要確認實際的部署設定。

## 下週計畫
- [ ] 和組員討論修正優先順序，建議先做 #12（分散式）、#4、#1
- [ ] 確認 10 月起的資料是否仍有 #4（空 batch / 0 URL）
- [ ] 查 09-17 coverage 的 indexed = 0 的原因（#7）
- [ ] 確認 crawler 存 URL 時怎麼 normalize，作為 #1 的修正依據
- [ ] 確認 `domain_stats_daily` 加總是否等於 `summary_daily`（#11）
