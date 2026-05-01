# 承認決裁システム 設計方針書（Architecture）

**バージョン**: 1.0  
**作成日**: 2026-05-01  
**対象**: 承認決裁システム（マルチテナント・休暇申請ワークフロー）

---

## 1. 設計方針の概要

本システムは、**マルチテナント型の汎用承認決裁基盤**として、以下の設計原則に従う。

| 原則 | 内容 |
|---|---|
| **関心の分離** | UI（Next.js）/ API（ASP.NET Core）/ ドメイン / インフラを分離する |
| **テナント分離** | 全テーブルに `tenant_id` を持たせ、ミドルウェアで強制的にスコープを掛ける |
| **拡張容易性** | 認証プロバイダー・申請種別・DB プロバイダーを抽象化し差し替え可能にする |
| **ステートレスAPI** | API は JWT に基づくステートレス設計とし、水平スケールを可能とする |
| **監査性** | 全状態遷移は `ApprovalStepLogs` に記録し、ソフトデリートで完全な履歴を保持 |

---

## 2. アーキテクチャスタイル

### 2.1 全体構成

```
┌─────────────────────────────────────────────────────────────┐
│                      Browser (Next.js 14)                    │
│           - App Router / Server Components                   │
│           - shadcn/ui or Ant Design                          │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTPS / REST + JWT
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              ASP.NET Core 7 Web API                          │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Middleware Pipeline                                │     │
│  │   1. ExceptionHandler                               │     │
│  │   2. Authentication (JWT)                           │     │
│  │   3. TenantResolver  ← tenant_id クレーム抽出        │     │
│  │   4. Authorization (RBAC + TenantScope)             │     │
│  └────────────────────────────────────────────────────┘     │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Controllers (Presentation)                         │     │
│  │  ApplicationServices (UseCase / Orchestration)      │     │
│  │  Domain (Entity / ValueObject / DomainService)      │     │
│  │  Infrastructure (EF Core / SMTP / Auth Provider)    │     │
│  └────────────────────────────────────────────────────┘     │
└────────────────────┬────────────────────────────────────────┘
                     │
       ┌─────────────┼─────────────────┐
       ▼             ▼                 ▼
┌────────────┐ ┌────────────┐ ┌─────────────────┐
│ EF InMemory│ │  MySQL 8   │ │ SMTP / MailKit  │
│ (dev/test) │ │ (production)│ │ (非同期送信)    │
└────────────┘ └────────────┘ └─────────────────┘
```

### 2.2 レイヤ構成（バックエンド）

| レイヤ | 役割 | 例 |
|---|---|---|
| **Presentation** | HTTP I/O・DTO 変換・認可属性 | `ApplicationsController` |
| **Application** | ユースケース調整・トランザクション境界 | `SubmitApplicationService` |
| **Domain** | ビジネスルール・エンティティ・値オブジェクト | `Application`, `LeaveBalance` |
| **Infrastructure** | EF Core 実装・SMTP・JWT・外部I/O | `ApplicationDbContext`, `SmtpEmailSender` |

依存方向: Presentation → Application → Domain ← Infrastructure（DI で逆転）

### 2.3 フロントエンド構成（Next.js 14）

```
app/
├ (auth)/login/page.tsx
├ (main)/
│  ├ dashboard/page.tsx
│  ├ applications/[id]/page.tsx
│  └ admin/tenants/page.tsx
├ api/                ← BFF 用（必要に応じて）
components/           ← UI コンポーネント
lib/
├ api-client.ts       ← Axios + JWT 自動付与
├ auth.ts             ← トークン管理
└ tenant-context.tsx  ← テナント情報 Context
```

---

## 3. マルチテナント設計

### 3.1 分離方式

**シングルDB / シェアードスキーマ + tenant_id カラム方式** を採用する。

| 比較項目 | DB 分離 | スキーマ分離 | カラム分離（採用） |
|---|---|---|---|
| 運用コスト | 高 | 中 | 低 |
| データ集計（システム管理者用） | 困難 | 困難 | 容易 |
| 漏洩リスク | 低 | 中 | 中（ミドルウェアで強制） |

**漏洩リスクへの対策**:
- 全クエリに自動で `tenant_id` フィルタを適用する EF Core グローバルクエリフィルタを実装
- `TenantContext` を `IHttpContextAccessor` 経由で DI し、リポジトリは常にテナントスコープで動作

### 3.2 テナント解決フロー

```
1. クライアント: Login → tenant_code + email + password
2. サーバ:      認証成功 → JWT に tenant_id, user_id, roles を含めて発行
3. クライアント: 以降のリクエストに Authorization: Bearer <jwt>
4. サーバ:      TenantResolver Middleware が tenant_id を抽出
                → ITenantContext に設定
                → DbContext のグローバルフィルタが自動適用
```

### 3.3 システム管理者の扱い

- システム管理者ユーザーは `users.tenant_id = NULL`
- JWT に `is_system_admin: true` クレームを含める
- システム管理者リクエストは `[AllowCrossTenant]` 属性付きエンドポイントでのみグローバルクエリフィルタを無効化

---

## 4. 認証・認可方針

### 4.1 認証

- **独自JWT認証** を初期実装。将来 AD/LDAP に切り替え可能とするため、抽象化する。

```csharp
public interface IAuthProvider
{
    Task<AuthResult> AuthenticateAsync(string tenantCode, string identifier, string credential);
    Task<UserPrincipal?> ResolveUserAsync(string identifier);
}

public class LocalDbAuthProvider : IAuthProvider { /* bcrypt + DB 検証 */ }
public class LdapAuthProvider : IAuthProvider { /* 将来実装 */ }
```

| 項目 | 設定 |
|---|---|
| アルゴリズム | HS256（小規模オンプレ向け）/ RS256 推奨 |
| 有効期限 | アクセストークン 60 分 / リフレッシュトークン 14 日 |
| パスワード保存 | bcrypt (work factor 12) |
| トークン格納 | HttpOnly Cookie 推奨。SPA 同一オリジン構成の場合は localStorage 可（XSS 対策必須） |

### 4.2 認可（RBAC + テナントスコープ）

```csharp
[Authorize(Roles = "TimeManager,SystemAdmin")]
[TenantScoped]                               // テナント一致を強制
public async Task<IActionResult> Close(Guid id) { ... }

[Authorize(Roles = "SystemAdmin")]
[AllowCrossTenant]                           // システム管理者のみ
public async Task<IActionResult> ListTenants() { ... }
```

ロール一覧:

| ロール識別子 | 名称 |
|---|---|
| `SystemAdmin` | システム管理者 |
| `TimeManager` | 時間管理者 |
| `Applicant` | 申請者 |
| `Consultor` | 合議者 |
| `Approver` | 承認者 |
| `Decider` | 決裁者 |

---

## 5. データアクセス方針

### 5.1 DB プロバイダー切り替え

`appsettings.json`:
```json
{
  "DatabaseProvider": "InMemory",   // "InMemory" | "MySQL"
  "ConnectionStrings": {
    "MySQL": "Server=...;Database=approval;..."
  }
}
```

`Program.cs`:
```csharp
var provider = builder.Configuration["DatabaseProvider"];
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    if (provider == "MySQL")
        opt.UseMySql(connStr, ServerVersion.AutoDetect(connStr));
    else
        opt.UseInMemoryDatabase("approval-dev");
});
```

### 5.2 グローバルクエリフィルタ

```csharp
modelBuilder.Entity<Application>()
    .HasQueryFilter(a =>
        a.TenantId == _tenantContext.TenantId &&
        !a.IsDeleted);
```

ソフトデリート列とテナントを同時にフィルタすることで、ハンドコードでの漏れを防ぐ。

### 5.3 トランザクション

- ユースケース 1 件 = 1 トランザクション（`IUnitOfWork` パターン）
- ステータス遷移とログ記録は同一トランザクション内で実施

---

## 6. ワークフロー設計（ステートマシン）

```
        submit
draft ─────────► in_progress(step=1)
                    │
                    │ approve
                    ▼
               in_progress(step=N+1)
                    │
                    │ decide (最終ステップ)
                    ▼
                 decided
                    │ close (時間管理者)
                    ▼
                 closed

任意ステップから:
  reject  → rejected (soft-delete) + 新 application を draft で生成
  cancel  → cancelled (soft-delete)
```

`IApplicationStateMachine` を提供し、許可されない遷移は `InvalidOperationException` を投げる。

---

## 7. 通知（メール）方針

| 項目 | 方針 |
|---|---|
| ライブラリ | MailKit |
| 実行モデル | バックグラウンドキュー（`Channel<EmailJob>` + Hosted Service） |
| 失敗時 | `EmailLogs` に `failed` で記録、管理画面から再送 |
| テンプレート | DB 保存。Scriban でプレースホルダー解決 |

---

## 8. ロギング・監査

- アプリケーションログ: Serilog（JSON 構造化）
- 監査ログ: `ApprovalStepLogs` に全ステータス変化を記録（誰が・いつ・代行か）
- メール送信ログ: `EmailLogs`

---

## 9. デプロイ・環境

| 環境 | DB | 認証 | 用途 |
|---|---|---|---|
| local-dev | InMemory | LocalDb | 開発・単体テスト |
| staging | MySQL | LocalDb | 結合テスト |
| production | MySQL | LocalDb / LDAP | 本番 |

CI/CD: GitHub Actions（lint → test → build → deploy）

---

## 10. 拡張ポイント

| 拡張対象 | 抽象化インターフェイス |
|---|---|
| 認証 | `IAuthProvider` |
| 申請種別 | `IApplicationTypeHandler`（休暇 / 指導受け / 汎用） |
| メール送信 | `IEmailSender`（SMTP / SendGrid / SES） |
| 印刷 | `IPrintRenderer`（HTML/CSS / QuestPDF） |

---

*本書は要件定義書 v1.2 に基づく設計方針を定めるものであり、詳細仕様は `basic-design.md` および `detail-design.md` を参照のこと。*
