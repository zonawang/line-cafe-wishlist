# LINE Cafe Wishlist

讓 Zona Cafe Bot 把喜歡但還沒決定造訪的店先存進個人想去清單，再從收藏直接安排時間：

```text
附近推薦 → 加入想去清單 → 查看收藏 → 安排時間
                                      ↓
                              LINE 主動提醒 → 咖啡足跡
```

這個 repo 延續 [`line-cafe-follow-up`](https://github.com/zonawang/line-cafe-follow-up) 的 Google Maps Grounding、偏好記憶、Datetime Picker、咖啡足跡與主動回訪功能。

## 目前功能

- 推薦卡片可將指定咖啡廳加入個人想去清單
- 重複收藏同一個 Google Maps 地點不會產生重複資料
- 輸入「我的想去清單」、「想去清單」或「我的收藏」查看最近 10 間收藏
- 收藏卡片可開啟 Google Maps、安排造訪時間或移出清單
- 從收藏安排時間後，沿用 Google Calendar 與造訪後提醒流程
- 傳送位置後，以 Gemini + Google Maps Grounding 推薦附近咖啡廳
- 個人咖啡偏好、換一批及更適合工作
- Datetime Picker 與 Google Calendar 預填連結
- 選定時間後，在 Firestore 建立 planned visit
- 使用 Cloud Tasks 安排造訪後提醒，預設在行程開始一小時後送出
- LINE Push 提供「開始評分」與「這次沒去」
- 評分沿用既有的 1～5 分、體驗標籤及咖啡足跡流程
- 足跡會保留原本安排的造訪日期，不會誤用填寫評分的時間
- 固定 Task ID、Firestore delivery lease 與狀態檢查降低重複提醒
- 提醒操作綁定 LINE 使用者與 conversation
- 內部 reminder endpoint 使用長密鑰驗證

## 資料狀態

Planned visit 儲存在 `cafe-planned-visits/{plannedVisitId}`：

```text
scheduled → sending → reminded → feedback_started
                         └──────→ canceled
```

`sending` 有五分鐘 delivery lease。LINE Push 失敗時會釋放回 `scheduled`，讓 Cloud Tasks 重試；已完成狀態再次收到相同 task 時會直接回傳成功。

## 本機設定

需求：Node.js 20 以上、Google Cloud 專案、Firestore Native mode、Cloud Tasks API 與 LINE Messaging API channel。

```bash
cp .env.example .env
npm install
npm run dev
```

必要環境變數：

```env
PORT=3000
LINE_CHANNEL_SECRET=...
LINE_CHANNEL_ACCESS_TOKEN=...
GOOGLE_CLOUD_PROJECT=line-zona
GOOGLE_CLOUD_LOCATION=global
GEMINI_MAPS_MODEL=gemini-2.5-flash
GEMINI_TRANSLATION_MODEL=gemini-2.5-flash
GEMINI_FUNCTION_MODEL=gemini-2.5-flash
FIRESTORE_SESSION_COLLECTION=cafe-search-sessions
FIRESTORE_PREFERENCES_COLLECTION=cafe-user-preferences
FIRESTORE_PREFERENCE_ACTIONS_COLLECTION=cafe-preference-actions
FIRESTORE_JOURNEY_USERS_COLLECTION=cafe-user-journeys
FIRESTORE_PLANNED_VISITS_COLLECTION=cafe-planned-visits
FIRESTORE_WISHLIST_USERS_COLLECTION=cafe-user-wishlists
CLOUD_TASKS_LOCATION=asia-east1
CLOUD_TASKS_QUEUE=cafe-follow-up-reminders
REMINDER_CALLBACK_URL=https://your-service.run.app/tasks/visit-reminders
REMINDER_TASK_SECRET=<至少 24 字元的隨機值>
FOLLOW_UP_DELAY_MINUTES=60
```

本機要呼叫 Vertex AI、Firestore 或 Cloud Tasks 時，先設定 Application Default Credentials：

```bash
gcloud auth application-default login
```

## 驗證

```bash
npm run typecheck
npm test
```

CI 會在 push 與 pull request 自動執行相同檢查。

## Rich Menu

`assets/rich-menu-v4-clean.json` 與 `assets/rich-menu-v4-clean.png` 是目前五格選單的設定與圖片，可編輯來源保存在 `assets/rich-menu-v4-clean.svg`。新版移除舊圖的滿版圓點紋理，改用乾淨的純色卡片；下排中央的「想去清單」會送出「我的想去清單」，直接進入 wishlist 流程。

## 建立 Cloud Tasks queue

```bash
gcloud services enable cloudtasks.googleapis.com

gcloud tasks queues create cafe-follow-up-reminders \
  --location asia-east1 \
  --max-attempts 5 \
  --min-backoff 30s \
  --max-backoff 10m
```

產生 reminder endpoint 使用的密鑰：

```bash
openssl rand -hex 32
```

把結果放進 Cloud Run 的 `REMINDER_TASK_SECRET`；不要 commit 真實密鑰。

## Cloud Run 部署

Runtime service account 至少需要：

- `roles/aiplatform.user`
- `roles/datastore.user`
- `roles/cloudtasks.enqueuer`
- `roles/serviceusage.serviceUsageConsumer`

第一次部署可以先將 `REMINDER_CALLBACK_URL` 設成合法的暫時 HTTPS URL：

```bash
gcloud run deploy line-cafe-wishlist \
  --source . \
  --region asia-east1 \
  --allow-unauthenticated \
  --no-cpu-throttling \
  --service-account line-cafe-wishlist@line-zona.iam.gserviceaccount.com \
  --env-vars-file cloud-run-env.yaml
```

取得 service URL 後，把 `REMINDER_CALLBACK_URL` 更新為：

```text
https://<service-url>/tasks/visit-reminders
```

重新部署並先驗證：

```bash
curl https://<service-url>/health
```

最後才將 LINE webhook 更新為：

```text
https://<service-url>/webhook
```

## Firestore TTL

Planned visit 會寫入 `expiresAt`，預設在提醒時間 90 天後清除。可啟用 TTL：

```bash
gcloud firestore fields ttls update expiresAt \
  --collection-group=cafe-planned-visits \
  --enable-ttl
```

其他既有暫存集合也應各自設定 TTL。

## 想去清單資料

收藏儲存在 `cafe-user-wishlists/{ownerId}/entries/{wishlistItemId}`。ID 由 Google Maps URI 雜湊產生，因此同一位使用者重複收藏同一地點時只會更新既有項目，不會建立重複卡片。

## 安全邊界

- `/webhook` 使用 LINE channel secret 驗證簽章。
- `/tasks/visit-reminders` 只接受正確的 `X-Cafe-Reminder-Secret`。
- Reminder body 只帶 planned visit ID；使用者、聊天室與店家資料從 Firestore 取得。
- Follow-up Postback 僅接受白名單 action 與安全 ID。
- 評分與略過操作會再次驗證 LINE `userId` 及 conversation。
- 收藏、移除及從收藏安排時間都會再次驗證 LINE `userId`；推薦卡收藏還會驗證原始 search session 與 conversation。
- 密鑰不得放入 repo、Cloud Tasks body 或記錄訊息。
