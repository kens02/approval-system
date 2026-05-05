# Excel VBA 暫定ツール データ設計書（シート定義）

**バージョン**: 1.0  
**作成日**: 2026-05-01

---

## 1. 設計方針

- 各シートは **テーブル形式（1行目=ヘッダ、2行目以降=データ）** で構成し、Excel の「テーブル機能」を有効化する
- 列名・列順・列値は **Web 版 `data-model.md` のカラム名にスネークケースで揃える**（移行容易性）
- 主キーは原則 **GUID（=`Mid(CreateObject("Scriptlet.TypeLib").Guid),2,36)`）**
- ソフトデリートは `is_deleted` 列で表現（行は削除しない）
- 日時はすべて `yyyy-mm-dd hh:nn:ss` 形式の文字列で保存（タイムゾーン非考慮）

---

## 2. シート一覧

| # | シート名 | 役割 | 行数想定 | 保護種別 |
|---|---|---|---|---|
| 1 | `定型用紙` | 申請書フォーム（作業領域） | 固定 | 入力欄以外を保護 |
| 2 | `申請リスト` | 申請メイン台帳 | ～10,000 | 行追加のみ可 |
| 3 | `休暇詳細` | 休暇申請の詳細 | ～10,000 | VBA経由のみ編集 |
| 4 | `決裁ルート` | 申請者ごとのルート定義 | ～100 | 環境設定パスワード保護 |
| 5 | `決裁ルートステップ` | ルートのステップ詳細 | ～500 | 環境設定パスワード保護 |
| 6 | `ユーザーマスタ` | ユーザー定義 | ～100 | 環境設定パスワード保護 |
| 7 | `メールテンプレート` | 通知文面 | ～20 | 環境設定パスワード保護 |
| 8 | `休暇残管理` | 月次付与・繰越・取得分 | ～10,000 | 環境設定パスワード保護 |
| 9 | `環境設定` | パスワード・設定値 | ～50 | 環境設定パスワード保護 |
| 10 | `操作ログ` | 監査ログ | 増分のみ | VBA以外編集不可 |
| 11 | `メール送信ログ` | 送信履歴 | 増分のみ | VBA以外編集不可 |
| 12 | `ダッシュボード` | 集計表示（任意） | 固定 | 全保護 |

---

## 3. シート定義

### 3.1 `申請リスト`（Web版 `Applications` 相当）

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | application_id | 文字列(GUID) | ✅ | 主キー |
| B | application_number | 文字列 | ✅ | yyyy-mm-NNNN |
| C | applicant_user_id | 文字列(GUID) | ✅ | 申請者ID |
| D | applicant_display_name | 文字列 | ✅ | 表示用（参照値） |
| E | application_type | 文字列 | ✅ | leave_annual_paid |
| F | status | 文字列 | ✅ | draft/in_progress/decided/closed/rejected/cancelled |
| G | current_step_order | 整数 | ✅ | 現ステップ番号 |
| H | parent_application_id | 文字列 | - | 差し戻し元 |
| I | is_deleted | 真偽（TRUE/FALSE） | ✅ | ソフトデリートフラグ |
| J | submitted_at | 日時 | - | - |
| K | decided_at | 日時 | - | - |
| L | closed_at | 日時 | - | - |
| M | closed_by_user_id | 文字列 | - | クローズ実施者 |
| N | created_at | 日時 | ✅ | - |
| O | updated_at | 日時 | ✅ | - |

**インデックス（運用上）**: A列でフィルタ・ソート可能にする。

### 3.2 `休暇詳細`（Web版 `LeaveApplications` 相当）

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | leave_id | 文字列(GUID) | ✅ | 主キー |
| B | application_id | 文字列(GUID) | ✅ | 申請リスト.A への外部キー |
| C | leave_type | 文字列 | ✅ | annual_paid_leave |
| D | take_unit | 文字列 | ✅ | hour / day |
| E | date_from | 日付 | ✅ | yyyy-mm-dd |
| F | date_to | 日付 | ✅ | - |
| G | time_from | 時刻 | - | hh:nn |
| H | time_to | 時刻 | - | - |
| I | total_minutes | 整数 | ✅ | 合計分数 |
| J | total_days | 小数 | ✅ | 日換算 |
| K | reason | 文字列 | - | 理由 |
| L | remarks | 文字列 | - | 備考 |

### 3.3 `決裁ルート`

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | route_id | 文字列(GUID) | ✅ | 主キー |
| B | route_name | 文字列 | ✅ | 管理用名称 |
| C | applicant_user_id | 文字列 | ✅ | 適用される申請者 |
| D | is_active | 真偽 | ✅ | 有効フラグ |
| E | created_at | 日時 | ✅ | - |
| F | updated_at | 日時 | ✅ | - |

### 3.4 `決裁ルートステップ`

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | step_id | 文字列(GUID) | ✅ | - |
| B | route_id | 文字列 | ✅ | 決裁ルート.A への外部キー |
| C | step_order | 整数 | ✅ | 1, 2, 3, ... |
| D | step_type | 文字列 | ✅ | consulted / approval / decision |
| E | assignee_user_id | 文字列 | ✅ | 担当者1名 |
| F | assignee_display_name | 文字列 | ✅ | 表示用 |
| G | assignee_email | 文字列 | ✅ | 通知先 |

### 3.5 `ユーザーマスタ`

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | user_id | 文字列(GUID) | ✅ | - |
| B | name_last | 文字列 | ✅ | 姓 |
| C | name_first | 文字列 | ✅ | 名 |
| D | display_name | 文字列 | ✅ | 表示名 |
| E | email | 文字列 | ✅ | メール |
| F | department | 文字列 | - | 部署 |
| G | role_viewer | 真偽 | ✅ | 閲覧者ロール |
| H | role_actor | 真偽 | ✅ | 申請者・決裁者ロール |
| I | role_admin | 真偽 | ✅ | 環境設定者ロール |
| J | is_active | 真偽 | ✅ | - |

> Web 版へ移行時: `role_*` 列 → `roles`（ビット和）に変換。`role_admin = TRUE` は SystemAdmin + TimeManager 双方を持つユーザーとしてマップする。

### 3.6 `メールテンプレート`

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | template_id | 文字列(GUID) | ✅ | - |
| B | event_type | 文字列 | ✅ | submitted/step_approved/rejected/decided/closed/cancelled |
| C | subject_template | 文字列 | ✅ | 件名（プレースホルダ可） |
| D | body_template | 長文字列 | ✅ | 本文 |
| E | is_html | 真偽 | ✅ | HTMLメールか |

**プレースホルダ一覧**: `{申請者氏名}` / `{申請番号}` / `{申請種別}` / `{取得日From}` / `{取得日To}` / `{フェーズ名}` / `{処理者氏名}` / `{コメント}` / `{ファイルパス}` / `{申請ID}`

### 3.7 `休暇残管理`

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | balance_id | 文字列(GUID) | ✅ | - |
| B | user_id | 文字列 | ✅ | - |
| C | balance_year | 整数 | ✅ | 西暦4桁 |
| D | balance_month | 整数 | ✅ | 1-12 |
| E | carry_over_days | 小数 | ✅ | 繰越（年度開始月のみ正値） |
| F | granted_days | 小数 | ✅ | 当月付与 |
| G | used_minutes | 整数 | ✅ | 当月使用分（参考） |

> 残日数は計算により都度算出（Web 版同様）。

### 3.8 `環境設定`（キー＝値方式）

| # | 列名 | 型 | 説明 |
|---|---|---|---|
| A | setting_key | 文字列 | 設定キー |
| B | setting_value | 文字列 | 設定値 |
| C | description | 文字列 | 説明（任意） |
| D | is_secret | 真偽 | パスワード等の機微情報フラグ |

#### 設定キー一覧

| キー | 例 | 説明 |
|---|---|---|
| `password.decision` | （ハッシュ） | 決裁パスワード（SHA-256 ハッシュ） |
| `password.admin` | （ハッシュ） | 環境設定パスワード（同左） |
| `password.salt` | ランダム文字列 | ハッシュ用 salt |
| `file.shared_path` | `\\fileserver\share\承認決裁\approval.xlsm` | 共有フォルダ上の自ファイル UNC パス |
| `mail.method` | `outlook` / `cdo` | 送信方式 |
| `mail.smtp_host` | `smtp.example.com` | CDO 使用時 |
| `mail.smtp_port` | `587` | - |
| `mail.smtp_user` | `noreply@...` | - |
| `mail.smtp_password` | （平文 or 簡易暗号化） | - |
| `mail.from_address` | `noreply@...` | 送信元 |
| `leave.monthly_grant_days` | `2` | 月次付与日数 |
| `leave.max_carryover_days` | `30` | 繰越上限 |
| `leave.work_minutes_per_day` | `465` | 1日労働分数 |
| `leave.fiscal_year_start_month` | `4` | 年度開始月 |
| `app.next_seq.2026-05` | `7` | 申請番号採番用カウンタ（年月別） |
| `lock.enabled` | `TRUE` | ロックファイル使用可否 |

> **重要**: パスワード列は SHA-256 + salt でハッシュ化して保存。平文では保存しない。

### 3.9 `操作ログ`（Web版 `ApprovalStepLogs` 相当・改ざん防止）

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | log_id | 文字列(GUID) | ✅ | - |
| B | application_id | 文字列 | ✅ | - |
| C | step_order | 整数 | ✅ | -1 はクローズ等のフロー外 |
| D | step_type | 文字列 | ✅ | consulted/approval/decision |
| E | assignee_user_id | 文字列 | ✅ | 実行者 |
| F | assignee_display_name | 文字列 | ✅ | - |
| G | action | 文字列 | ✅ | submitted/approved/rejected/decided/closed/cancelled |
| H | is_proxy | 真偽 | ✅ | 代行フラグ |
| I | action_at | 日時 | ✅ | - |
| J | comment | 文字列 | - | 差戻時必須 |
| K | hash | 文字列 | ✅ | 前行 hash + 当行内容の SHA-256（改ざん検知） |

> 改ざん検知: 起動時に hash チェーンを検証する処理を任意で実装。

### 3.10 `メール送信ログ`

| # | 列名 | 型 | 必須 | 説明 |
|---|---|---|:-:|---|
| A | log_id | 文字列(GUID) | ✅ | - |
| B | application_id | 文字列 | - | システム通知は空可 |
| C | event_type | 文字列 | ✅ | - |
| D | to_email | 文字列 | ✅ | 宛先 |
| E | subject | 文字列 | ✅ | - |
| F | body_excerpt | 文字列 | - | 先頭500文字 |
| G | sent_at | 日時 | ✅ | - |
| H | status | 文字列 | ✅ | success / failed |
| I | error_message | 文字列 | - | 失敗時 |

### 3.11 `ダッシュボード`（任意）

セルベースの集計表示（数式 or VBA）。
- 自分の残日数
- 申請中の件数（ステータス別）
- クローズ待ち件数（環境設定者向け）
- 月別取得実績の簡易グラフ

---

## 4. 採番ロジック

### 4.1 申請番号

```
formatの key: app.next_seq.{yyyy}-{mm}
1. 当該年月のカウンタを取得（無ければ 0）
2. +1 して書き戻し
3. yyyy-mm-NNNN の形式で文字列化
```

VBA 擬似コード:
```vb
Function GenApplicationNumber() As String
    Dim ym As String: ym = Format(Now, "yyyy-mm")
    Dim k As String: k = "app.next_seq." & ym
    Dim n As Long: n = CLng(GetSetting(k, "0")) + 1
    SetSetting k, CStr(n)                 ' 環境設定シートに書き戻し
    GenApplicationNumber = ym & "-" & Format(n, "0000")
End Function
```

### 4.2 GUID

```vb
Function NewGuid() As String
    NewGuid = Mid$(CreateObject("Scriptlet.TypeLib").Guid, 2, 36)
End Function
```

---

## 5. データ整合性

| ルール | 実装方法 |
|---|---|
| 外部キー制約 | VBA 側で参照存在を検証。Excel に物理 FK は無し |
| 必須項目 | UserForm / 提出時バリデーションで担保 |
| ユニーク制約 | `application_number` は VBA 採番＋書き込み前重複チェック |
| 状態遷移 | `StateMachine` モジュールで集中管理（`detail-design.md`） |
| ソフトデリート | `is_deleted=TRUE` の行は通常一覧から非表示（オートフィルタ条件） |

---

## 6. ファイル分割の指針（運用上）

- 1 Book で 5,000 件を超えたら **年度別ファイル分割** を検討
  - 例: `approval-2026.xlsm`, `approval-2027.xlsm`
  - 過去ファイルは読み取り専用化、最新ファイルのみ編集
- ユーザーマスタ・決裁ルートは **新ファイル作成時にコピー**
- 残日数は新ファイルへ繰越データを引き継ぐ

---

*シート保護のパスワード制御は `access-control.md` を参照。*
