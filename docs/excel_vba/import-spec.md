# Excel VBA 暫定ツール → Web 版インポート仕様書

**バージョン**: 1.0  
**作成日**: 2026-05-01

---

## 1. 目的

暫定ツール（Excel）で運用された申請データを、Web 版稼働時に **無損失で移行** する。  
本書は VBA 側の CSV エクスポート仕様と、Web 版（C#）のインポート API・コマンド仕様を定義する。

---

## 2. 全体フロー

```
[Excel VBA]                 [中間ファイル]                   [Web 版]
                                                                
[F_AdminMenu]                                                  
   └─► CSV エクスポート ───► import_yyyymmdd_hhmmss/        
                                ├ tenants.csv               
                                ├ users.csv                 
                                ├ approval_routes.csv       
                                ├ approval_route_steps.csv  
                                ├ applications.csv          
                                ├ leave_applications.csv    
                                ├ approval_step_logs.csv    
                                ├ leave_balances.csv        
                                ├ email_templates.csv       
                                └ system_settings.csv       
                                                              ↓
                                                       [dotnet ApprovalSystem.Importer]
                                                              ↓
                                                          [Web 版 DB]
```

---

## 3. CSV ファイル仕様

### 3.1 共通

| 項目 | 仕様 |
|---|---|
| 文字コード | UTF-8 (BOM 付き) |
| 改行 | LF (`\n`) |
| 区切り | カンマ (`,`) |
| 引用符 | ダブルクォート（カンマ・改行を含む値のみ） |
| 1行目 | ヘッダ（Web 版テーブル列名と完全一致） |
| 日時 | ISO 8601 (`yyyy-MM-ddTHH:mm:ss`) |
| 真偽 | `true` / `false` |
| NULL | 空文字列 |

### 3.2 ファイル一覧と対応マッピング

| ファイル | Web 版テーブル | 暫定ツールのソース |
|---|---|---|
| `tenants.csv` | `Tenants` | （新規生成。1テナント分の固定データ） |
| `users.csv` | `Users` | `ユーザーマスタ` シート |
| `approval_routes.csv` | `ApprovalRoutes` | `決裁ルート` シート |
| `approval_route_steps.csv` | `ApprovalRouteSteps` | `決裁ルートステップ` シート |
| `applications.csv` | `Applications` | `申請リスト` シート（is_deleted 行も含む） |
| `leave_applications.csv` | `LeaveApplications` | `休暇詳細` シート |
| `approval_step_logs.csv` | `ApprovalStepLogs` | `操作ログ` シート |
| `leave_balances.csv` | `LeaveBalances` | `休暇残管理` シート |
| `email_templates.csv` | `EmailTemplates` | `メールテンプレート` シート |
| `system_settings.csv` | `SystemSettings` | `環境設定` シート（一部） |
| `email_logs.csv` | `EmailLogs` | `メール送信ログ` シート（任意） |

---

## 4. 列マッピング（主要テーブル）

### 4.1 `tenants.csv`（1行のみ生成）

| 列 | 値 | 備考 |
|---|---|---|
| id | （新規 GUID） | エクスポート時に `M_Util.NewGuid()` で生成し、以降の `tenant_id` で参照 |
| name | 環境設定の `tenant.name` | ユーザー入力 |
| code | 環境設定の `tenant.code` | ユーザー入力（英数小文字） |
| is_active | true | 固定 |
| created_at | エクスポート日時 | - |
| updated_at | エクスポート日時 | - |

### 4.2 `users.csv`

| Web 版列 | 暫定列 | 変換ルール |
|---|---|---|
| id | user_id | そのまま |
| tenant_id | （4.1 で生成した tenant id） | 全行に同じ ID を埋める |
| name_last | name_last | - |
| name_first | name_first | - |
| display_name | display_name | - |
| email | email | - |
| department | department | - |
| roles | role_viewer / role_actor / role_admin | ビット和に変換: viewer→0, actor→Applicant\|Approver\|Decider\|Consultor=15, admin→TimeManager\|SystemAdmin=48。和を取る |
| password_hash | （ダミー値 or 強制リセットフラグ） | 移行直後にユーザーへパスワード再設定を依頼 |
| is_active | is_active | - |
| created_at | （現在時刻） | - |
| updated_at | （現在時刻） | - |

### 4.3 `applications.csv`

| Web 版列 | 暫定列 | 変換ルール |
|---|---|---|
| id | application_id | そのまま |
| tenant_id | tenant_id（生成） | - |
| application_number | application_number | フォーマット同一 |
| applicant_user_id | applicant_user_id | - |
| application_type | application_type | `leave_annual_paid` |
| status | status | 同一値 |
| current_step_order | current_step_order | - |
| parent_application_id | parent_application_id | - |
| is_deleted | is_deleted | TRUE/FALSE → true/false |
| submitted_at | submitted_at | ISO 8601 |
| decided_at | decided_at | - |
| closed_at | closed_at | - |
| closed_by_user_id | closed_by_user_id | - |
| created_at | created_at | - |
| updated_at | updated_at | - |

### 4.4 `approval_step_logs.csv`

| Web 版列 | 暫定列 | 変換ルール |
|---|---|---|
| id | log_id | - |
| tenant_id | tenant_id | - |
| application_id | application_id | - |
| step_order | step_order | - |
| step_type | step_type | - |
| assignee_user_id | assignee_user_id | - |
| action | action | - |
| is_proxy | is_proxy | - |
| action_at | action_at | - |
| comment | comment | - |
| is_completed | TRUE 固定 | 移行後はすべて完了済 |

### 4.5 `system_settings.csv`

| Web 版列 | 暫定キー（環境設定シート） |
|---|---|
| tenant_id | （生成） |
| monthly_grant_days | `leave.monthly_grant_days` |
| max_carryover_days | `leave.max_carryover_days` |
| work_minutes_per_day | `leave.work_minutes_per_day` |
| fiscal_year_start_month | `leave.fiscal_year_start_month` |

---

## 5. VBA 側エクスポート実装方針

### 5.1 起動経路

`F_AdminMenu` →「CSV エクスポート」→ 環境設定パスワード認証 → 出力先フォルダ選択 → 実行

### 5.2 出力先ディレクトリ

```
C:\Users\<user>\Documents\承認決裁_export\import_<yyyymmdd_hhmmss>\
   ├ tenants.csv
   ├ users.csv
   ├ ...
   └ _manifest.json    ← エクスポートメタ情報
```

### 5.3 マニフェスト

```json
{
  "exported_at": "2026-05-01T18:30:00",
  "source_file": "\\\\fileserver\\share\\承認決裁\\approval.xlsm",
  "exporter_version": "1.0",
  "tenant": { "id": "uuid", "name": "...", "code": "..." },
  "row_counts": {
    "users": 12,
    "applications": 348,
    "leave_applications": 348,
    "approval_step_logs": 1245
  },
  "checksum_sha256": {
    "users.csv": "...",
    "applications.csv": "..."
  }
}
```

### 5.4 主要関数（M_Export）

```vb
Public Sub ExportAll(targetDir As String)
    If Not M_Auth.RequireAuth(AuthLevel_Admin) Then Exit Sub
    
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    On Error GoTo Cleanup
    
    Dim tenantId As String: tenantId = M_Util.NewGuid()
    
    ExportTenants targetDir & "\tenants.csv", tenantId
    ExportUsers targetDir & "\users.csv", tenantId
    ExportRoutes targetDir & "\approval_routes.csv", tenantId
    ExportRouteSteps targetDir & "\approval_route_steps.csv", tenantId
    ExportApplications targetDir & "\applications.csv", tenantId
    ExportLeaveDetails targetDir & "\leave_applications.csv", tenantId
    ExportLogs targetDir & "\approval_step_logs.csv", tenantId
    ExportBalances targetDir & "\leave_balances.csv", tenantId
    ExportTemplates targetDir & "\email_templates.csv", tenantId
    ExportSettings targetDir & "\system_settings.csv", tenantId
    
    WriteManifest targetDir, tenantId
    MsgBox "エクスポート完了: " & targetDir, vbInformation
    
Cleanup:
    Application.Calculation = xlCalculationAutomatic
    Application.ScreenUpdating = True
    If Err.Number <> 0 Then MsgBox "エクスポート失敗: " & Err.Description, vbCritical
End Sub

Private Sub ExportApplications(filePath As String, tenantId As String)
    Dim ws As Worksheet: Set ws = ThisWorkbook.Worksheets("申請リスト")
    Dim lastRow As Long: lastRow = ws.Cells(ws.Rows.Count, 1).End(xlUp).Row
    
    Dim fnum As Integer: fnum = FreeFile
    Open filePath For Output As #fnum
    
    ' ヘッダ
    Print #fnum, JoinCsv(Array( _
        "id","tenant_id","application_number","applicant_user_id", _
        "application_type","status","current_step_order","parent_application_id", _
        "is_deleted","submitted_at","decided_at","closed_at","closed_by_user_id", _
        "created_at","updated_at"))
    
    Dim r As Long
    For r = 2 To lastRow
        Print #fnum, JoinCsv(Array( _
            CStr(ws.Cells(r,1).Value), tenantId, _
            CStr(ws.Cells(r,2).Value), CStr(ws.Cells(r,3).Value), _
            CStr(ws.Cells(r,5).Value), CStr(ws.Cells(r,6).Value), _
            CStr(ws.Cells(r,7).Value), CStr(ws.Cells(r,8).Value), _
            BoolToStr(ws.Cells(r,9).Value), _
            ToIso(ws.Cells(r,10).Value), ToIso(ws.Cells(r,11).Value), _
            ToIso(ws.Cells(r,12).Value), CStr(ws.Cells(r,13).Value), _
            ToIso(ws.Cells(r,14).Value), ToIso(ws.Cells(r,15).Value)))
    Next r
    Close #fnum
End Sub

Private Function JoinCsv(arr As Variant) As String
    Dim s As String, i As Long, v As String
    For i = LBound(arr) To UBound(arr)
        v = CStr(arr(i))
        If InStr(v, ",") + InStr(v, """") + InStr(v, vbLf) > 0 Then
            v = """" & Replace(v, """", """""") & """"
        End If
        s = s & IIf(i = LBound(arr), v, "," & v)
    Next i
    JoinCsv = s
End Function

Private Function ToIso(v As Variant) As String
    If IsEmpty(v) Or IsNull(v) Or v = "" Then Exit Function
    ToIso = Format(CDate(v), "yyyy-mm-dd") & "T" & Format(CDate(v), "hh:nn:ss")
End Function
```

---

## 6. Web 版インポート側

### 6.1 形式

`dotnet ApprovalSystem.Importer` という CLI ツールを別途用意。

```bash
dotnet ApprovalSystem.Importer \
  --source ./import_20260501_183000 \
  --target-connection "Server=...;Database=approval;..." \
  --create-tenant   # tenants.csv の内容で新規テナント作成
```

オプション:
| フラグ | 説明 |
|---|---|
| `--source <dir>` | エクスポート出力ディレクトリ |
| `--target-connection <conn>` | 接続文字列 |
| `--create-tenant` | テナント新規作成 |
| `--existing-tenant <id>` | 既存テナントへマージ |
| `--dry-run` | 検証のみ実施 |
| `--force-password-reset` | 全ユーザーをパスワード再設定対象としてマーク |

### 6.2 インポート順序

依存関係に基づく順序:

1. tenants.csv
2. users.csv
3. system_settings.csv
4. approval_routes.csv
5. approval_route_steps.csv
6. email_templates.csv
7. applications.csv
8. leave_applications.csv
9. approval_step_logs.csv
10. leave_balances.csv
11. email_logs.csv

### 6.3 検証ステップ（dry-run）

| 項目 | チェック |
|---|---|
| ファイル存在 | 必須 CSV が揃っているか |
| ヘッダ | 列名・列順が定義通りか |
| 外部キー | application_id の applicant_user_id, parent_application_id が users.csv / applications.csv 内に存在するか |
| ユニーク | application_number が同テナント内で重複しないか |
| 列挙値 | status, action, step_type の値が許可リストに含まれるか |
| 日時形式 | ISO 8601 で解釈可能か |

エラーがあれば、レポート（CSV 行番号付き）を出力して中断。

### 6.4 トランザクション

- 1つの DB トランザクションで全テーブルをインポート
- 途中失敗時は完全ロールバック
- 既存テナントへのマージは `--existing-tenant` オプション時のみ（重複時は ERROR）

---

## 7. パスワード移行の取り扱い

暫定ツールには Web 版で使用する **個別ユーザーのログインパスワードは存在しない**（Excel のシート保護パスワードのみ）。

**方針**: 移行時にユーザーパスワードを **強制再設定**

1. インポート時、全ユーザーに一時パスワードトークン（`reset_token`）を発行
2. システム管理者宛に `users_password_reset.csv` を出力（ユーザーごとのリセット URL リスト）
3. メールでユーザーに通知し、初回ログイン時に新パスワード設定を強制

---

## 8. 移行手順（運用）

```
[暫定ツール側]
1. 環境設定者が F_AdminMenu → CSV エクスポート
2. 出力フォルダを Web 版担当者に共有
3. 暫定ツールの操作を停止（読み取り専用化）

[Web 版側]
4. Web 版を停止（メンテナンスモード）
5. dotnet ApprovalSystem.Importer --dry-run で検証
6. dotnet ApprovalSystem.Importer 本実行
7. 検証クエリで件数・サンプルレコードを確認
8. ユーザーパスワードリセット通知メール送信
9. Web 版稼働開始
10. 暫定ツールをアーカイブ（読み取り専用 .xlsm のまま保存）
```

---

## 9. ロールバック

インポート失敗時:
- DB はトランザクションで自動ロールバック
- 暫定ツールの読み取り専用解除（運用継続）
- 出力 CSV は破棄せず、原因調査用に保管

インポート成功後の取り消し:
- 当該テナントを `is_active=false` 化（論理削除）
- 関連データは保持（再インポート時の混乱を避けるため、既存テナントは事前に削除する運用とする）

---

## 10. 既知の制約

| 制約 | 対応 |
|---|---|
| 暫定ツールのメール送信ログは Web 版テーブルに完全対応しない（`tenant_id` 等が無い） | tenant_id を補完しつつインポート、無理な場合はスキップ |
| 操作ログのハッシュチェーン整合性が崩れている | 警告のみ表示、強制インポート可（`--ignore-hash-mismatch`） |
| Excel 上で手動編集された行 | 整合性チェックでエラーになれば該当行を除外して継続 |

---

*エクスポート機能は VBA 側 `M_Export` モジュール、インポート機能は別途 `ApprovalSystem.Importer` プロジェクトで実装する。*
