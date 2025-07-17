# BigQuery DDL 草稿

說明：本文件提供以 GCP BigQuery 為主的三層式資料模型（Bronze／Silver／Gold）DDL 草稿，對應你要從 Shopline Open API 抽取之資料域，再加上 AI 客服對話與每日分類整併｡各段落皆可直接複製調整成基礎 infra；欄位型別、分區鍵、雜湊欄位請依實際數據量與下游需求微調｡

---

## 命名約定與資料集規劃

| 層級             | Dataset 建議      | 命名規則                                   | 分區                                 | 備註              |
| -------------- | --------------- | -------------------------------------- | ---------------------------------- | --------------- |
| Bronze（Raw）    | `shopline_raw`  | `raw_<domain>_dt=` ingestion partition | DATE 分區（\_ingestion\_date）         | 保留原始 JSON；利於重算｡ |
| Silver（Core）   | `shopline_core` | 實體化維度／事實表                              | DATE/TIMESTAMP 分區（變更時間）＋Clustering | 正規化、型別轉換、去重｡    |
| Gold（Mart／CDP） | `shopline_mart` | 分析寬表或彙總                                | 依業務邏輯                              | 面向 Looker／Apps｡ |

時間全部以 UTC 儲存；報表層再轉 Asia/Taipei（GMT+8）｡

---

## Bronze 層（Raw Landing）

Bronze 直接落地 API 回傳原封 JSON（NDJSON），以「每日抽」或「增量抽」檔案匯入；每列一筆 API record，並保留擷取時間與原始回應字串，方便審計及重新解析｡

### raw\_customers

來源：`GET /v1/customers`；支援 `updated_after`、`updated_before`、`per_page`、`previous_id` 等參數以利增量大批次抓取｡ fileciteturn15file11L19-L25

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_raw.raw_customers` (
  _ingestion_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  _ingestion_date DATE NOT NULL DEFAULT CURRENT_DATE(),
  response_json STRING OPTIONS(description="完整原始 API JSON")
)
PARTITION BY _ingestion_date;
```

---

### raw\_orders

來源：`GET /v1/orders/search`；可依條件分批抓訂單；回應內含付款、物流、金額、時間戳等欄位，適合作為增量水位｡ fileciteturn15file9L5-L10

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_raw.raw_orders` (
  _ingestion_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  _ingestion_date DATE NOT NULL DEFAULT CURRENT_DATE(),
  response_json STRING
)
PARTITION BY _ingestion_date;
```

---

### raw\_products

來源：`GET /v1/products`；支援欄位選擇（fields／excludes）與分頁；官方公告 2025‑07‑30 將對商品相關 API 進行重大異動，建議保留原始版本以利日後回溯｡ fileciteturn15file0L30-L36 fileciteturn15file0L7-L10

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_raw.raw_products` (
  _ingestion_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  _ingestion_date DATE NOT NULL DEFAULT CURRENT_DATE(),
  response_json STRING
)
PARTITION BY _ingestion_date;
```

---

### raw\_customer\_promotions

來源：`GET /v1/customers/:customerID/promotions`；可取得顧客當前可用促銷／優惠資訊，利於行銷分群｡ fileciteturn15file1L29-L34

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_raw.raw_customer_promotions` (
  _ingestion_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  _ingestion_date DATE NOT NULL DEFAULT CURRENT_DATE(),
  customer_id STRING,
  response_json STRING
)
PARTITION BY _ingestion_date;
```

---

### raw\_member\_point\_rules

來源：`GET /v1/member_point_rules`；提供點數賺取與到期規則等資料｡ fileciteturn15file4L17-L21

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_raw.raw_member_point_rules` (
  _ingestion_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  _ingestion_date DATE NOT NULL DEFAULT CURRENT_DATE(),
  response_json STRING
)
PARTITION BY _ingestion_date;
```

---

## Silver 層（Core 正規化）

Silver 層將 Raw JSON 解析、型別化、去重後形成維度與事實表；所有主鍵保留原始 Shopline id（STRING），並附帶來源與處理欄位｡

### dim\_customer

來源欄位自 customers API；批次抓取支援 updated\_after／previous\_id，有利維護「最後版本」紀錄｡ fileciteturn15file11L19-L25

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.dim_customer` (
  customer_id STRING NOT NULL,
  email STRING,
  phone STRING,
  name STRING,
  gender STRING,
  birthday DATE,
  country STRING,
  city STRING,
  tags ARRAY<STRING>,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  raw_payload STRING,
  _ingested_at TIMESTAMP,
  _source_date DATE,
  _is_deleted BOOL DEFAULT FALSE
)
PARTITION BY DATE(updated_at)
CLUSTER BY customer_id;
```

---

### dim\_customer\_contact\_index

透過 customers/search 以 email、手機、關鍵字等欄位搜尋顧客；可作為身份對映（AI 對話回填）與資料校補｡ fileciteturn15file16L41-L44

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.dim_customer_contact_index` (
  contact_hash STRING NOT NULL,
  contact_type STRING,            -- email / phone / external_id
  contact_value STRING,
  customer_id STRING,
  first_seen TIMESTAMP,
  last_seen TIMESTAMP,
  source STRING,                  -- customers_api / ai_conversation / manual
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(last_seen)
CLUSTER BY contact_hash, customer_id;
```

---

### dim\_member\_point\_rule

點數規則資料；供忠誠度計算與點數推估｡ fileciteturn15file4L17-L21

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.dim_member_point_rule` (
  rule_id STRING NOT NULL,
  rule_name STRING,
  earn_rate NUMERIC,      -- 每消費金額換點比例（需依 API 欄位映射）
  expiry_days INT64,      -- 點數有效天數（若有）
  is_active BOOL,
  effective_start TIMESTAMP,
  effective_end TIMESTAMP,
  raw_payload STRING,
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(effective_start)
CLUSTER BY rule_id;
```

---

### dim\_product

商品主檔；來自 /v1/products；因官方將於 2025‑07‑30 有商品相關 API 異動，建議保留 `api_version` 與欄位變異 JSON｡ fileciteturn15file0L30-L36 fileciteturn15file0L7-L10

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.dim_product` (
  product_id STRING NOT NULL,
  handle STRING,
  title STRING,
  sku_root STRING,
  active BOOL,
  product_type STRING,
  vendor STRING,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  price_cents INT64,
  compare_at_price_cents INT64,
  currency_iso STRING,
  tags ARRAY<STRING>,
  api_version STRING,
  raw_payload STRING,
  _ingested_at TIMESTAMP,
  _source_date DATE,
  _is_deleted BOOL DEFAULT FALSE
)
PARTITION BY DATE(updated_at)
CLUSTER BY product_id;
```

---

### dim\_product\_detail

單筆深度資訊（例如多規格、庫存、圖片、描述等），可於差異時補抓；對應 /v1/products/\:id 詳情｡ fileciteturn15file10L12-L36

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.dim_product_detail` (
  product_id STRING NOT NULL,
  variant_id STRING,
  option_names ARRAY<STRING>,
  option_values ARRAY<STRING>,
  inventory_qty INT64,
  weight NUMERIC,
  weight_unit STRING,
  barcode STRING,
  image_urls ARRAY<STRING>,
  description_html STRING,
  updated_at TIMESTAMP,
  raw_payload STRING,
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(updated_at)
CLUSTER BY product_id, variant_id;
```

---

### fact\_order\_header

訂單頭資料；/v1/orders/search 回應內含訂單金額、付款、物流、時間戳等｡ fileciteturn15file9L41-L58

此外，訂單回應亦直接帶出 customer\_id（可作顧客關聯）；此點在訂單 JSON 例中可見｡ fileciteturn16file4L5-L8

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.fact_order_header` (
  order_id STRING NOT NULL,
  order_number STRING,
  customer_id STRING,
  order_status STRING,
  financial_status STRING,
  fulfillment_status STRING,
  currency_iso STRING,
  total_amount_cents INT64,
  subtotal_amount_cents INT64,
  discount_amount_cents INT64,
  shipping_amount_cents INT64,
  tax_amount_cents INT64,
  payment_method STRING,
  logistic_method STRING,
  created_at TIMESTAMP,
  paid_at TIMESTAMP,
  updated_at TIMESTAMP,
  cancelled_at TIMESTAMP,
  fulfilled_at TIMESTAMP,
  raw_payload STRING,
  _ingested_at TIMESTAMP,
  _source_date DATE
)
PARTITION BY DATE(created_at)
CLUSTER BY customer_id, order_status;
```

---

### fact\_order\_line

訂單明細（展開品項）；訂單 JSON 中 subtotal\_items／promotion\_items 等欄位可拆成一列一商品，含品項價格、數量與促銷折抵｡ fileciteturn16file4L81-L90

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.fact_order_line` (
  order_id STRING NOT NULL,
  line_id STRING NOT NULL,
  line_type STRING,                  -- Product / ProductSet / Promotion / Shipping 等
  product_id STRING,
  variant_id STRING,
  quantity INT64,
  line_price_cents INT64,
  line_discount_cents INT64,
  currency_iso STRING,
  promotion_id STRING,
  raw_payload STRING,
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(_ingested_at)
CLUSTER BY order_id, product_id;
```

---

### bridge\_order\_customer

雖然 fact\_order\_header 已含 customer\_id，但建立一個橋接表可加速多對多分析（包含歷史變更或匿名訂單補 mapping）。訂單 JSON 範例顯示訂單內含 customer\_id 等顧客資訊｡ fileciteturn16file4L5-L8

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.bridge_order_customer` (
  order_id STRING NOT NULL,
  customer_id STRING NOT NULL,
  order_created_at TIMESTAMP,
  link_confidence FLOAT64 DEFAULT 1.0,  -- 1.0=API 原生；<1.0=模糊比對
  link_method STRING,                   -- api / email_match / phone_match / heuristic
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(order_created_at)
CLUSTER BY customer_id;
```

---

### fact\_customer\_promotion\_eligibility

顧客當前可用促銷清單；資料來自 /v1/customers/\:id/promotions｡ fileciteturn15file1L29-L34

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.fact_customer_promotion_eligibility` (
  customer_id STRING NOT NULL,
  promotion_id STRING NOT NULL,
  promotion_type STRING,
  promotion_name STRING,
  start_at TIMESTAMP,
  end_at TIMESTAMP,
  usage_limit INT64,
  usage_count INT64,
  raw_payload STRING,
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(_ingested_at)
CLUSTER BY customer_id, promotion_id;
```

---

## AI 對話資料（Silver 擴充）

AI 客服／聊天紀錄需與顧客身份對映；customers/search 可協助依 email、手機等欄位反查顧客，適合資料修補｡ fileciteturn15file16L41-L44

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.fact_ai_conversation_event` (
  conversation_id STRING NOT NULL,
  message_id STRING NOT NULL,
  event_ts TIMESTAMP NOT NULL,
  user_external_id STRING,
  customer_id STRING,
  channel STRING,             -- line / webchat / email / ig / phone
  sender_type STRING,         -- user / agent / bot
  message_text STRING,
  intent STRING,
  sentiment_score FLOAT64,
  category STRING,            -- 手動或模型分類
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(event_ts)
CLUSTER BY customer_id, channel;
```

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.fact_ai_conversation_daily` (
  conversation_date DATE NOT NULL,
  customer_id STRING,
  channel STRING,
  message_count INT64,
  user_message_count INT64,
  bot_message_count INT64,
  distinct_intents ARRAY<STRING>,
  top_intent STRING,
  avg_sentiment FLOAT64,
  last_message_ts TIMESTAMP,
  _ingested_at TIMESTAMP
)
PARTITION BY conversation_date
CLUSTER BY customer_id;
```

---

## Gold 層（Mart／CDP 寬表）

Gold 層針對行銷分析預先計算常用指標：RFM、購買週期、客服互動、促銷資格等｡

### mart\_customer\_profile

基本顧客資訊＋最後一次訂單時間＋最近客服互動摘要｡ 主要取材自 dim\_customer、fact\_order\_header（含時間與金額）與 AI 對話每日表｡ 訂單回應內含付款、物流與時間戳；可抓出最近一次交易｡ fileciteturn15file9L41-L58 fileciteturn16file4L5-L8

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_mart.mart_customer_profile` (
  customer_id STRING NOT NULL,
  email STRING,
  phone STRING,
  name STRING,
  gender STRING,
  birthday DATE,
  city STRING,
  first_order_ts TIMESTAMP,
  last_order_ts TIMESTAMP,
  lifetime_orders INT64,
  lifetime_spend_cents INT64,
  last_ai_touch_ts TIMESTAMP,
  last_ai_top_intent STRING,
  last_ai_avg_sentiment FLOAT64,
  loyalty_points_balance NUMERIC,   -- 之後可從點數日快照計算
  loyalty_tier STRING,
  _snapshot_date DATE
)
PARTITION BY _snapshot_date
CLUSTER BY customer_id;
```

---

### mart\_customer\_rfm

R、F、M 指標；可用訂單金額（total\_amount\_cents）、訂單數、最後付款／建立時間計算｡ /v1/orders/search 回應含金額與時間欄位｡ fileciteturn15file9L41-L48

同時透過訂單內 customer\_id 建立顧客關聯｡ fileciteturn16file4L5-L8

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_mart.mart_customer_rfm` (
  snapshot_date DATE NOT NULL,
  customer_id STRING NOT NULL,
  recency_days INT64,
  frequency_orders_90d INT64,
  frequency_orders_365d INT64,
  monetary_cents_90d INT64,
  monetary_cents_365d INT64,
  r_score INT64,
  f_score INT64,
  m_score INT64,
  rfm_segment STRING,
  _calculated_at TIMESTAMP
)
PARTITION BY snapshot_date
CLUSTER BY customer_id;
```

---

### mart\_customer\_purchase\_cycle

以顧客歷史訂單計算平均／中位購買間隔、最近一次到本次間隔、預測下次回購日｡ 訂單回應提供 created\_at／paid\_at／updated\_at 可作時間基準｡ fileciteturn15file9L41-L58

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_mart.mart_customer_purchase_cycle` (
  snapshot_date DATE NOT NULL,
  customer_id STRING NOT NULL,
  orders_count INT64,
  avg_days_between FLOAT64,
  median_days_between FLOAT64,
  last_interval_days INT64,
  expected_next_purchase_date DATE,
  confidence FLOAT64,
  _calculated_at TIMESTAMP
)
PARTITION BY snapshot_date
CLUSTER BY customer_id;
```

---

### mart\_customer\_ai\_touchpoints

整併每日 AI 對話與顧客關聯；customers/search 可支援資料補對映；適合分析客服互動與購買關係｡ fileciteturn15file16L41-L44

```sql
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_mart.mart_customer_ai_touchpoints` (
  snapshot_date DATE NOT NULL,
  customer_id STRING,
  conversation_days_30d INT64,
  user_messages_30d INT64,
  bot_messages_30d INT64,
  distinct_intents_30d INT64,
  neg_sentiment_count_30d INT64,
  last_conversation_ts TIMESTAMP,
  _calculated_at TIMESTAMP
)
PARTITION BY snapshot_date
CLUSTER BY customer_id;
```

---

### mart\_customer\_offer\_wallet

彙總顧客目前可用促銷（起訖時間、使用次數）供行銷；資料源自 /v1/customers/\:id/promotions｡ fileciteturn15file1L29-L34

```sql
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_mart.mart_customer_offer_wallet` (
  snapshot_date DATE NOT NULL,
  customer_id STRING NOT NULL,
  promotion_id STRING NOT NULL,
  promotion_name STRING,
  promotion_type STRING,
  valid_from TIMESTAMP,
  valid_to TIMESTAMP,
  usage_limit INT64,
  usage_count INT64,
  is_active BOOL,
  _calculated_at TIMESTAMP
)
PARTITION BY snapshot_date
CLUSTER BY customer_id, promotion_id;
```

---

## 增量載入與時間旅行（建議）

* **Customers 增量水位**：使用 updated\_after 參數；API 提供分頁 previous\_id 以避免漏抓｡ 
* **Orders 增量水位**：依 /v1/orders/search 回應中的 updated\_at 欄位（付款／物流更新也會改動）；適合高頻抽取｡ 
* **Products 雙軌抽**：因 2025‑07‑30 Breaking Change，建議 parallel 抓舊版與新欄位（fields／excludes 控制）以檢驗差異｡ 
* **顧客促銷同步**：行銷活動前後批次抓 /customers/\:id/promotions 更新顧客錢包｡
---

## ６．快速測試查詢範例（供你檢驗 ETL）

以下查詢片段僅示意；實際欄位需依你解析 JSON 後的欄位名調整｡

```
-- 最近 7 天新增或更新顧客
SELECT *
FROM `{{project_id}}.shopline_core.dim_customer`
WHERE updated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY updated_at DESC;
```

```
-- 顧客終身消費
SELECT c.customer_id,
       COUNT(DISTINCT h.order_id) AS lifetime_orders,
       SUM(h.total_amount_cents) AS lifetime_spend_cents
FROM `{{project_id}}.shopline_core.dim_customer` c
LEFT JOIN `{{project_id}}.shopline_core.fact_order_header` h USING (customer_id)
GROUP BY 1;
```

```
-- 展開訂單明細查看商品銷量
SELECT l.product_id,
       SUM(l.quantity) AS qty,
       SUM(l.line_price_cents - IFNULL(l.line_discount_cents,0)) AS net_sales_cents
FROM `{{project_id}}.shopline_core.fact_order_line` l
GROUP BY 1
ORDER BY qty DESC;
```

---

## AI 對話分類資料模型（含情緒／Intent／解析管線）

針對第二階段需求——整併「AI 客服對話紀錄」與「每日對話分類」——提供資料模型、欄位設計、解析管線與與`Shopline`顧客／訂單資料串接策略｡

> 快速重點︰
> １．原始訊息（每則 message event）→ NLU 推論（intent／情緒）→ 會話彙總（每日、會話級）→ 顧客對映（identity resolution）→ 行銷分析寬表｡
> ２．顧客對映可使用 email／手機／其他識別碼透過 `customers/search` 進行補對照｡ fileciteturn15file16L41-L44
> ３．若已知訂單包含 customer\_id，亦可在互動後根據顧客下單郵件或訊息資訊回填對映｡ fileciteturn16file4L5-L8

---

### ８．１ 資料流高階架構

```
[Channel 事件：LINE／WebChat／Email Bot]
        │  (Streaming, JSON)
        ▼
Pub/Sub topic: ai_conversation_events
        ▼ Dataflow / Beam (schema normalize)
BQ Bronze: raw_ai_messages
        ▼ Batch / Mini-batch N minutes
NLU 推論服務（Vertex AI / Cloud Run 模型）
        ▼ 回寫結果
BQ Silver: fact_ai_conversation_event  (含 intent, sentiment, category)
        │
        ├─ Identity Resolver  (比對 email／phone → dim_customer_contact_index) ─┐
        │                                                                       │
        ▼                                                                       ▼
BQ Silver: fact_ai_conversation_session         BQ Silver: fact_ai_conversation_daily
        │                                                                       │
        └──────────────────────────────► BQ Gold: mart_customer_ai_touchpoints ◄─┘
```

---

### Channel Raw 訊息表（Bronze 擴充）

> 若你要保留最原始、未脫敏的對話內容，請於 Bronze 層新增 raw
> aI\_messages 表；建議對敏感欄位（例如個資）作 Tokenization｡

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_raw.raw_ai_messages` (
  _ingestion_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  _ingestion_date DATE NOT NULL DEFAULT CURRENT_DATE(),
  channel STRING,                -- line / webchat / email / ig / phone
  payload_json STRING            -- 原始事件 JSON
)
PARTITION BY _ingestion_date;
```

> 若訊息量極大，可改用分區＋叢集：Partition by \_ingestion\_date；Cluster by channel｡

---

### 訊息事件解析（Silver：fact\_ai\_conversation\_event 強化版）

在主文件 §３ 已有初版 fact\_ai\_conversation\_event；此處補強欄位（語言偵測、情緒、模型版本、原始／正規化文字、PII 遮罩旗標等）｡

```
CREATE OR REPLACE TABLE `{{project_id}}.shopline_core.fact_ai_conversation_event` (
  conversation_id STRING NOT NULL,      -- 外部客服／Bot 平台提供之會話 ID
  message_id STRING NOT NULL,
  event_ts TIMESTAMP NOT NULL,
  message_seq INT64,                    -- 同一會話排序
  user_external_id STRING,              -- Channel 端使用者 ID (Line UID, email 等)
  customer_id STRING,                   -- 映射後 Shopline 顧客 ID；未解決則 NULL｡
  contact_match_confidence FLOAT64,     -- Identity matching 信心度 (0-1)
  channel STRING,                       -- line / webchat / email / ig / phone
  sender_type STRING,                   -- user / agent / bot / system
  language_code STRING,                 -- zh-TW / en / ...
  message_text STRING,
  message_text_masked STRING,           -- PII 遮罩後文字
  intent STRING,                        -- 主 intent (model top-1)
  intent_confidence FLOAT64,
  intents_all ARRAY<STRUCT<intent STRING, score FLOAT64>>,
  sentiment_score FLOAT64,              -- -1 ~ +1
  sentiment_magnitude FLOAT64,
  sentiment_label STRING,               -- positive / negative / neutral
  category STRING,                      -- 自訂客服分類 (ex: 售後 / 物流 / 發票)
  emotion STRING,                       -- anger / joy / sadness / ... (選用)
  toxicity_score FLOAT64,               -- 過激言論指標 (選用)
  model_version STRING,                 -- NLU / Sentiment 模型版本
  raw_payload STRING,                   -- from Bronze (清洗後 JSON)
  _ingested_at TIMESTAMP,
  _processed_at TIMESTAMP
)
PARTITION BY DATE(event_ts)
CLUSTER BY customer_id, channel;
```

Identity Resolver 會填寫 customer\_id／contact\_match\_confidence；此 Resolver 可呼叫 Shopline `customers/search` 依 email／手機等欄位反查顧客｡ 

---

### 會話 Session 表（Silver 新增）

多數客服平台將多則訊息歸屬單一會話；若平台僅提供訊息流，你可自訂 session 規則（例：同 user\_external\_id 且 30 分鐘內無互動則結束）。此表彙總會話層級統計，並試圖對映顧客｡訂單回應內含 customer\_id，可在顧客與互動後產生消費時回寫連結。

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.fact_ai_conversation_session` (
  conversation_id STRING NOT NULL,
  customer_id STRING,                   -- 會話期間解決身份則填寫
  channel STRING,
  session_start_ts TIMESTAMP,
  session_end_ts TIMESTAMP,
  duration_seconds INT64,
  user_message_count INT64,
  agent_message_count INT64,
  bot_message_count INT64,
  first_response_latency_seconds INT64, -- 第一次客服回應延遲
  intents_array ARRAY<STRING>,
  dominant_intent STRING,
  avg_sentiment FLOAT64,
  min_sentiment FLOAT64,
  max_sentiment FLOAT64,
  neg_sentiment_flag BOOL,
  escalated_flag BOOL,                  -- 是否轉真人
  resolved_flag BOOL,                   -- 是否標記為已解決
  related_order_id STRING,              -- 若會話導向訂單
  raw_payload STRING,                   -- Session level meta JSON
  _ingested_at TIMESTAMP,
  _processed_at TIMESTAMP
)
PARTITION BY DATE(session_start_ts)
CLUSTER BY customer_id, channel;
```

---

### 每日對話分類彙總（Silver：fact\_ai\_conversation\_daily 強化版）

此表於既有定義基礎上新增情緒分桶與意圖分布統計｡

```
CREATE OR REPLACE TABLE `{{project_id}}.shopline_core.fact_ai_conversation_daily` (
  conversation_date DATE NOT NULL,
  customer_id STRING,
  channel STRING,
  conversation_count INT64,
  message_count INT64,
  user_message_count INT64,
  bot_message_count INT64,
  distinct_intents ARRAY<STRING>,
  top_intent STRING,
  intent_counts ARRAY<STRUCT<intent STRING, cnt INT64>>,
  avg_sentiment FLOAT64,
  pos_sentiment_pct FLOAT64,
  neg_sentiment_pct FLOAT64,
  neu_sentiment_pct FLOAT64,
  last_message_ts TIMESTAMP,
  _ingested_at TIMESTAMP,
  _processed_at TIMESTAMP
)
PARTITION BY conversation_date
CLUSTER BY customer_id;
```

顧客對映仍依賴 `customers/search`（email／手機）或訂單 customer\_id 回填。
---

### Intent 分類體系維度（Taxonomy）

為了維持跨渠道一致性，建議建立一份 Intent Taxonomy 維度表，將模型輸出（細分類）對映成營運語彙（粗分類 → 類別 → 主題）｡

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.dim_intent_taxonomy` (
  intent_code STRING NOT NULL,        -- 模型內部代碼
  intent_name STRING,                 -- 易讀名稱
  intent_group STRING,                -- 上層群組 (ex: 售後服務)
  intent_topic STRING,                -- 子主題 (ex: 退貨 / 換貨)
  is_active BOOL,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(updated_at)
CLUSTER BY intent_code;
```

---

### 模型版本註冊表（Model Registry）

追蹤情緒／Intent 模型版本、訓練資料集、門檻值等，方便回溯分析。

```
CREATE TABLE IF NOT EXISTS `{{project_id}}.shopline_core.dim_nlu_model_version` (
  model_version STRING NOT NULL,
  model_type STRING,                -- intent / sentiment / emotion / toxicity
  training_data_hash STRING,
  training_end_date DATE,
  deployed_at TIMESTAMP,
  threshold_default FLOAT64,
  notes STRING,
  _ingested_at TIMESTAMP
)
PARTITION BY DATE(deployed_at)
CLUSTER BY model_version;
```

---

### Identity Resolution 流程細節

**步驟 A．抓取可用識別資訊：** 由訊息事件取得 user\_external\_id、可能的 email／手機、促銷碼；同時查訂單紀錄（若聊天中建立訂單）取得 customer\_id｡ 訂單 JSON 包含 customer\_id，可協助對映。

**步驟 B．以 Shopline API 反查：** 呼叫 `GET /v1/customers/search` 依 email、手機等條件補對映缺漏的顧客資料。

**步驟 C．寫入 dim\_customer\_contact\_index：** 建立 contact\_hash（標準化 email／手機），更新或新增顧客對映索引

---

### 解析管線實作（Pseudo Flow）

**觸發：** Pub/Sub 接收渠道事件 → Dataflow Streaming 作 schema normalization → Bronze raw\_ai\_messages｡

**Batch NLU：** Cloud Run (or Vertex AI Batch Prediction) 每 5 分鐘抓 Bronze 新資料 → 呼叫模型 → 寫入 Silver fact\_ai\_conversation\_event｡

**Identity Match：** BigQuery Stored Procedure：對於 customer\_id IS NULL 的訊息，擷取 email／手機 → 透過外部函式呼叫 Shopline `customers/search` → 回寫 customer\_id；同時更新 dim\_customer\_contact\_index｡ 

**Sessionization：** SQL/Beam 作會話切分；輸出 fact\_ai\_conversation\_session｡

**Daily Rollup：** Partition Merge Query 彙總至 fact\_ai\_conversation\_daily｡

**Mart 更新：** 更新 mart\_customer\_ai\_touchpoints；再與 mart\_customer\_rfm、mart\_customer\_profile Join 供 Looker｡

---

### 與訂單行為關聯分析範例 SQL

**A．客服互動後 7 天轉單率**

```
WITH ai AS (
  SELECT customer_id, MAX(event_ts) AS last_ai_ts
  FROM `{{project_id}}.shopline_core.fact_ai_conversation_event`
  WHERE sentiment_label = 'negative'
  GROUP BY 1
),
ord AS (
  SELECT customer_id, COUNT(*) AS orders_after
  FROM `{{project_id}}.shopline_core.fact_order_header`
  WHERE created_at BETWEEN TIMESTAMP_ADD(ai.last_ai_ts, INTERVAL 1 MINUTE)
                      AND TIMESTAMP_ADD(ai.last_ai_ts, INTERVAL 7 DAY)
  GROUP BY 1
)
SELECT ai.customer_id,
       orders_after
FROM ai
LEFT JOIN ord USING (customer_id);
```

訂單表可由訂單 JSON 取得 customer\_id 關聯顧客。

**B．Top Intent 與負向情緒分布**

```
SELECT dominant_intent,
       COUNT(*) AS sessions,
       AVG(avg_sentiment) AS avg_sentiment
FROM `{{project_id}}.shopline_core.fact_ai_conversation_session`
GROUP BY 1
ORDER BY sessions DESC;
```

---

### 敏感資料與隱私

* 在 Bronze 保留原始訊息時強烈建議 PII 遮罩；將原始明碼改為 Token，僅於 Identity Resolver 暫時解密｡
* Silver 層 message\_text 建議僅存去識別化版本；原文另存 Secure GCS 並設 Object ACL｡
* 在建 contact\_hash 時以標準化（lowercase email、E.164 phone）後 SHA256 雜湊儲存；需回查時再透過映射表解碼｡
* 與 Shopline 顧客對映所需 email／手機資訊可透過 `customers/search` 查詢，不必長期保存明碼；降低風險

---

### 監控與品質指標

| 指標            | 描述                                                      | 判斷          | 修復動作                                                 |
| ------------- | ------------------------------------------------------- | ----------- | ---------------------------------------------------- |
| 未對映顧客率        | fact\_ai\_conversation\_event 中 customer\_id IS NULL 比率 | >20% 警示     | 觸發批次 customers/search 回填 |
| 對映衝突          | 同 contact\_hash 映射多個 customer\_id                       | 人工審核        | 更新 contact\_index｡                                   |
| Session 無訂單連結 | 會話後 7 天無任何訂單，但客服標記成功                                    | 檢查訂單 API 抽取 | 透過訂單內 customer\_id 回填  |

---

### 待辦清單（`AI`對話模型落地）

| 優先 | 任務                                    | 產出                                                            | 備註                                   |
| -- | ------------------------------------- | ------------------------------------------------------------- | ------------------------------------ |
| P0 | 建 raw\_ai\_messages Landing           | Bronze 表                                                      | Pub/Sub → BQ                         |
| P0 | NLU 模型選型＆API 封裝                       | Cloud Run 服務                                                  | Intent／情緒模型                          |
| P1 | Identity Resolver 呼叫 customers/search | 函式／DAG                                                        | 顧客對映｡ |
| P1 | Sessionization Job                    | fact\_ai\_conversation\_session                               | 依 30 分閒置規則                           |
| P2 | Daily Rollup & Mart 更新                | fact\_ai\_conversation\_daily／mart\_customer\_ai\_touchpoints | 與訂單／RFM 串接                           |
| P2 | 品質監控儀表板                               | Looker                                                        | 未對映顧客率、情緒趨勢、轉單率                      |

---