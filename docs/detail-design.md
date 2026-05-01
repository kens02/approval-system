# 承認決裁システム 詳細設計書

**バージョン**: 1.0  
**作成日**: 2026-05-01

---

## 1. 本書の位置づけ

`basic-design.md` の内容を実装可能なクラス・メソッド・処理シーケンスレベルで詳細化する。

---

## 2. プロジェクト構成

### 2.1 ソリューション

```
ApprovalSystem.sln
├ src/
│  ├ ApprovalSystem.Api               (ASP.NET Core 7 Web API)
│  ├ ApprovalSystem.Application       (UseCase / DTO)
│  ├ ApprovalSystem.Domain            (Entity / Enum / DomainService)
│  ├ ApprovalSystem.Infrastructure    (EF Core / SMTP / Auth)
│  └ ApprovalSystem.Web               (Next.js 14)
├ tests/
│  ├ ApprovalSystem.UnitTests
│  ├ ApprovalSystem.IntegrationTests
│  └ ApprovalSystem.Web.E2E
└ docs/
```

### 2.2 主要 NuGet

| パッケージ | バージョン | 用途 |
|---|---|---|
| Microsoft.EntityFrameworkCore | 7.x | ORM |
| Pomelo.EntityFrameworkCore.MySql | 7.x | MySQL Provider |
| Microsoft.EntityFrameworkCore.InMemory | 7.x | 開発用 |
| Microsoft.AspNetCore.Authentication.JwtBearer | 7.x | JWT |
| BCrypt.Net-Next | 4.x | パスワードハッシュ |
| MailKit | 4.x | SMTP |
| Scriban | 5.x | テンプレート |
| Serilog.AspNetCore | 7.x | ログ |
| FluentValidation.AspNetCore | 11.x | バリデーション |
| QuestPDF | 2024.x | PDF 出力 |

---

## 3. ドメインモデル詳細

### 3.1 エンティティ（C#）

```csharp
public class Tenant
{
    public Guid Id { get; private set; }
    public string Name { get; set; } = "";
    public string Code { get; set; } = "";    // ログイン時の組織コード
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}

public class User
{
    public Guid Id { get; private set; }
    public Guid? TenantId { get; set; }       // SystemAdmin は null
    public string NameLast { get; set; } = "";
    public string NameFirst { get; set; } = "";
    public string DisplayName { get; set; } = "";
    public string Email { get; set; } = "";
    public string? Department { get; set; }
    public UserRole Roles { get; set; }       // [Flags]
    public string PasswordHash { get; set; } = "";
    public bool IsActive { get; set; } = true;
}

[Flags]
public enum UserRole
{
    None        = 0,
    Applicant   = 1 << 0,
    Consultor   = 1 << 1,
    Approver    = 1 << 2,
    Decider     = 1 << 3,
    TimeManager = 1 << 4,
    SystemAdmin = 1 << 5,
}

public class Application
{
    public Guid Id { get; private set; }
    public Guid TenantId { get; set; }
    public string ApplicationNumber { get; set; } = "";  // yyyy-mm-NNNN
    public Guid ApplicantUserId { get; set; }
    public ApplicationType ApplicationType { get; set; }
    public ApplicationStatus Status { get; set; }
    public int CurrentStepOrder { get; set; }
    public Guid? ParentApplicationId { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? SubmittedAt { get; set; }
    public DateTime? DecidedAt { get; set; }
    public DateTime? ClosedAt { get; set; }
    public Guid? ClosedByUserId { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}

public enum ApplicationStatus { Draft, InProgress, Decided, Closed, Rejected, Cancelled }
public enum ApplicationType   { LeaveAnnualPaid /* future: Guidance, Generic */ }

public class LeaveApplication
{
    public Guid Id { get; private set; }
    public Guid TenantId { get; set; }
    public Guid ApplicationId { get; set; }
    public LeaveType LeaveType { get; set; }      // AnnualPaidLeave
    public TakeUnit TakeUnit { get; set; }        // Hour | Day
    public DateOnly DateFrom { get; set; }
    public DateOnly DateTo { get; set; }
    public TimeOnly? TimeFrom { get; set; }
    public TimeOnly? TimeTo { get; set; }
    public int TotalMinutes { get; set; }
    public decimal TotalDays { get; set; }
    public string? Reason { get; set; }
    public string? Remarks { get; set; }
}

public class ApprovalStepLog
{
    public Guid Id { get; private set; }
    public Guid TenantId { get; set; }
    public Guid ApplicationId { get; set; }
    public int StepOrder { get; set; }
    public StepType StepType { get; set; }
    public Guid AssigneeUserId { get; set; }
    public StepAction Action { get; set; }
    public bool IsProxy { get; set; }
    public DateTime ActionAt { get; set; }
    public string? Comment { get; set; }
    public bool IsCompleted { get; set; }
}

public enum StepType   { Consulted, Approval, Decision }
public enum StepAction { Approved, Rejected, Decided, Closed, Cancelled }
```

### 3.2 ドメインサービス

```csharp
public interface IApplicationStateMachine
{
    void Submit(Application app);
    void Approve(Application app, User actor, bool isProxy);
    void Decide(Application app, User actor, bool isProxy);
    void Reject(Application app, User actor, string comment, bool isProxy);
    void Cancel(Application app, User actor);
    void Close(Application app, User actor);
}
```

不正遷移は `DomainException` を投げる。

---

## 4. ユースケース層（Application Services）

### 4.1 申請提出ユースケース

```csharp
public class SubmitApplicationUseCase
{
    private readonly IAppDbContext _db;
    private readonly IApplicationStateMachine _sm;
    private readonly INotificationDispatcher _notify;
    private readonly ITenantContext _tenant;

    public async Task<SubmitResult> ExecuteAsync(Guid applicationId, Guid actorId, CancellationToken ct)
    {
        var app = await _db.Applications.FirstOrThrowAsync(a => a.Id == applicationId, ct);
        AssertCanSubmit(app, actorId);

        _sm.Submit(app);                              // status: draft → in_progress (step=1)
        app.SubmittedAt = DateTime.UtcNow;

        var step1 = await _db.ApprovalRouteSteps
            .Where(s => s.RouteId == ResolveRouteId(app) && s.StepOrder == 1)
            .FirstAsync(ct);

        await _db.SaveChangesAsync(ct);
        await _notify.NotifyAsync(EventType.Submitted, app, recipients: new[] { step1.AssigneeUserId }, ct);
        return SubmitResult.Ok(app.ApplicationNumber);
    }
}
```

### 4.2 承認・差し戻し・決裁・クローズ

各ユースケースは以下の共通構造:

1. ロード（テナントスコープは EF Global Filter で自動）
2. 認可チェック（担当ステップ or 時間管理者の代行）
3. 状態遷移（StateMachine）
4. ログ記録（ApprovalStepLog 追加）
5. 通知発火

### 4.3 差し戻しユースケース（複製ロジック）

```csharp
public async Task<RejectResult> ExecuteAsync(Guid appId, Guid actorId, string comment, bool isProxy, CancellationToken ct)
{
    if (string.IsNullOrWhiteSpace(comment))
        throw new ValidationException("差し戻しコメントは必須です");

    var current = await _db.Applications.FirstAsync(a => a.Id == appId, ct);
    var actor = await _db.Users.FirstAsync(u => u.Id == actorId, ct);

    _sm.Reject(current, actor, comment, isProxy);   // status=rejected, is_deleted=true

    // 申請内容コピー
    var leave = await _db.LeaveApplications.FirstAsync(l => l.ApplicationId == current.Id, ct);
    var newApp = new Application {
        TenantId = current.TenantId,
        ApplicantUserId = current.ApplicantUserId,
        ApplicationType = current.ApplicationType,
        Status = ApplicationStatus.Draft,
        ParentApplicationId = current.Id,
        ApplicationNumber = await _numberer.NextAsync(current.TenantId, DateTime.UtcNow, ct),
    };
    _db.Applications.Add(newApp);
    _db.LeaveApplications.Add(CloneLeave(leave, newApp.Id));

    _db.ApprovalStepLogs.Add(new ApprovalStepLog {
        ApplicationId = current.Id, StepOrder = current.CurrentStepOrder,
        AssigneeUserId = actorId, Action = StepAction.Rejected,
        Comment = comment, IsProxy = isProxy, IsCompleted = true,
        ActionAt = DateTime.UtcNow,
    });

    await _db.SaveChangesAsync(ct);
    await _notify.NotifyAsync(EventType.Rejected, current,
        recipients: new[] { current.ApplicantUserId },
        extra: new { NewApplicationId = newApp.Id, Comment = comment }, ct);
    return RejectResult.Ok(newApp.Id);
}
```

### 4.4 クローズユースケース

```csharp
public async Task ExecuteAsync(Guid appId, Guid actorId, CancellationToken ct)
{
    var actor = await _db.Users.FirstAsync(u => u.Id == actorId, ct);
    if (!actor.Roles.HasFlag(UserRole.TimeManager) && !actor.Roles.HasFlag(UserRole.SystemAdmin))
        throw new ForbiddenException();

    var app = await _db.Applications.FirstAsync(a => a.Id == appId, ct);
    _sm.Close(app, actor);                       // decided → closed
    app.ClosedAt = DateTime.UtcNow;
    app.ClosedByUserId = actorId;

    _db.ApprovalStepLogs.Add(new ApprovalStepLog {
        ApplicationId = app.Id, StepOrder = -1, StepType = StepType.Decision,
        AssigneeUserId = actorId, Action = StepAction.Closed,
        ActionAt = DateTime.UtcNow, IsCompleted = true,
    });
    await _db.SaveChangesAsync(ct);
    await _notify.NotifyAsync(EventType.Closed, app,
        recipients: new[] { app.ApplicantUserId }, ct);
}
```

---

## 5. インフラ層詳細

### 5.1 DbContext

```csharp
public class AppDbContext : DbContext, IAppDbContext
{
    private readonly ITenantContext _tenant;
    public AppDbContext(DbContextOptions<AppDbContext> opt, ITenantContext tenant) : base(opt) { _tenant = tenant; }

    public DbSet<Application> Applications => Set<Application>();
    // ... 他

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Application>().HasIndex(a => new { a.TenantId, a.ApplicationNumber }).IsUnique();
        b.Entity<Application>().HasQueryFilter(a =>
            (_tenant.IsCrossTenantAllowed || a.TenantId == _tenant.TenantId) && !a.IsDeleted);
        // ... 他テーブル同様
    }
}
```

### 5.2 申請番号採番

```csharp
public class ApplicationNumberGenerator
{
    public async Task<string> NextAsync(Guid tenantId, DateTime when, CancellationToken ct)
    {
        var prefix = $"{when:yyyy-MM}";
        // SELECT ... FOR UPDATE で行ロック
        var max = await _db.Applications
            .IgnoreQueryFilters()
            .Where(a => a.TenantId == tenantId && a.ApplicationNumber.StartsWith(prefix))
            .Select(a => a.ApplicationNumber)
            .OrderByDescending(s => s)
            .FirstOrDefaultAsync(ct);
        var seq = max == null ? 1 : int.Parse(max[^4..]) + 1;
        return $"{prefix}-{seq:D4}";
    }
}
```

### 5.3 通知ディスパッチャ

```csharp
public class NotificationDispatcher : INotificationDispatcher
{
    private readonly Channel<EmailJob> _queue;     // BackgroundService から消費

    public async Task NotifyAsync(EventType ev, Application app, IEnumerable<Guid> recipients, ...)
    {
        var template = await _db.EmailTemplates
            .Where(t => t.EventType == ev && (t.RouteId == routeId || t.RouteId == null))
            .OrderBy(t => t.RouteId == null ? 1 : 0)
            .FirstAsync();
        foreach (var userId in recipients)
        {
            var user = await _db.Users.FindAsync(userId);
            var rendered = Scriban.Template.Parse(template.BodyTemplate).Render(BuildModel(app, user));
            await _queue.Writer.WriteAsync(new EmailJob(user.Email, subject, rendered));
        }
    }
}

public class EmailSenderHostedService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var job in _queue.Reader.ReadAllAsync(ct))
        {
            try { await _smtp.SendAsync(job); LogSuccess(job); }
            catch (Exception ex) { LogFailure(job, ex); }
        }
    }
}
```

### 5.4 認証

```csharp
public class JwtAuthService
{
    public string IssueToken(User user, Tenant? tenant)
    {
        var claims = new List<Claim> {
            new("sub", user.Id.ToString()),
            new("email", user.Email),
            new("roles", string.Join(",", user.Roles.ToFlagStrings())),
        };
        if (tenant != null) claims.Add(new("tenant_id", tenant.Id.ToString()));
        if (user.Roles.HasFlag(UserRole.SystemAdmin)) claims.Add(new("is_system_admin", "true"));
        // HS256 sign, exp=60min
    }
}
```

---

## 6. API コントローラ（抜粋）

```csharp
[ApiController]
[Route("api/applications")]
[Authorize]
public class ApplicationsController : ControllerBase
{
    [HttpPost("leave")]
    [Authorize(Roles = "Applicant,TimeManager,SystemAdmin")]
    public async Task<IActionResult> CreateLeave([FromBody] CreateLeaveRequest req, CancellationToken ct)
    {
        var id = await _createUseCase.ExecuteAsync(req, User.GetUserId(), ct);
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }

    [HttpPost("{id}/approve")]
    public Task<IActionResult> Approve(Guid id, [FromBody] ApproveRequest req, CancellationToken ct)
        => _approveUseCase.ExecuteAsync(id, User.GetUserId(), req.Comment, ct).ToActionResult();

    [HttpPost("{id}/reject")]
    public Task<IActionResult> Reject(Guid id, [FromBody] RejectRequest req, CancellationToken ct)
        => _rejectUseCase.ExecuteAsync(id, User.GetUserId(), req.Comment, ct).ToActionResult();

    [HttpPost("{id}/decide")]
    public Task<IActionResult> Decide(Guid id, CancellationToken ct) => ...;

    [HttpPost("{id}/close")]
    [Authorize(Roles = "TimeManager,SystemAdmin")]
    public Task<IActionResult> Close(Guid id, CancellationToken ct) => ...;
}
```

---

## 7. フロントエンド詳細

### 7.1 ルーティング

```
/login
/dashboard
/applications
/applications/new
/applications/[id]
/applications/[id]/edit
/close-pending          (時間管理者)
/admin/users
/admin/routes
/admin/email-templates
/admin/system-settings
/admin/balances
/admin/email-logs
/sysadmin/tenants       (システム管理者)
/sysadmin/tenants/[id]
```

### 7.2 状態管理

- React Server Components + Client Components 併用
- 認証状態: `next-auth` または独自 Context
- API クライアント: Axios + Interceptor で JWT 付与・401 時に refresh

### 7.3 フォームバリデーション

- `react-hook-form` + `zod`
- 例: 取得時刻は 15 分単位（00, 15, 30, 45）に制限

### 7.4 印刷

- `@media print` で操作系 UI を `display: none`
- `window.print()` 起動

---

## 8. エラーハンドリング設計

| 種別 | HTTP | レスポンス例 |
|---|---|---|
| バリデーション失敗 | 400 | `{ "error": "validation", "details": [...] }` |
| 未認証 | 401 | `{ "error": "unauthorized" }` |
| 権限不足 | 403 | `{ "error": "forbidden" }` |
| 存在しない | 404 | `{ "error": "not_found" }` |
| ドメイン例外 | 409 | `{ "error": "invalid_state", "message": "..." }` |
| サーバ内部 | 500 | `{ "error": "internal" }`（詳細はログのみ） |

`ProblemDetailsMiddleware` で統一形式に変換。

---

## 9. ログ・監査

| ログ種別 | 出力先 | フィールド |
|---|---|---|
| アプリログ | 標準出力 → 集約基盤 | timestamp, level, traceId, tenantId, userId, message |
| 監査ログ | `ApprovalStepLogs` テーブル | applicationId, action, actor, isProxy |
| メールログ | `EmailLogs` テーブル | event, to, status, error |

---

## 10. パフォーマンス・インデックス設計

| テーブル | インデックス |
|---|---|
| Applications | (TenantId, ApplicationNumber) UNIQUE / (TenantId, Status, CurrentStepOrder) / (TenantId, ApplicantUserId) |
| ApprovalStepLogs | (TenantId, ApplicationId, StepOrder) |
| LeaveApplications | (TenantId, ApplicationId) |
| LeaveBalances | (TenantId, UserId, BalanceYear, BalanceMonth) UNIQUE |
| EmailLogs | (TenantId, SentAt) |

---

## 11. デプロイ・構成

### 11.1 Docker Compose（例）

```yaml
services:
  api:
    build: ./src/ApprovalSystem.Api
    environment:
      - DatabaseProvider=MySQL
      - ConnectionStrings__MySQL=Server=db;...
    depends_on: [db]
  web:
    build: ./src/ApprovalSystem.Web
    environment:
      - NEXT_PUBLIC_API_BASE=http://api:8080
  db:
    image: mysql:8.0
    environment: { MYSQL_ROOT_PASSWORD: secret, MYSQL_DATABASE: approval }
```

### 11.2 環境変数

| キー | 例 |
|---|---|
| `DatabaseProvider` | `InMemory` / `MySQL` |
| `Jwt:SecretKey` | 32 byte 以上 |
| `Jwt:ExpiresMinutes` | `60` |
| `Smtp:Host` / `Smtp:Port` / `Smtp:User` / `Smtp:Password` | - |
| `Smtp:From` | `noreply@example.com` |

---

*テスト仕様は `test-spec.md`、API 仕様は `api-spec.md`、テーブル定義は `data-model.md` を参照。*
