# 承認決裁システム API 仕様書

**バージョン**: 1.0  
**作成日**: 2026-05-01  
**ベースURL**: `/api`  
**形式**: REST / JSON / UTF-8  
**認証**: Bearer JWT（`Authorization: Bearer <token>`）

---

## 1. 共通仕様

### 1.1 ヘッダ

| ヘッダ | 必須 | 内容 |
|---|:-:|---|
| `Authorization` | ✅（auth系除く） | `Bearer <jwt>` |
| `Content-Type` | ✅（POST/PUT） | `application/json` |
| `Accept-Language` | - | `ja-JP`（既定） |

### 1.2 共通レスポンス

成功:
```json
{ "data": { ... } }
```

エラー（RFC 7807 ProblemDetails 拡張）:
```json
{
  "type": "https://approval-system/errors/validation",
  "title": "バリデーションエラー",
  "status": 400,
  "detail": "取得時刻は15分単位で指定してください",
  "errors": { "timeFrom": ["15分単位で指定してください"] }
}
```

### 1.3 ステータスコード

| コード | 用途 |
|---|---|
| 200 | 取得・更新成功 |
| 201 | 作成成功（`Location` ヘッダで新リソースURL） |
| 204 | 更新・削除成功（ボディなし） |
| 400 | バリデーションエラー |
| 401 | 未認証 / トークン期限切れ |
| 403 | 権限不足 / テナント不一致 |
| 404 | 対象なし |
| 409 | 状態遷移不正 |
| 500 | サーバ内部エラー |

### 1.4 ページング・ソート共通

```
GET /api/applications?page=1&pageSize=20&sort=submittedAt:desc&status=in_progress
```

レスポンス:
```json
{
  "data": [ ... ],
  "pagination": { "page": 1, "pageSize": 20, "total": 87, "totalPages": 5 }
}
```

### 1.5 JWT クレーム

| クレーム | 内容 |
|---|---|
| `sub` | ユーザーID |
| `email` | メール |
| `tenant_id` | テナントID（システム管理者は無し） |
| `roles` | カンマ区切り (`Applicant,Approver`) |
| `is_system_admin` | true/false |
| `exp` | 失効時刻 |

---

## 2. 認証

### 2.1 ログイン

```
POST /api/auth/login
```

リクエスト:
```json
{
  "tenantCode": "demo",     // システム管理者は省略可
  "email": "user@demo",
  "password": "********"
}
```

レスポンス 200:
```json
{
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "expiresInSeconds": 3600,
    "user": {
      "id": "uuid", "displayName": "山田 太郎",
      "roles": ["Applicant"], "tenantId": "uuid"
    }
  }
}
```

エラー:
- 401: 認証失敗
- 403: テナント停止中

### 2.2 リフレッシュ

```
POST /api/auth/refresh
{ "refreshToken": "..." }
```

### 2.3 ログアウト

```
POST /api/auth/logout
```

---

## 3. 申請（休暇）

### 3.1 一覧取得

```
GET /api/applications?status=&applicantId=&dateFrom=&dateTo=&page=&pageSize=
```

応答 `data[]`:
```json
{
  "id": "uuid",
  "applicationNumber": "2026-05-0001",
  "applicantUserId": "uuid",
  "applicantDisplayName": "山田 太郎",
  "status": "in_progress",
  "currentStepOrder": 2,
  "leave": {
    "leaveType": "annual_paid_leave",
    "takeUnit": "hour",
    "dateFrom": "2026-05-10", "dateTo": "2026-05-10",
    "timeFrom": "09:00", "timeTo": "11:00",
    "totalMinutes": 120, "totalDays": 0.26
  },
  "submittedAt": "2026-05-01T10:00:00Z"
}
```

### 3.2 休暇申請作成

```
POST /api/applications/leave
```

リクエスト:
```json
{
  "leaveType": "annual_paid_leave",
  "takeUnit": "hour",
  "dateFrom": "2026-05-10",
  "dateTo": "2026-05-10",
  "timeFrom": "09:00",
  "timeTo": "11:00",
  "reason": "私用",
  "remarks": ""
}
```

応答 201: `{ "data": { "id": "uuid" } }`

バリデーション:
- 時間取得時: `dateFrom == dateTo`
- 時刻は 15 分単位（00,15,30,45）
- `timeFrom < timeTo`

### 3.3 詳細取得

```
GET /api/applications/{id}
```

応答:
```json
{
  "data": {
    "id": "uuid", "applicationNumber": "2026-05-0001",
    "status": "in_progress", "currentStepOrder": 2,
    "applicant": { "id": "uuid", "displayName": "山田 太郎" },
    "leave": { ... },
    "route": [
      { "stepOrder": 1, "stepType": "approval", "assignee": {...}, "status": "approved", "actionAt": "..." },
      { "stepOrder": 2, "stepType": "approval", "assignee": {...}, "status": "pending" },
      { "stepOrder": 3, "stepType": "decision", "assignee": {...}, "status": "pending" }
    ],
    "logs": [ ... ],
    "parentApplicationId": null
  }
}
```

### 3.4 更新

```
PUT /api/applications/{id}
```

許可条件: `status=draft`（申請者）/ 任意（時間管理者・システム管理者）。

### 3.5 取り消し

```
DELETE /api/applications/{id}
```

許可: 申請者（`in_progress` 中）/ 時間管理者 / システム管理者。

### 3.6 提出

```
POST /api/applications/{id}/submit
```

応答 200:
```json
{ "data": { "applicationNumber": "2026-05-0042", "status": "in_progress" } }
```

---

## 4. 承認・決裁・クローズ

### 4.1 承認・合議

```
POST /api/applications/{id}/approve
{ "comment": "問題なし" }
```

権限: 当該ステップ担当者 または 時間管理者（代行）。

### 4.2 差し戻し

```
POST /api/applications/{id}/reject
{ "comment": "理由が不明確です" }    // 必須
```

応答 200:
```json
{ "data": { "newApplicationId": "uuid", "newApplicationNumber": "2026-05-0043" } }
```

### 4.3 決裁

```
POST /api/applications/{id}/decide
{ "comment": "" }
```

成功時、`status = decided`。テナントの時間管理者全員＋申請者に通知。

### 4.4 クローズ

```
POST /api/applications/{id}/close
```

権限: 時間管理者 / システム管理者。`status = closed` に遷移。

応答 200:
```json
{ "data": { "id": "uuid", "status": "closed", "closedAt": "..." } }
```

---

## 5. 残日数

### 5.1 取得

```
GET /api/leave-balance/{userId}?asOf=2026-05-01
```

応答:
```json
{
  "data": {
    "userId": "uuid",
    "asOf": "2026-05-01",
    "fiscalYear": 2026,
    "carriedOverDays": 12.0,
    "grantedDaysYTD": 4.0,
    "usedDaysYTD": 1.5,
    "remainingDays": 14.5,
    "remainingMinutes": 6735
  }
}
```

### 5.2 修正

```
PUT /api/leave-balance/{userId}
{ "balanceYear": 2026, "balanceMonth": 4, "carryOverDays": 12.0, "grantedDays": 2.0 }
```

権限: 時間管理者 / システム管理者。

---

## 6. マスタ管理（テナント内）

### 6.1 ユーザー

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/api/users` | 一覧 |
| POST | `/api/users` | 作成 |
| GET | `/api/users/{id}` | 詳細 |
| PUT | `/api/users/{id}` | 更新 |
| DELETE | `/api/users/{id}` | 無効化 |

POST ボディ:
```json
{
  "nameLast": "山田", "nameFirst": "太郎",
  "displayName": "山田 太郎",
  "email": "yamada@demo", "department": "営業部",
  "roles": ["Applicant"], "password": "initial-pass"
}
```

### 6.2 決裁ルート

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/api/approval-routes` | 一覧 |
| POST | `/api/approval-routes` | 作成 |
| GET | `/api/approval-routes/{id}` | 詳細 |
| PUT | `/api/approval-routes/{id}` | 更新（ステップ含む） |
| DELETE | `/api/approval-routes/{id}` | 削除 |

POST ボディ:
```json
{
  "name": "山田 太郎 通常ルート",
  "applicantUserId": "uuid",
  "steps": [
    { "stepOrder": 1, "stepType": "approval", "assigneeUserId": "uuid" },
    { "stepOrder": 2, "stepType": "decision", "assigneeUserId": "uuid" }
  ]
}
```

### 6.3 メールテンプレート

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/api/email-templates` | 一覧（テナント＋ルート別） |
| GET | `/api/email-templates/{id}` | 詳細 |
| PUT | `/api/email-templates/{id}` | 更新 |
| POST | `/api/email-templates` | 新規（ルート別作成時） |

ボディ:
```json
{
  "routeId": null,
  "eventType": "submitted",
  "subjectTemplate": "【承認依頼】{{ application_type }} - {{ applicant_name }}",
  "bodyTemplate": "...",
  "isHtml": false
}
```

### 6.4 システム設定

```
GET  /api/system-settings
PUT  /api/system-settings
```

ボディ:
```json
{
  "monthlyGrantDays": 2,
  "maxCarryoverDays": 30,
  "workMinutesPerDay": 465,
  "fiscalYearStartMonth": 4
}
```

### 6.5 メール送信ログ

```
GET  /api/email-logs?status=&dateFrom=&dateTo=
POST /api/email-logs/{id}/resend
```

---

## 7. テナント管理（システム管理者専用）

すべて `/api/admin/` プレフィックス。

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/api/admin/tenants` | 一覧 |
| POST | `/api/admin/tenants` | 作成 |
| GET | `/api/admin/tenants/{id}` | 詳細 |
| PUT | `/api/admin/tenants/{id}` | 更新 |
| DELETE | `/api/admin/tenants/{id}` | 無効化 |
| GET | `/api/admin/tenants/{id}/stats` | 利用統計 |

POST ボディ:
```json
{
  "code": "acme",
  "name": "株式会社ACME",
  "initialAdmin": {
    "email": "admin@acme",
    "displayName": "管理者",
    "password": "init-pass"
  }
}
```

統計レスポンス:
```json
{
  "data": {
    "tenantId": "uuid",
    "userCount": 42,
    "applicationCount": { "total": 320, "inProgress": 5, "decided": 3, "closed": 290 },
    "lastActivityAt": "..."
  }
}
```

---

## 8. 統計・印刷

### 8.1 休暇統計

```
GET /api/statistics/leave?period=fiscal&year=2026&userId=&department=
```

応答:
```json
{
  "data": {
    "period": { "from": "2026-04-01", "to": "2027-03-31" },
    "byMonth": [ { "month": "2026-04", "totalDays": 12.5 }, ... ],
    "byUser": [ { "userId": "uuid", "displayName": "...", "usedDays": 3.5, "remainingDays": 14.0 } ],
    "byLeaveType": [ { "leaveType": "annual_paid_leave", "totalDays": 120.0 } ],
    "pendingApprovalCount": 5,
    "pendingCloseCount": 2
  }
}
```

### 8.2 印刷データ取得

```
GET /api/print/list?period=fiscal&year=2026&...      // 一覧表用
GET /api/print/form/{applicationId}                   // 定型用紙用
```

応答は印刷用に整形されたデータ（HTML レンダリングはフロント側で実施）。  
PDF が必要な場合は `?format=pdf` で `application/pdf` を返す（QuestPDF 使用）。

---

## 9. エラーコード一覧（抜粋）

| code | HTTP | 意味 |
|---|---|---|
| `validation` | 400 | バリデーションエラー |
| `unauthorized` | 401 | 未認証 |
| `forbidden` | 403 | 権限不足 |
| `tenant_mismatch` | 403 | テナント不一致 |
| `not_found` | 404 | 対象なし |
| `invalid_state` | 409 | 状態遷移不正（例: closed を更に承認） |
| `duplicate` | 409 | ユニーク制約違反 |
| `internal` | 500 | サーバ内部 |

---

## 10. レート制限・冪等性

- レート制限: ログインのみ 10req/分/IP（429 で返却）
- 冪等性: `POST /approve|reject|decide|close` は同一申請に対する二重実行を 409 で拒否（`is_completed` チェック）

---

*OpenAPI/Swagger スキーマは実装時に自動生成し `/swagger` で公開する。*
