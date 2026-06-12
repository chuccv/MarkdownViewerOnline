# Thiết kế kỹ thuật — Filter Usage Analytics (Mageplaza LayeredNavigationUltimate)

- **Ngày:** 2026-06-11
- **Module đích:** `Mageplaza_LayeredNavigationUltimate`
- **Vị trí code:** `/var/www/html/magento/app/code/Mageplaza/LayeredNavigationUltimate/` (project dev hiện tại; sau khi nghiệm thu sẽ sync về source repo `/var/www/html/MP-Magento-Extension/layered-navigation-m2-ultimate/`)
- **Trạng thái:** Draft v2 — chờ duyệt
- **Phạm vi:** Thiết kế, chưa implement

---

## 1. Mục tiêu & Ràng buộc

### 1.1. Mục tiêu

Thu thập hành vi sử dụng bộ lọc (layered navigation) của khách tại frontend, lưu DB, và cung cấp cho Admin một Dashboard gồm:

1. **Top Popular Filters** — thuộc tính/giá trị được lọc nhiều nhất (chart + grid).
2. **Zero-results** — các tổ hợp filter khách tìm nhưng ra 0 sản phẩm (grid), phục vụ bộ phận thu mua.
3. Export CSV / Excel XML, lọc theo Date Range toàn cục.
4. Cron tự dọn log + report cũ theo số ngày cấu hình.

### 1.2. Ràng buộc bắt buộc (đã chốt với anh Neil)

| # | Ràng buộc | Giải pháp thiết kế |
|---|---|---|
| R1 | **Không ảnh hưởng page load** trang catalog | Tracking bắn bằng `navigator.sendBeacon()` lúc `requestIdleCallback` — fire-and-forget, không block render/navigation. JS tracker ~2KB, vanilla, load defer. |
| R2 | **Thân thiện mọi loại cache** (FPC/Varnish) | Không log phía server trong request lọc (page có thể serve từ cache, PHP không chạy). State filter nhúng vào HTML (cache-safe vì gắn với URL). Endpoint ghi log là **POST riêng** — built-in FPC chỉ cache GET/HEAD (verify tại `vendor/magento/framework/App/PageCache/Kernel.php:132,164` của chính project), nên POST không bị cache và không làm bẩn cache. |
| R3 | **Ghi data nhanh, chuẩn** | Web request chỉ làm đúng 1 việc: 1 multi-row INSERT append-only vào bảng raw (~1ms, không tranh chấp khóa). Tổng hợp report chạy nền bằng cron. |
| R6 | **Extension thương mại — không biết trước quy mô merchant** | Không giả định traffic, không đòi hỏi hạ tầng ngoài (RabbitMQ/Redis). Đường ghi nóng là append-only INSERT (scale hàng nghìn beacon/giây); aggregation tách khỏi request path theo đúng kiến trúc report của Magento core. |
| R4 | Hỗ trợ **cả Luma + Hyva**, cả **AJAX + non-AJAX** | 1 tracker vanilla JS dùng chung; cover non-AJAX bằng đọc state khi page load, cover AJAX bằng `MutationObserver` trên container bị replace. Không sửa JS của module base. |
| R5 | Tắt config = **zero footprint** | `Enable = No` → block/JS không render gì. |

### 1.3. Các quyết định đã chốt

- **Kiến trúc ghi:** sendBeacon → beacon **chỉ INSERT bảng raw** (append-only); cron **aggregate tăng dần mỗi giờ** từ raw vào bảng report theo watermark `log_id` (kiến trúc report của Magento core). Dashboard chỉ đọc bảng report, số liệu trễ tối đa ~1 giờ. Không message queue (extension không được giả định merchant có RabbitMQ/Redis; MySQL-backed queue cũng chỉ là 1 INSERT khác, không lợi gì).
- **Schema 2 bảng:** `mageplaza_layer_analytics_request_log` (raw track, bám spec gốc §3.2) + `mageplaza_layer_analytics_report` (1 bảng tổng hợp, mỗi dòng = 1 cặp (attribute, option) với **2 cột đếm** `hits` + `zero_hits`). Không cột `type`, không hash tổng hợp, không filler. Không dùng mô hình cha–con/FK CASCADE.
- **Cron clean chỉ dọn bảng raw log** — đúng nguyên văn spec gốc §2.1 ("quét bảng log... DELETE các bản ghi" — không nhắc gì tới report). Bảng report là dữ liệu tổng hợp nhỏ gọn, **giữ lâu dài** để Admin xem được xu hướng lịch sử kể cả khi raw đã bị dọn.
- **Export:** dùng `exportButton` built-in của UI listing (CSV + Excel XML). File Excel xuất ra là `.xml` (Excel 2003 XML Spreadsheet — Magento core ghi thẳng đuôi `.xml`, verify tại `ConvertToXml.php:133`). Excel/LibreOffice mở được. Không phải `.xlsx` thật, không cần PhpSpreadsheet.
- **Môi trường thực tế đã verify:** project dùng built-in FPC trên Redis (db1), không Varnish (`http_cache_hosts` null), mode developer; code `app/code/Mageplaza/LayeredNavigation*` khớp 100% source repo MP.

---

## 2. Kiến trúc tổng thể

```
┌─ FRONTEND (Luma / Hyva, FPC cached) ─────────────────────────────────┐
│                                                                       │
│  Category / Search / Custom Products Page                             │
│  └─ .layered-filter-block-container          ← bị replace sau AJAX    │
│      └─ <script type="application/json"                               │
│             id="mp-layer-analytics-state">   ← ViewModel render        │
│         {"filters":[...],"resultCount":12,"url":"..."}                 │
│                                                                       │
│  mp-layer-tracker.js (defer, vanilla, ~2KB, nằm NGOÀI vùng AJAX)      │
│   ├─ page load        → đọc state  ─┐  (cover non-AJAX + direct URL)  │
│   └─ MutationObserver → đọc state  ─┤  (cover AJAX Luma + Hyva)       │
│                                     ▼                                 │
│        filters rỗng? → bỏ qua                                         │
│        filters có data → sendBeacon ngay                               │
│        navigator.sendBeacon(POST /mplayer/analytics/track)            │
└───────────────────────────────────┬───────────────────────────────────┘
                                    │ POST (bypass FPC)
┌─ BACKEND ──────────────────────────▼──────────────────────────────────┐
│  Controller\Analytics\Track (HttpPost + CSRF-exempt, trả 204)          │
│   └─ Model\Analytics\Logger::log(payload, sessionId, storeId)         │
│       └─ INSERT  mageplaza_layer_analytics_request_log                │
│          (N dòng, 1 query, append-only — không hot-row, mọi quy mô)   │
│                                                                       │
│  Cron\Aggregate (0 * * * *) → đọc raw log_id > watermark, UPSERT 1 bảng│
│   report: mỗi (attr,value)/giờ → hits += COUNT(*),                     │
│           zero_hits += COUNT(* WHERE result_count=0)                   │
│                                                                       │
│  Cron\CleanLog (0 1 * * *) → DELETE raw log cũ theo batch             │
│   (KHÔNG đụng report — report chỉ xóa khi Admin bấm Clear Report)     │
│                                                                       │
│  ADMIN  mplayer/analytics/index   (đọc DUY NHẤT bảng report)          │
│   ├─ Date Range selector (global)                                     │
│   ├─ Bar chart Top 5 attributes (Chart.js bundled Magento admin)      │
│   ├─ Grid: Top Popular Filters  (+ Export CSV/Excel XML)              │
│   └─ Grid: Zero-results         (+ Export CSV/Excel XML)              │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 3. Cơ chế thu thập dữ liệu

### 3.1. Nhúng state phía server (ViewModel)

**File mới:** `ViewModel/AnalyticsState.php`

- Inject `Magento\Catalog\Model\Layer\Resolver` → lấy layer hiện hành.
- Trả về JSON:
  - `filters`: mảng `{code, value}` từ `$layer->getState()->getFilters()` — `value` là giá trị raw trên URL (option id / `100-200`). **Không gửi `label`** — label là presentation, có thể thay đổi, không lưu vào DB.
  - **Quy tắc multi-select:** 1 phần tử = 1 cặp (attribute, value). 5 attribute / 8 value = payload 8 phần tử = 8 dòng raw chung `request_uid` (1 câu INSERT). Giới hạn 20 phần tử/payload đếm theo cặp. Cả Top Filters (`hits`) lẫn Zero-results (`zero_hits`) đều đếm theo từng (attribute, value): 1 lượt ra 0 sản phẩm với 3 filter → mỗi dòng trong 3 dòng `+1 zero_hits`.
  - `resultCount`: `$layer->getProductCollection()->getSize()` — chính xác, không scrape DOM.
  - `url`: URL hiện tại (không gồm query `_=`...).
  - `endpoint`: URL POST track (build sẵn theo store).
- Chỉ render khi `Helper::isAnalyticsEnabled()` = true. Trả chuỗi rỗng nếu tắt (R5).

**Layout & template:**

- `view/frontend/layout/catalog_category_view.xml`, `catalogsearch_result_index.xml`, layout tương ứng của Custom Products Page (route `mplayer` của Ultimate), và bộ `hyva_*` tương ứng — thêm 2 block:
  - `mp.layer.analytics.state` (template `state.phtml`) **bên trong** khối layer navigation (container `.layered-filter-block-container`) để JSON được làm mới theo mỗi response AJAX. Template **chỉ in thẻ JSON**, tuyệt đối không chứa `<script src>` — vì jQuery `.html()` của Luma thực thi lại mọi script trong HTML được replace, sẽ gây load tracker trùng sau mỗi lần lọc.
  - `mp.layer.analytics.tracker` (template `tracker.phtml`) gắn vào container `before.body.end` — nằm **ngoài** vùng bị AJAX replace, in thẻ `<script defer src=".../mp-layer-tracker.js">` đúng 1 lần cho cả vòng đời trang.
- **Cache-safety:** JSON là hàm thuần của URL → trang nào cache thì JSON đó đúng với trang đó. Không có dữ liệu theo phiên/khách trong JSON.

### 3.2. Tracker JS

**File mới:** `view/frontend/web/js/mp-layer-tracker.js` — vanilla JS, KHÔNG phụ thuộc jQuery/RequireJS/Alpine (chạy được cả Luma lẫn Hyva), bọc toàn bộ trong try/catch + IIFE.

Luồng xử lý:

1. `DOMContentLoaded` → `readAndSend()` — cover **non-AJAX** (full reload) và khách vào thẳng URL có filter (kể cả trang serve từ FPC).
2. `MutationObserver` quan sát `childList` của `.layered-filter-block-container` (debounce 300ms) → `readAndSend()` — cover **AJAX** của cả Luma (`submit-filter.js` replace bằng jQuery `.html()`) và Hyva (`ajaxSubmit` replace bằng `innerHTML`). Không đụng vào code base module.
3. `readAndSend()`:
   - Parse JSON từ `#mp-layer-analytics-state`; không có thẻ hoặc `filters` rỗng → bỏ qua (không log pageview thường, không log khi bỏ hết filter).
   - Không dùng `sessionStorage` hay hash check — F5/reload trên cùng URL filter vẫn tính là 1 lượt quan tâm.
   - `requestIdleCallback` (fallback `setTimeout 0`) → `navigator.sendBeacon(endpoint, Blob JSON)`; fallback `fetch(..., {keepalive: true})` cho browser không có sendBeacon.
4. Payload gửi đi:

```json
{
  "filters": [{"code": "color", "value": "5"}],
  "result_count": 0,
  "url": "https://store.com/jackets?color=5&size=42"
}
```

### 3.3. Endpoint ghi log

**Files mới:**

- `etc/frontend/routes.xml` — đăng ký module vào route frontend hiện có của layer (frontName `mplayer`).
- `Controller/Analytics/Track.php` — implement `HttpPostActionInterface` + `CsrfAwareActionInterface` (`createCsrfValidationException` trả null, `validateForCsrf` trả true — chuẩn cho beacon endpoint không có form_key).

Xử lý:

1. Config tắt → 204 luôn (endpoint chết im lặng, không lộ thông tin).
2. Parse + validate payload: JSON hợp lệ, `filters` là mảng 1–20 phần tử, mỗi phần tử chỉ có `code` ≤ 64 và `value` ≤ 255 (không có `label`), `url` ≤ 2048, `result_count` ép int ≥ 0. Sai → 204 + bỏ qua (không bao giờ trả 4xx/5xx ra khách).
3. Lấy `session_id` từ `Magento\Framework\Session\SessionManagerInterface`, `store_id` từ `StoreManagerInterface` — **không tin client**.
4. Gọi writer qua **`Api/AnalyticsWriterInterface`** — v1 preference là `Model\Analytics\Logger`. Interface là chỗ cắm sẵn: merchant quy mô cực lớn có RabbitMQ có thể swap `QueueWriter` qua `di.xml` mà không đụng controller/tracker/schema.
5. `Model\Analytics\Logger::log()` — **chỉ làm đúng 1 việc**:
   - Sinh `request_uid` (md5 uniqid) cho lượt này — chỉ để nhóm các dòng cùng lượt. Không tính hash ở web request.
   - **INSERT multi-row append-only** vào `..._request_log` (N filter = N dòng, 1 câu query, ~1ms).
   - **Không đụng bảng report** trong web request — tránh hoàn toàn tranh chấp hot-row khi nhiều user cùng lọc một giá trị phổ biến (extension không biết trước quy mô merchant). Tổng hợp do cron đảm nhiệm (§6).
6. Exception → nuốt im lặng (không ghi log file riêng). Response: `204 No Content`, `Cache-Control: no-store`, không render layout (`ResultFactory::TYPE_RAW`).

**Chi phí mỗi beacon:** bootstrap Magento + đúng 1 multi-row INSERT (không đụng report) ≈ nhẹ hơn nhiều một page view; beacon chạy nền nên khách không cảm nhận được (R1, R3).

---

## 4. DB Schema (`etc/db_schema.xml` + `db_schema_whitelist.json` của Ultimate)

Mô hình **2 bảng**: 1 bảng raw track + 1 bảng report tổng hợp dùng **2 cột đếm** (`hits` + `zero_hits`) trên cùng 1 dòng/giá-trị-filter. Không có FK — report là dữ liệu dẫn xuất (derived), cron cộng dồn từ raw.

### 4.1. `mageplaza_layer_analytics_request_log` — raw track, 1 dòng = 1 cặp (attribute_code, option_value) trong 1 lượt lọc

Bám spec gốc §3.2, bổ sung các cột cần cho việc nhóm tổ hợp và multi-store:

| Cột | Kiểu | Null | Ghi chú |
|---|---|---|---|
| `log_id` | int unsigned, PK, AI | N | |
| `request_uid` | char(32) | N | *bổ sung* — nhóm các filter cùng 1 lượt lọc (server sinh md5(uniqid)) |
| `session_id` | varchar(64) | N | phân biệt phiên |
| `store_id` | smallint unsigned | N | *bổ sung* — multi-store, default 0 |
| `attribute_code` | varchar(64) | N | |
| `option_value` | varchar(255) | N | giá trị raw (option id / `100-200`) — **không lưu label** |
| `target_url` | varchar(2048) | N | URL trang khách đang lọc |
| `result_count` | int unsigned | N | số sản phẩm trả về của lượt lọc |
| `created_at` | timestamp, default CURRENT_TIMESTAMP | N | |

**Indexes:** (`created_at`) — cron clean + truy vấn đối soát; (`request_uid`) — nhóm các dòng cùng lượt khi cron gộp; (`store_id`).

Bảng này là **nguồn chân lý** (source of truth): dùng đối soát, debug, và tái dựng lại bảng report nếu cần (re-aggregate theo `request_uid`).

### 4.2. `mageplaza_layer_analytics_report` — tổng hợp theo giờ, dashboard đọc trực tiếp

| Cột | Kiểu | Null | Ghi chú |
|---|---|---|---|
| `report_id` | int unsigned, PK, AI | N | |
| `report_date` | date | N | ngày ghi nhận |
| `report_hour` | smallint unsigned | N | 0–23 — giữ chiều giờ để phân tích khung giờ |
| `store_id` | smallint unsigned | N | default 0 |
| `attribute_code` | varchar(64) | N | code thô |
| `option_value` | varchar(255) | N | option id / range value thô — **không lưu label**, resolve động khi query |
| `hits` | int unsigned | N | tổng số lượt filter này được dùng — phục vụ **Top Popular** |
| `zero_hits` | int unsigned | N | số lượt nằm trong lần lọc ra **0 sản phẩm** — phục vụ **Zero-result** |
| `last_requested` | timestamp | N | lần gần nhất |

- **UNIQUE KEY** (`report_date`, `report_hour`, `store_id`, `attribute_code`, `option_value`) — khóa cho `ON DUPLICATE KEY UPDATE`, đặt thẳng trên cột thật (đủ ngắn để index, không cần hash).
- **Indexes:** (`report_date`).

Tổng hợp ở **mức giờ** (1 dòng/giá-trị-filter/giờ): dashboard mức ngày chỉ là `SUM(hits)` / `SUM(zero_hits)` gộp 24 dòng. Bảng nhỏ (vài nghìn dòng/ngày/store) → dashboard nhanh ở mọi quy mô.

- **Top Popular** đọc cùng bảng: `ORDER BY hits DESC`.
- **Zero-result** đọc cùng bảng: `WHERE zero_hits > 0 ORDER BY zero_hits DESC` — chỉ ra **filter value** xuất hiện nhiều nhất trong các lượt ra 0 sản phẩm (mức từng giá trị, không theo URL/tổ hợp đầy đủ).

### 4.3. Resolve label động khi query

Bảng report lưu `option_value` thô — không bao giờ lưu label (label có thể đổi → vỡ report). Dashboard resolve tại query-time qua LEFT JOIN:

| Loại | Nhận biết | Resolve | Fallback |
|---|---|---|---|
| EAV select/multiselect | `option_value` là số nguyên, code không thuộc 2 loại dưới | LEFT JOIN `eav_attribute_option_value` theo `option_id` | `[code: value]` |
| Price range | `attribute_code = 'price'` | Format thẳng value thô `100 - 200` | — |
| Category | `attribute_code = 'cat'` | LEFT JOIN `catalog_category_entity_varchar` theo `entity_id = CAST(option_value AS UNSIGNED)`, attr `name`, `store_id = 0` | `Category #ID` |

### 4.4. Vì sao tách raw và report

- **Raw** trả lời "chuyện gì đã xảy ra" (audit, debug, re-aggregate) và bám đúng schema spec gốc §3.2.
- **Report** trả lời "tổng hợp bao nhiêu" — dashboard không bao giờ phải GROUP BY trên bảng raw hàng triệu dòng; cron cộng dồn sẵn mỗi giờ.
- `zero_hits` cộng theo từng dòng raw có `result_count = 0` — phản ánh đúng số lượt mỗi filter value rơi vào kết quả rỗng.
- Web request không bao giờ chạm bảng report → không hot-row contention ở bất kỳ quy mô merchant nào (R6).

---

## 5. Cấu hình hệ thống

**File sửa:** `etc/adminhtml/system.xml` (Ultimate) — thêm group vào section `layered_navigation`:

```
layered_navigation/analytics/enable          select Yes/No   default 0   (Enable Analytics)
layered_navigation/analytics/clean_log_days  text            default 90  (Clean Log Older Than (Days))
```

- `clean_log_days`: validate `validate-digits`; comment ghi rõ "Để trống hoặc 0 = không tự xóa". Field hiện khi `enable = 1` (depends).
- **File sửa:** `etc/config.xml` — default values.
- **File sửa:** `Helper/Data.php` (Ultimate, đã có) — thêm `isAnalyticsEnabled($storeId)`, `getCleanLogDays()`.

## 6. Cron: tổng hợp report + dọn dữ liệu

### 6.1. `Cron/Aggregate.php` — tổng hợp tăng dần raw → report (`0 * * * *` — mỗi giờ)

- Schedule: `0 * * * *` — mỗi giờ 1 lần, đủ cho production. Khi cần chạy ngay (dev/test): `bin/magento mplayer:analytics:aggregate` — gọi cùng logic với cron, không phải service riêng.
- **Watermark:** lưu `log_id` cuối đã xử lý qua `Magento\Framework\FlagManager` (key `mplayer_analytics_agg_watermark`). Mỗi lần chạy chỉ đọc `WHERE log_id > watermark ORDER BY log_id LIMIT 50000` — chi phí không phụ thuộc kích thước bảng.
- Vì N dòng của 1 lượt lọc được ghi bằng 1 câu multi-row INSERT nên `log_id` của chúng liền kề — ranh giới batch lùi về `request_uid` trọn vẹn cuối cùng để không cắt đôi một lượt.
- **Một câu GROUP BY** theo (`DATE(created_at)`, `HOUR(created_at)`, `store_id`, `attribute_code`, `option_value`) trên batch, tính đồng thời 2 counter:
  - `hits = COUNT(*)`
  - `zero_hits = COUNT(* WHERE result_count = 0)` → `SUM(CASE WHEN result_count = 0 THEN 1 ELSE 0 END)`
- Bulk `INSERT ... ON DUPLICATE KEY UPDATE hits = hits + VALUES(hits), zero_hits = zero_hits + VALUES(zero_hits), last_requested = VALUES(last_requested)`.
- Mỗi batch bọc 1 transaction cùng với update watermark — crash giữa chừng thì lần sau chạy lại từ watermark cũ, không mất/không đếm đôi.
- Lặp batch tới khi hết dòng mới. Chỉ chạy khi `enable = Yes`.
- **Hệ quả:** dashboard trễ tối đa ~1 giờ; trang Analytics hiển thị ghi chú "Data updated at <thời điểm watermark>".

### 6.2. `Cron/CleanLog.php` — dọn RAW LOG (`0 1 * * *`) — KHÔNG đụng bảng report

- **File mới:** `etc/crontab.xml` — group `default`, khai báo cả 2 job trên.
- `Cron/CleanLog.php`:
  - Bỏ qua nếu `enable = No` hoặc `clean_log_days` ≤ 0.
  - `DELETE FROM mageplaza_layer_analytics_request_log WHERE created_at < (NOW() - INTERVAL X DAY) LIMIT 10000` lặp tới khi affected rows = 0 — batch để không giữ lock dài.
  - **Chỉ xóa bảng raw log** — đúng nguyên văn spec gốc §2.1. **Tuyệt đối không xóa bảng report.**
  - Chỉ xóa các dòng raw có `log_id ≤ watermark` (đã được aggregate) — phòng trường hợp cron Aggregate bị kẹt lâu ngày thì dữ liệu chưa tổng hợp không bị xóa mất.

### 6.3. Xóa report — CHỈ khi Admin chủ động

- Bảng report không bao giờ bị xóa tự động. Trên trang Analytics dashboard có nút **"Clear Report Data"**:
  - Controller `Controller/Adminhtml/Analytics/ClearReport.php` (POST + form_key, `ADMIN_RESOURCE = Mageplaza_LayeredNavigationUltimate::analytics`).
  - Bấm nút → confirm dialog ("This will permanently delete all aggregated report data. Continue?") → `TRUNCATE` bảng report → message thành công, redirect về dashboard.
  - **Không reset watermark** khi clear: phần raw đã aggregate rồi sẽ không bị cộng lại (không đếm đôi); report bắt đầu tích lũy mới từ dữ liệu phát sinh sau đó.

## 7. Admin Dashboard

### 7.1. Routing / Menu / ACL

- Route admin `mplayer` đã thuộc Ultimate (`etc/adminhtml/routes.xml` hiện có) — không cần sửa.
- **File sửa:** `etc/adminhtml/menu.xml` — thêm node `Mageplaza_LayeredNavigationUltimate::analytics`, title "Analytics", action `mplayer/analytics/index`, parent `Mageplaza_LayeredNavigation::layer_navigation`, sortOrder 30 (sau Custom Products Pages, trước Configuration).
- **File sửa:** `etc/acl.xml` — resource `Mageplaza_LayeredNavigationUltimate::analytics` dưới `Mageplaza_LayeredNavigation::layer`.
- **Files mới:** `Controller/Adminhtml/Analytics/Index.php` (trang dashboard), `.../Analytics/Chart.php` (ajax JSON cho chart). Cả hai `ADMIN_RESOURCE = 'Mageplaza_LayeredNavigationUltimate::analytics'`.

### 7.2. Bố cục trang `mplayer_analytics_index.xml`

Từ trên xuống:

0. **Header trang**:
   - Nút **"Clear Report Data"** (confirm dialog, gọi `mplayer/analytics/clearReport` — §6.3, xóa sạch 100% bảng report).
   - Dòng thông báo **"Data updated at <thời điểm watermark>"** — để Admin biết số liệu trễ tối đa 1 giờ theo cron.
1. **Date Range selector** (block + template riêng): preset Hôm nay / Hôm qua / 7 ngày qua / Tháng này / Tùy chỉnh (2 ô date). Default: **7 ngày qua**.
   - Đổi range → JS của block: (a) gọi ajax `mplayer/analytics/chart?from=...&to=...` vẽ lại chart; (b) áp filter `report_date` vào 2 UI listing qua API filter của `uiRegistry` (set `params.filters` trên data source rồi reload) — grid và export tự ăn theo filter này.
2. **Bar chart Top 5 attribute** — `<canvas>` + **Chart.js v4.5.0** có sẵn trong Magento core (`lib/web/chartjs/Chart.min.js`, requirejs alias `chartJs` khai báo ở `module-ui/view/base/requirejs-config.js`; chính `module-backend/.../dashboard/chart.js` đang dùng). Không thêm thư viện ngoài. Data: `{labels: [attr], data: [clicks], percentages: [%]}`.
3. **Grid Top Popular Filters** — UI component `mplayer_analytics_popular_listing` (insert qua `uiComponent` trong layout).
4. **Grid Zero-results** — UI component `mplayer_analytics_zeroresult_listing`.

### 7.3. Data providers — đọc DUY NHẤT bảng report (không đụng bảng raw)

Cả 2 grid đọc chung 1 bảng `mageplaza_layer_analytics_report`, khác nhau ở counter và điều kiện.

**Top Popular Filters** (`hits`):

```sql
SELECT r.attribute_code,
       r.option_value,
       COALESCE(eaov.value, r.option_value) AS option_label,  -- resolve label động
       SUM(r.hits)                           AS total_clicks,
       SUM(r.hits) * 100.0 / (
           SELECT SUM(hits) FROM mageplaza_layer_analytics_report
           WHERE report_date BETWEEN :from AND :to
       ) AS percentage
FROM mageplaza_layer_analytics_report r
LEFT JOIN eav_attribute_option_value eaov
    ON eaov.option_id = r.option_value AND eaov.store_id = 0
WHERE r.report_date BETWEEN :from AND :to
GROUP BY r.attribute_code, r.option_value
ORDER BY total_clicks DESC
```

Cột grid: Attribute Name (resolve label EAV) | Option Label (resolve động) | Total Clicks | Percentage.

**Zero-results** (`zero_hits`):

```sql
SELECT r.attribute_code,
       r.option_value,
       COALESCE(eaov.value, r.option_value) AS option_label,
       SUM(r.zero_hits)     AS total_zero_hits,
       MAX(r.last_requested) AS last_requested
FROM mageplaza_layer_analytics_report r
LEFT JOIN eav_attribute_option_value eaov
    ON eaov.option_id = r.option_value AND eaov.store_id = 0
WHERE r.report_date BETWEEN :from AND :to
GROUP BY r.attribute_code, r.option_value
HAVING total_zero_hits > 0
ORDER BY total_zero_hits DESC
```

Cột grid: Attribute Name | Option Label | Zero Hits | Last Requested.

- *(Price / Category: resolve trong Collection PHP theo logic §4.3 thay vì JOIN trực tiếp trong SQL.)*
- Cả 2 dùng `Magento\Framework\View\Element\UiComponent\DataProvider\DataProvider` + `SearchResult` custom (`Model/ResourceModel/Report/PopularGrid/Collection.php`, `ZeroResultGrid/Collection.php`) khai báo qua `di.xml` — hưởng nguyên sorting/paging/filter/export của UI component.
- Lưu ý kỹ thuật: filter/sort trên cột aggregate (`total_clicks`, `total_zero_hits`) phải map qua `addFieldToFilter` → `HAVING` trong collection custom.

### 7.4. Export

- Cả 2 listing khai báo `exportButton` chuẩn trong `listing_top` → có sẵn **CSV** và **Excel XML**. Export chạy qua `MetadataProvider` của Magento, tôn trọng đúng filter (gồm date range) + sort đang áp.
- File Excel xuất ra đuôi `.xml` (đúng hành vi Magento core, không phải `.xlsx`). Spec gốc ghi `.xlsx` — chấp nhận lệch, built-in đủ dùng cho mục đích analytics.

## 8. Xử lý lỗi & an toàn

- Endpoint track **không bao giờ** ném lỗi ra khách: mọi nhánh sai đều 204; exception nuốt im lặng (không ghi log file riêng).
- Cron Aggregate bọc mỗi batch + watermark trong 1 transaction — crash thì chạy lại từ watermark cũ, số liệu report không bao giờ thiếu hoặc đếm đôi.
- Tracker JS bọc try/catch toàn bộ; lỗi analytics không được phép phá trang catalog.
- Chống rác: giới hạn 20 filter/payload, cắt độ dài mọi field, ép kiểu `result_count`. (Không chống bot chuyên sâu ở v1 — bot thường không chạy JS + sendBeacon; ghi nhận là hạn chế đã biết.)
- Không lưu PII: chỉ `session_id` (định danh phiên, đã có sẵn trong cookie Magento), không IP, không customer_id.

## 9. Danh sách file (tất cả trong `app/code/Mageplaza/LayeredNavigationUltimate/`)

**Mới:**

| File | Vai trò |
|---|---|
| `Controller/Analytics/Track.php` | Endpoint POST nhận beacon |
| `Controller/Adminhtml/Analytics/Index.php` | Trang dashboard |
| `Controller/Adminhtml/Analytics/Chart.php` | Ajax JSON data cho chart |
| `Cron/Aggregate.php` | Cron tổng hợp tăng dần raw → report (watermark) |
| `Cron/CleanLog.php` | Cron dọn raw log (không đụng report) |
| `Controller/Adminhtml/Analytics/ClearReport.php` | Admin chủ động xóa report (nút Clear Report Data) |
| `Console/Command/AggregateAnalytics.php` | CLI `bin/magento mplayer:analytics:aggregate` — chạy aggregate ngay, dùng khi dev/test |
| `Api/AnalyticsWriterInterface.php` | Hợp đồng writer (chỗ cắm QueueWriter tương lai) |
| `Model/Analytics/Logger.php` | Writer v1: INSERT raw append-only |
| `Model/ResourceModel/RequestLog.php` (+ Collection) | Resource model bảng raw |
| `Model/ResourceModel/Report.php` (+ Collection) | Resource model bảng `..._report` |
| `Model/ResourceModel/Report/PopularGrid/Collection.php` | SearchResult grid Top Filters (`hits`, resolve label §4.3) |
| `Model/ResourceModel/Report/ZeroResultGrid/Collection.php` | SearchResult grid Zero-results (`zero_hits`) |
| `ViewModel/AnalyticsState.php` | JSON state cho frontend |
| `Block/Adminhtml/Analytics/DateRange.php` + template | Date range selector + chart |
| `etc/frontend/routes.xml` | Route frontend cho endpoint track |
| `etc/crontab.xml` | Lịch cron |
| `view/frontend/templates/analytics/state.phtml` | In JSON state (trong container layer) |
| `view/frontend/templates/analytics/tracker.phtml` | In thẻ script tracker (before.body.end, ngoài vùng AJAX) |
| `view/frontend/web/js/mp-layer-tracker.js` | Tracker vanilla JS |
| `view/frontend/layout/*` (category, search, products page + bản `hyva_*`) | Gắn block state + tracker |
| `view/adminhtml/layout/mplayer_analytics_index.xml` | Layout dashboard |
| `view/adminhtml/ui_component/mplayer_analytics_popular_listing.xml` | Grid 1 |
| `view/adminhtml/ui_component/mplayer_analytics_zeroresult_listing.xml` | Grid 2 |
| `view/adminhtml/web/js/analytics/dashboard.js` | JS date range + vẽ chart — `define(['jquery', 'chartJs'], …)`, dùng Chart.js v4.5.0 core (không thêm lib ngoài) |
| `view/adminhtml/templates/analytics/dashboard.phtml` | Template: `<canvas>` cho chart + `<script type="text/x-magento-init">` nạp `dashboard.js` |

**Sửa:**

| File | Thay đổi |
|---|---|
| `etc/db_schema.xml` + `etc/db_schema_whitelist.json` | Thêm 2 bảng |
| `etc/adminhtml/system.xml` | Group `analytics` |
| `etc/config.xml` | Defaults |
| `etc/adminhtml/menu.xml` | Node Analytics |
| `etc/acl.xml` | Resource Analytics |
| `etc/di.xml` | Map 2 grid collection vào CollectionFactory |
| `Helper/Data.php` | `isAnalyticsEnabled()`, `getCleanLogDays()` |

**Không sửa module base/pro** — toàn bộ hook qua layout + MutationObserver.

## 10. Chiến lược kiểm thử

| Loại | Nội dung | Verify |
|---|---|---|
| Unit | Resolve label động: EAV / price / category đúng format + fallback khi thiếu option | phpunit |
| Unit | Validate payload: >20 filter, field quá dài, JSON hỏng → bị từ chối êm | phpunit |
| Integration | POST track → N dòng raw đúng store/session, KHÔNG đụng bảng report | phpunit integration |
| Integration | Cron Aggregate: `hits` & `zero_hits` đúng, chạy lặp không đếm đôi (watermark), batch cắt đúng ranh giới request_uid | phpunit integration |
| Integration | CleanLog chỉ xóa raw log đúng ngưỡng ngày và `log_id ≤ watermark`; bảng report còn nguyên | phpunit integration |
| Integration | ClearReport: xóa sạch bảng report, watermark giữ nguyên, raw không bị cộng lại (không đếm đôi) | phpunit integration |
| E2E (Playwright, `magento.ddev.site`, Hyva) | Lọc AJAX → beacon bắn (network tab) → DB có record với result_count đúng | playwright-cli + magerun2 db:query |
| E2E | Vào thẳng URL có filter khi page đã nằm trong FPC → vẫn log (chứng minh R2) | bật FPC, hit 2 lần, check DB |
| E2E | Lọc ra 0 sản phẩm → grid Zero-results hiện đúng filter value với `zero_hits` tăng | admin grid |
| E2E | Reload cùng URL filter (F5) → tính thêm 1 lượt (không chặn F5, không sessionStorage) | check DB count |
| Manual | Tắt Enable → page không còn JSON state/JS; export CSV + Excel XML từ 2 grid | browser |

## 11. Giới hạn đã biết & đường nâng cấp

- Bảng report là dữ liệu dẫn xuất: nếu cần sửa logic tổng hợp về sau, có thể re-aggregate bằng reset watermark về 0 + Clear Report — nhưng chỉ dựng lại được phần raw còn trong cửa sổ retention; phần report lịch sử cũ hơn sẽ mất, cần cân nhắc trước khi làm.
- Số liệu dashboard trễ tối đa ~1 giờ theo chu kỳ cron Aggregate (hiển thị "Data updated at" trên trang).
- Report tích lũy vĩnh viễn (không tự xóa): mỗi (attribute, option)/giờ là 1 dòng — sau nhiều năm bảng vẫn chỉ cỡ trăm nghìn dòng, chấp nhận được; Admin có nút Clear Report Data khi muốn làm gọn.
- **Zero-result ở mức từng filter value, không theo URL/tổ hợp đầy đủ:** chỉ rõ "filter value nào hay rơi vào kết quả rỗng nhất", không truy được chính xác từng URL tổ hợp. Đổi lại schema gọn 1 bảng, không filler. Nếu sau cần đúng URL thì raw log vẫn còn `target_url` để truy ngược.
- Chưa lọc bot chuyên sâu, chưa report theo store view ở UI (data đã có `store_id`, UI v2 thêm store switcher).
- Export Excel là file `.xml` (Excel 2003 XML Spreadsheet — hành vi chuẩn của Magento built-in, không phải `.xlsx`).
- Không lưu label: dashboard resolve động khi query — đổi tên option thì báo cáo tự cập nhật theo tên mới, không vỡ report.
