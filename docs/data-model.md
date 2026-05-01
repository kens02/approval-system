# 承認決裁システム データ設計書

**バージョン**: 1.0  
**作成日**: 2026-05-01  
**対象 DBMS**: MySQL 8.0 / EF Core InMemory（開発）

---

## 1. ER 図（概念）

```
                ┌─────────┐
                │ Tenants │
                └────┬────┘
                     │ 1
       ┌─────────────┼─────────────────────────────┐
       │             │                             │
       ▼ N           ▼ N                           ▼ N
   ┌────────┐  ┌──────────────┐         ┌──────────────────┐
   │ Users  │  │ ApprovalRoutes│         │ SystemSettings  │
   └───┬────┘  └──────┬───────┘         └──────────────────┘
       │              │ 1
       │              ▼ N
       │       ┌──────────────────┐
       │       │ApprovalRouteSteps│
       │       └──────────────────┘
       │
       │ 1
       ▼ N
   ┌────────────┐ 1     N ┌────────────────┐
   │Applications├────────►│LeaveApplications│
   └─────┬──────┘         └────────────────┘
         │ 1
         ▼ N
   ┌──────────────────┐
   │ApprovalStepLogs  │
   └──────────────────┘

   ┌──────────────┐    ┌────────────────┐    ┌────────────┐
   │LeaveBalances │    │ EmailTemplates │    │ EmailLogs  │
   └──────────────┘    └────────────────┘    └────────────┘
```

---

## 2. テーブル定義

> 注: 全テーブル（`Tenants` を除く）に `tenant_id` を付与し、ミドルウェアで強制スコープ。  
> 全テーブル共通カラム: `created_at TIMESTAMP NOT NULL`, `updated_at TIMESTAMP NOT NULL`。

### 2.1 Tenants（テナント）

| カラム | 型 | NULL | デフォルト | 説明 |
|---|---|:-:|---|---|
| id | CHAR(36) PK | × | UUID | テナントID |
| name | VARCHAR(100) | × | - | 組織名 |
| code | VARCHAR(50) UNIQUE | × | - | 組織コード（ログイン時入力） |
| is_active | BOOLEAN | × | true | 有効フラグ |
| created_at | TIMESTAMP | × | NOW | - |
| updated_at | TIMESTAMP | × | NOW | - |

### 2.2 Users（ユーザー）

| カラム | 型 | NULL | デフォルト | 説明 |
|---|---|:-:|---|---|
| id | CHAR(36) PK | × | UUID | - |
| tenant_id | CHAR(36) FK→Tenants.id | ○ | NULL | システム管理者は NULL |
| name_last | VARCHAR(50) | × | - | 姓 |
| name_first | VARCHAR(50) | × | - | 名 |
| display_name | VARCHAR(100) | × | - | 表示名 |
| email | VARCHAR(255) | × | - | - |
| department | VARCHAR(100) | ○ | NULL | 所属部署 |
| roles | INT | × | 1 | UserRole [Flags] のビット和 |
| password_hash | VARCHAR(255) | × | - | bcrypt |
| is_active | BOOLEAN | × | true | - |

**インデックス**:
- UNIQUE (tenant_id, email)
- INDEX (tenant_id, is_active)

### 2.3 ApprovalRoutes（決裁ルート）

| カラム | 型 | NULL | 説明 |
|---|---|:-:|---|
| id | CHAR(36) PK | × | - |
| tenant_id | CHAR(36) FK | × | - |
| name | VARCHAR(100) | × | ルート名 |
| applicant_user_id | CHAR(36) FK→Users | × | このルートが適用される申請者 |

**インデックス**: UNIQUE (tenant_id, applicant_user_id)（申請者あたり1ルート）

### 2.4 ApprovalRouteSteps（ルートステップ）

| カラム | 型 | NULL | 説明 |
|---|---|:-:|---|
| id | CHAR(36) PK | × | - |
| tenant_id | CHAR(36) FK | × | - |
| route_id | CHAR(36) FK→ApprovalRoutes | × | - |
| step_order | INT | × | 1, 2, 3, ... |
| step_type | ENUM('consulted','approval','decision') | × | - |
| assignee_user_id | CHAR(36) FK→Users | × | 担当者1名 |

**制約**: UNIQUE (tenant_id, route_id, step_order)

### 2.5 Applications（申請）

| カラム | 型 | NULL | デフォルト | 説明 |
|---|---|:-:|---|---|
| id | CHAR(36) PK | × | UUID | - |
| tenant_id | CHAR(36) FK | × | - | - |
| application_number | VARCHAR(15) | × | - | yyyy-mm-NNNN |
| applicant_user_id | CHAR(36) FK→Users | × | - | - |
| application_type | ENUM('leave_annual_paid'…) | × | leave_annual_paid | - |
| status | ENUM('draft','in_progress','decided','closed','rejected','cancelled') | × | draft | - |
| current_step_order | INT | × | 0 | 現在のステップ番号 |
| parent_application_id | CHAR(36) FK→Applications.id | ○ | NULL | 差し戻し元 |
| is_deleted | BOOLEAN | × | false | ソフトデリート |
| submitted_at | TIMESTAMP | ○ | NULL | - |
| decided_at | TIMESTAMP | ○ | NULL | - |
| closed_at | TIMESTAMP | ○ | NULL | - |
| closed_by_user_id | CHAR(36) FK→Users | ○ | NULL | - |

**インデックス**:
- UNIQUE (tenant_id, application_number)
- INDEX (tenant_id, status, current_step_order)
- INDEX (tenant_id, applicant_user_id, submitted_at)
- INDEX (parent_application_id)

### 2.6 LeaveApplications（休暇申請詳細）

| カラム | 型 | NULL | 説明 |
|---|---|:-:|---|
| id | CHAR(36) PK | × | - |
| tenant_id | CHAR(36) FK | × | - |
| application_id | CHAR(36) FK→Applications | × | - |
| leave_type | ENUM('annual_paid_leave'…) | × | - |
| take_unit | ENUM('hour','day') | × | - |
| date_from | DATE | × | - |
| date_to | DATE | × | - |
| time_from | TIME | ○ | 時間取得時 |
| time_to | TIME | ○ | 時間取得時 |
| total_minutes | INT | × | 合計分数 |
| total_days | DECIMAL(5,2) | × | 日換算 |
| reason | VARCHAR(500) | ○ | - |
| remarks | VARCHAR(1000) | ○ | - |

**制約**: UNIQUE (tenant_id, application_id)

### 2.7 ApprovalStepLogs（ステップ処理ログ）

| カラム | 型 | NULL | 説明 |
|---|---|:-:|---|
| id | CHAR(36) PK | × | - |
| tenant_id | CHAR(36) FK | × | - |
| application_id | CHAR(36) FK | × | - |
| step_order | INT | × | -1 はクローズ等のフロー外操作 |
| step_type | ENUM('consulted','approval','decision') | × | - |
| assignee_user_id | CHAR(36) FK→Users | × | 実行者 |
| action | ENUM('approved','rejected','decided','closed','cancelled') | × | - |
| is_proxy | BOOLEAN | × | 代行フラグ |
| action_at | TIMESTAMP | × | - |
| comment | VARCHAR(1000) | ○ | 差し戻し時必須 |
| is_completed | BOOLEAN | × | - |

**インデックス**: INDEX (tenant_id, application_id, step_order)

### 2.8 LeaveBalances（休暇残管理）

| カラム | 型 | NULL | 説明 |
|---|---|:-:|---|
| id | CHAR(36) PK | × | - |
| tenant_id | CHAR(36) FK | × | - |
| user_id | CHAR(36) FK→Users | × | - |
| balance_year | INT | × | 西暦4桁 |
| balance_month | INT | × | 1-12 |
| carry_over_days | DECIMAL(5,2) | × | 繰越（年度開始月のみ正値） |
| granted_days | DECIMAL(5,2) | × | 当月付与 |
| used_minutes | INT | × | 当月使用分（参考値、再集計可） |

**制約**: UNIQUE (tenant_id, user_id, balance_year, balance_month)

### 2.9 SystemSettings（テナント設定）

| カラム | 型 | NULL | デフォルト | 説明 |
|---|---|:-:|---|---|
| id | CHAR(36) PK | × | - | - |
| tenant_id | CHAR(36) UNIQUE FK | × | - | - |
| monthly_grant_days | DECIMAL(5,2) | × | 2.00 | 月次付与日数 |
| max_carryover_days | DECIMAL(5,2) | × | 30.00 | 繰越上限 |
| work_minutes_per_day | INT | × | 465 | 1日労働分数 |
| fiscal_year_start_month | INT | × | 4 | 年度開始月 |

### 2.10 EmailTemplates（メールテンプレート）

| カラム | 型 | NULL | 説明 |
|---|---|:-:|---|
| id | CHAR(36) PK | × | - |
| tenant_id | CHAR(36) FK | × | - |
| route_id | CHAR(36) FK→ApprovalRoutes | ○ | NULL=テナント共通 |
| event_type | ENUM('submitted','step_approved','rejected','decided','closed','cancelled') | × | - |
| subject_template | VARCHAR(200) | × | Scriban 形式 |
| body_template | TEXT | × | Scriban 形式 |
| is_html | BOOLEAN | × | - |

**制約**: UNIQUE (tenant_id, route_id, event_type)（route_id NULL 含む）

### 2.11 EmailLogs（メール送信ログ）

| カラム | 型 | NULL | 説明 |
|---|---|:-:|---|
| id | CHAR(36) PK | × | - |
| tenant_id | CHAR(36) FK | × | - |
| application_id | CHAR(36) FK | ○ | システム通知は NULL 可 |
| event_type | ENUM(...) | × | - |
| to_email | VARCHAR(255) | × | - |
| subject | VARCHAR(200) | × | - |
| sent_at | TIMESTAMP | × | - |
| status | ENUM('success','failed') | × | - |
| error_message | VARCHAR(1000) | ○ | - |

**インデックス**: INDEX (tenant_id, sent_at), INDEX (status, sent_at)

---

## 3. 列挙型一覧

| 名前 | 値 |
|---|---|
| ApplicationStatus | draft, in_progress, decided, closed, rejected, cancelled |
| ApplicationType | leave_annual_paid |
| LeaveType | annual_paid_leave（将来: special_leave, sick_leave 等） |
| TakeUnit | hour, day |
| StepType | consulted, approval, decision |
| StepAction | approved, rejected, decided, closed, cancelled |
| EventType | submitted, step_approved, rejected, decided, closed, cancelled |
| UserRole [Flags] | applicant=1, consultor=2, approver=4, decider=8, time_manager=16, system_admin=32 |

---

## 4. 採番ルール

### 4.1 申請番号

```
{yyyy}-{mm}-{NNNN}
   |     |     └─ 同テナント・同年月内の連番（4桁ゼロ埋め）
   |     └─ 申請提出月（2桁）
   └─ 申請提出年（4桁）
```

実装メモ:
- 採番は提出時（`submitted_at` 設定時）に実施
- トランザクション内で `SELECT ... FOR UPDATE` 同等のロックを使用
- テナント間で番号は重複可

---

## 5. 残日数計算ロジック

```
remaining(user, asOf) =
    LeaveBalances.carry_over_days[at fiscal_year_start_month of current FY]
  + Σ LeaveBalances.granted_days[from FY start through current month]
  − Σ approved LeaveApplications.total_minutes[in current FY] / work_minutes_per_day
```

- 残日数は永続化しない（都度計算）
- バッチ:
  - 月次付与（毎月 1 日 0:00）: 全有効ユーザーに `granted_days` を追加
  - 年度繰越（年度開始月 1 日）: `min(remaining, max_carryover_days)` を `carry_over_days` として登録

---

## 6. ソフトデリート方針

| テーブル | 物理削除 | ソフトデリート | 備考 |
|---|:-:|:-:|---|
| Applications | × | ✅ (`is_deleted`) | 履歴保持 |
| LeaveApplications | × | （Applications 連動） | - |
| Users | × | ✅ (`is_active=false`) | - |
| Tenants | × | ✅ (`is_active=false`) | - |
| ApprovalStepLogs | × | × | 監査のため削除不可 |
| EmailLogs | × | × | 監査のため削除不可 |

---

## 7. マイグレーション戦略

- 開発環境: EF Core InMemory（Migration 不要、起動時にシード）
- 本番環境: `dotnet ef migrations add Init` → `dotnet ef database update`
- マイグレーションは Git 管理（`Infrastructure/Migrations/`）

---

## 8. 初期シードデータ（例）

| テーブル | データ |
|---|---|
| Tenants | demo / 株式会社デモ |
| Users (sysadmin) | admin@platform / SystemAdmin / tenant_id=NULL |
| Users (demo tenant) | yamada@demo / Applicant, suzuki@demo / Approver, sato@demo / Decider, tanaka@demo / TimeManager |
| ApprovalRoutes | yamada のルート: ステップ1=承認(suzuki), ステップ2=決裁(sato) |
| SystemSettings | demo テナント既定値 |
| EmailTemplates | 6 イベント分のデフォルトテンプレート |

---

*詳細な物理 DDL は実装時に EF Core マイグレーションで自動生成される。*
