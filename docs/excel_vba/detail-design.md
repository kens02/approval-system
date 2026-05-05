# Excel VBA 暫定ツール 詳細設計書

**バージョン**: 1.0  
**作成日**: 2026-05-01

---

## 1. VBA プロジェクト構成

### 1.1 モジュール一覧

```
ApprovalSystem.xlsm (VBAProject)
├ ThisWorkbook                      ' Workbook イベント
├ シートモジュール (Sheet1, Sheet2, …) ' シート固有イベント
├ 標準モジュール
│   ├ M_Main                        ' エントリポイント・ボタンハンドラ
│   ├ M_Workflow                    ' ステータス遷移ロジック
│   ├ M_Application                 ' 申請CRUD
│   ├ M_Notification                ' メール送信
│   ├ M_Auth                        ' パスワード認証
│   ├ M_Sheet                       ' シート保護解除/再保護ヘルパ
│   ├ M_LeaveBalance                ' 残日数計算
│   ├ M_Settings                    ' 環境設定アクセス
│   ├ M_Util                        ' GUID, 日付フォーマット, ハッシュ
│   ├ M_Lock                        ' ロックファイル制御
│   ├ M_Export                      ' CSV エクスポート
│   ├ M_Import                      ' CSV インポート（補助）
│   ├ M_Print                       ' 印刷
│   └ M_Debug                       ' デバッグモード（debug-mode.md 参照）
├ クラスモジュール
│   ├ C_Application                 ' 申請ドメインオブジェクト
│   ├ C_LeaveDetail                 ' 休暇詳細
│   ├ C_RouteStep                   ' ルートステップ
│   ├ C_AuthSession                 ' 認証セッション
│   └ C_Logger                      ' 操作ログ書き込み
└ UserForm
    ├ F_PasswordPrompt              ' パスワード入力
    ├ F_DecisionDialog               ' 承認/差戻のコメント入力
    ├ F_ApplicationCard              ' カード入力（後続実装）
    ├ F_AdminMenu                    ' 環境設定者メニュー
    └ F_DebugMailPreview             ' デバッグモード時のメール送信プレビュー
```

### 1.2 命名規則

| 種別 | プレフィックス | 例 |
|---|---|---|
| 標準モジュール | `M_` | `M_Workflow` |
| クラスモジュール | `C_` | `C_Application` |
| UserForm | `F_` | `F_PasswordPrompt` |
| 定数 | 大文字スネーク | `SHEET_APPLICATIONS` |
| 変数 | キャメル | `applicationId` |
| 関数（Public） | パスカル | `SubmitApplication` |
| 関数（Private） | アンダースコア | `private_validate` |

---

## 2. 主要モジュール詳細

### 2.1 `ThisWorkbook`

```vb
Private Sub Workbook_Open()
    On Error GoTo Failed
    
    ' 1) ロックファイル制御
    If Not M_Lock.AcquireLock() Then
        ThisWorkbook.ChangeFileAccess Mode:=xlReadOnly
        MsgBox "他のユーザーが編集中のため読み取り専用で開きました", vbInformation
    End If
    
    ' 2) 環境設定キャッシュ
    M_Settings.LoadCache
    
    ' 3) シート保護を再適用（万一解除されている場合の保険）
    M_Sheet.EnsureProtections
    
    ' 4) 操作ログのハッシュチェーン検証（任意）
    If M_Settings.GetBool("audit.verify_on_open", True) Then
        If Not M_Audit.VerifyLogChain Then
            MsgBox "操作ログに改ざんの疑いがあります。環境設定者に連絡してください。", vbExclamation
        End If
    End If
    
    ' 5) ダッシュボードを表示
    Worksheets("ダッシュボード").Activate
    Exit Sub
Failed:
    MsgBox "起動処理でエラー: " & Err.Description, vbCritical
End Sub

Private Sub Workbook_BeforeClose(Cancel As Boolean)
    M_Lock.ReleaseLock
End Sub
```

### 2.2 `M_Auth`（認証）

```vb
Public Enum AuthLevel
    AuthLevel_None = 0
    AuthLevel_Decision = 1
    AuthLevel_Admin = 2
End Enum

Public Function RequireAuth(level As AuthLevel) As Boolean
    ' デバッグモード中は無条件で認証成功とする（debug-mode.md 参照）
    If M_Debug.IsDebugMode Then
        AuthSession.AuthorizedAt = Now
        AuthSession.DecisionAuthorized = True
        AuthSession.AdminAuthorized = True
        RequireAuth = True
        Exit Function
    End If
    
    ' 既存セッションを確認
    If IsAuthorized(level) Then
        RequireAuth = True
        Exit Function
    End If
    
    ' プロンプト表示
    Dim caption As String
    caption = IIf(level = AuthLevel_Admin, "環境設定パスワード", "決裁パスワード")
    
    Dim plain As String
    plain = F_PasswordPrompt.Prompt(caption)
    If plain = vbNullString Then Exit Function
    
    ' 検証
    Dim salt As String, expected As String, actual As String
    salt = M_Settings.GetString("password.salt")
    expected = IIf(level = AuthLevel_Admin, _
                   M_Settings.GetString("password.admin"), _
                   M_Settings.GetString("password.decision"))
    actual = M_Util.HashPassword(plain, salt)
    
    If actual = expected Then
        AuthSession.AuthorizedAt = Now
        If level = AuthLevel_Admin Then AuthSession.AdminAuthorized = True
        AuthSession.DecisionAuthorized = True
        AuthSession.FailCount = 0
        RequireAuth = True
    Else
        AuthSession.FailCount = AuthSession.FailCount + 1
        MsgBox "パスワードが違います", vbExclamation
        If AuthSession.FailCount >= 3 Then
            AuthSession.LockUntil = DateAdd("n", 5, Now)
            MsgBox "認証ロック: 5分間操作不可", vbCritical
        End If
    End If
End Function

Public Function IsAuthorized(level As AuthLevel) As Boolean
    Const TIMEOUT_MIN As Long = 30
    If AuthSession.AuthorizedAt = 0 Then Exit Function
    If DateDiff("n", AuthSession.AuthorizedAt, Now) > TIMEOUT_MIN Then
        ResetSession
        Exit Function
    End If
    Select Case level
        Case AuthLevel_Decision: IsAuthorized = AuthSession.DecisionAuthorized Or AuthSession.AdminAuthorized
        Case AuthLevel_Admin:    IsAuthorized = AuthSession.AdminAuthorized
    End Select
End Function

Public Sub ResetSession()
    AuthSession.AuthorizedAt = 0
    AuthSession.DecisionAuthorized = False
    AuthSession.AdminAuthorized = False
End Sub
```

### 2.3 `M_Workflow`（状態遷移）

```vb
Public Sub Submit(appId As String)
    If Not M_Auth.RequireAuth(AuthLevel_Decision) Then Exit Sub
    
    Dim app As C_Application: Set app = M_Application.Load(appId)
    If app.Status <> "draft" Then Err.Raise vbObjectError + 100, , "draft 以外は提出不可"
    
    app.Status = "in_progress"
    app.CurrentStepOrder = 1
    app.SubmittedAt = Now
    app.ApplicationNumber = M_Util.GenApplicationNumber()
    
    M_Application.Save app
    M_Logger.Append app.Id, 0, "decision", CurrentUserId(), "submitted", False, ""
    
    ' ステップ1担当へ通知
    Dim step1 As C_RouteStep
    Set step1 = M_Application.GetStep(app.RouteId, 1)
    M_Notification.Send "submitted", app, Array(step1.AssigneeEmail)
End Sub

Public Sub Approve(appId As String, comment As String)
    If Not M_Auth.RequireAuth(AuthLevel_Decision) Then Exit Sub
    
    Dim app As C_Application: Set app = M_Application.Load(appId)
    AssertNotDeleted app
    AssertCanApprove app
    
    Dim isProxy As Boolean: isProxy = M_Auth.IsAuthorized(AuthLevel_Admin) And _
                                       Not IsCurrentStepAssignee(app)
    Dim totalSteps As Long: totalSteps = M_Application.CountSteps(app.RouteId)
    
    ' ログ
    M_Logger.Append app.Id, app.CurrentStepOrder, _
                    M_Application.GetStepType(app.RouteId, app.CurrentStepOrder), _
                    CurrentUserId(), "approved", isProxy, comment
    
    If app.CurrentStepOrder >= totalSteps Then
        ' 最終ステップ → 決裁
        Decide appId, comment, isProxy
        Exit Sub
    End If
    
    app.CurrentStepOrder = app.CurrentStepOrder + 1
    M_Application.Save app
    
    Dim nextStep As C_RouteStep
    Set nextStep = M_Application.GetStep(app.RouteId, app.CurrentStepOrder)
    M_Notification.Send "step_approved", app, Array(nextStep.AssigneeEmail)
End Sub

Public Sub Decide(appId As String, comment As String, isProxy As Boolean)
    Dim app As C_Application: Set app = M_Application.Load(appId)
    app.Status = "decided"
    app.DecidedAt = Now
    M_Application.Save app
    
    M_Logger.Append app.Id, app.CurrentStepOrder, "decision", _
                    CurrentUserId(), "decided", isProxy, comment
    
    Dim recipients As Collection: Set recipients = New Collection
    recipients.Add M_Application.GetApplicantEmail(app)
    Dim u As Variant
    For Each u In M_Settings.GetAdminEmails()
        recipients.Add u
    Next u
    M_Notification.SendMany "decided", app, recipients
End Sub

Public Sub Reject(appId As String, comment As String)
    If Trim(comment) = "" Then Err.Raise vbObjectError + 101, , "差し戻しコメント必須"
    If Not M_Auth.RequireAuth(AuthLevel_Decision) Then Exit Sub
    
    Dim app As C_Application: Set app = M_Application.Load(appId)
    Dim isProxy As Boolean: isProxy = M_Auth.IsAuthorized(AuthLevel_Admin) And _
                                       Not IsCurrentStepAssignee(app)
    
    ' 元行ソフトデリート
    app.Status = "rejected"
    app.IsDeleted = True
    M_Application.Save app
    M_Logger.Append app.Id, app.CurrentStepOrder, _
                    M_Application.GetStepType(app.RouteId, app.CurrentStepOrder), _
                    CurrentUserId(), "rejected", isProxy, comment
    
    ' 内容コピーで新規 draft 作成
    Dim newApp As C_Application: Set newApp = M_Application.CloneAsDraft(app)
    M_Application.Save newApp
    
    ' 申請者へ通知
    M_Notification.Send "rejected", app, Array(M_Application.GetApplicantEmail(app)), _
                        comment, newApp.Id
End Sub

Public Sub Cancel(appId As String)
    If Not M_Auth.RequireAuth(AuthLevel_Decision) Then Exit Sub
    Dim app As C_Application: Set app = M_Application.Load(appId)
    
    ' 申請者本人 or 環境設定者のみ可
    If app.ApplicantUserId <> CurrentUserId() And Not M_Auth.IsAuthorized(AuthLevel_Admin) Then
        MsgBox "取り消し権限がありません", vbExclamation: Exit Sub
    End If
    
    app.Status = "cancelled": app.IsDeleted = True
    M_Application.Save app
    M_Logger.Append app.Id, -1, "decision", CurrentUserId(), "cancelled", False, ""
    
    ' 関与済み担当者＋環境設定者へ通知
    Dim recipients As Collection: Set recipients = M_Application.GetInvolvedEmails(app)
    M_Notification.SendMany "cancelled", app, recipients
End Sub

Public Sub Close_(appId As String)
    If Not M_Auth.RequireAuth(AuthLevel_Admin) Then Exit Sub
    
    Dim app As C_Application: Set app = M_Application.Load(appId)
    If app.Status <> "decided" Then Err.Raise vbObjectError + 102, , "decided 状態のみクローズ可"
    
    app.Status = "closed"
    app.ClosedAt = Now
    app.ClosedByUserId = CurrentUserId()
    M_Application.Save app
    
    M_Logger.Append app.Id, -1, "decision", CurrentUserId(), "closed", True, ""
    M_Notification.Send "closed", app, Array(M_Application.GetApplicantEmail(app))
End Sub
```

### 2.4 `M_Sheet`（保護制御）

```vb
Public Sub UnprotectFor(ws As Worksheet, ByRef oldState As Boolean)
    oldState = ws.ProtectContents
    If Not oldState Then Exit Sub
    
    Dim pwd As String
    Select Case ws.Name
        Case "決裁ルート", "決裁ルートステップ", "ユーザーマスタ", _
             "メールテンプレート", "休暇残管理", "環境設定"
            pwd = M_Auth.GetAdminPwd_PlainSession()  ' 認証済セッションから取得
        Case Else
            pwd = COMMON_SHEET_PROTECTION_PWD          ' 定数
    End Select
    ws.Unprotect Password:=pwd
End Sub

Public Sub ReprotectIf(ws As Worksheet, oldState As Boolean)
    If Not oldState Then Exit Sub
    Dim pwd As String
    pwd = ResolveProtectionPwd(ws.Name)
    ws.Protect Password:=pwd, UserInterfaceOnly:=True, AllowFormattingCells:=False
End Sub

Public Sub WriteRow(ws As Worksheet, rowIdx As Long, values() As Variant)
    Dim oldState As Boolean
    UnprotectFor ws, oldState
    On Error GoTo Cleanup
    
    Dim i As Long
    For i = 0 To UBound(values)
        ws.Cells(rowIdx, i + 1).Value = values(i)
    Next i
    
Cleanup:
    Dim errSaved As Long: errSaved = Err.Number
    Dim errMsg As String:  errMsg = Err.Description
    ReprotectIf ws, oldState
    If errSaved <> 0 Then Err.Raise errSaved, , errMsg
End Sub
```

### 2.5 `M_Notification`（メール送信）

```vb
Public Sub Send(eventType As String, app As C_Application, recipients As Variant, _
                Optional comment As String = "", Optional newAppId As String = "")
    Dim tmpl As Object: Set tmpl = M_Settings.GetTemplate(eventType)
    Dim model As Object: Set model = BuildModel(app, comment, newAppId)
    Dim subject As String: subject = ExpandPlaceholders(tmpl("subject"), model)
    Dim body As String:    body = ExpandPlaceholders(tmpl("body"), model)
    
    Dim r As Variant
    For Each r In recipients
        On Error Resume Next
        SendOne CStr(r), subject, body, CBool(tmpl("is_html"))
        Dim status As String, errMsg As String
        If Err.Number = 0 Then
            status = "success"
        Else
            status = "failed"
            errMsg = Err.Description
        End If
        On Error GoTo 0
        M_Logger.AppendMail app.Id, eventType, CStr(r), subject, status, errMsg
    Next r
End Sub

Private Sub SendOne(toAddr As String, subject As String, body As String, isHtml As Boolean)
    ' デバッグモード時はダイアログ表示のみ（debug-mode.md 参照）
    If M_Debug.IsDebugMode Then
        M_Debug.ShowMailDialog toAddr, subject, body, isHtml
        Exit Sub
    End If
    
    Select Case M_Settings.GetString("mail.method")
        Case "outlook"
            SendViaOutlook toAddr, subject, body, isHtml
        Case "cdo"
            SendViaCdo toAddr, subject, body, isHtml
    End Select
End Sub

Private Sub SendViaOutlook(toAddr As String, subject As String, body As String, isHtml As Boolean)
    Dim ol As Object, mail As Object
    Set ol = CreateObject("Outlook.Application")
    Set mail = ol.CreateItem(0)
    With mail
        .To = toAddr
        .Subject = subject
        If isHtml Then .HTMLBody = body Else .Body = body
        .Send
    End With
End Sub

Private Function ExpandPlaceholders(template As String, model As Object) As String
    Dim s As String: s = template
    Dim k As Variant
    For Each k In model.keys
        s = Replace(s, "{" & k & "}", CStr(model(k)))
    Next k
    ExpandPlaceholders = s
End Function

Private Function BuildModel(app As C_Application, comment As String, newAppId As String) As Object
    Dim m As Object: Set m = CreateObject("Scripting.Dictionary")
    m("申請番号") = app.ApplicationNumber
    m("申請者氏名") = M_Application.GetApplicantName(app)
    m("申請種別") = "休暇申請"
    m("取得日From") = Format(app.LeaveDateFrom, "yyyy/mm/dd")
    m("取得日To") = Format(app.LeaveDateTo, "yyyy/mm/dd")
    m("フェーズ名") = M_Application.GetCurrentStepLabel(app)
    m("処理者氏名") = M_Util.GetCurrentUserName()
    m("コメント") = comment
    m("ファイルパス") = M_Settings.GetString("file.shared_path")
    m("申請ID") = app.Id
    Set BuildModel = m
End Function
```

### 2.6 `M_Lock`（ロックファイル）

```vb
Public Function AcquireLock() As Boolean
    Dim path As String: path = LockFilePath()
    If Dir(path) <> "" Then AcquireLock = False: Exit Function
    
    Dim fnum As Integer: fnum = FreeFile
    Open path For Output As #fnum
    Print #fnum, Environ("USERNAME") & ":" & Format(Now, "yyyy-mm-dd hh:nn:ss")
    Close #fnum
    AcquireLock = True
End Function

Public Sub ReleaseLock()
    On Error Resume Next
    Kill LockFilePath()
End Sub

Public Sub ForceReleaseLock()
    If Not M_Auth.RequireAuth(AuthLevel_Admin) Then Exit Sub
    ReleaseLock
    MsgBox "ロックを強制解除しました", vbInformation
End Sub

Private Function LockFilePath() As String
    LockFilePath = ThisWorkbook.Path & "\.lock"
End Function
```

### 2.7 `M_Util`（ユーティリティ）

```vb
Public Function NewGuid() As String
    NewGuid = Mid$(CreateObject("Scriptlet.TypeLib").Guid, 2, 36)
End Function

Public Function HashPassword(plain As String, salt As String) As String
    Dim utf8 As Object: Set utf8 = CreateObject("System.Text.UTF8Encoding")
    Dim sha As Object:  Set sha = CreateObject("System.Security.Cryptography.SHA256Managed")
    Dim bytes() As Byte
    bytes = sha.ComputeHash_2(utf8.GetBytes_4(salt & ":" & plain))
    HashPassword = ByteArrayToHex(bytes)
End Function

Private Function ByteArrayToHex(b() As Byte) As String
    Dim s As String, i As Long
    For i = LBound(b) To UBound(b)
        s = s & Right("0" & Hex(b(i)), 2)
    Next i
    ByteArrayToHex = LCase(s)
End Function

Public Function GenApplicationNumber() As String
    Dim ym As String: ym = Format(Now, "yyyy-mm")
    Dim k As String: k = "app.next_seq." & ym
    Dim n As Long: n = CLng(M_Settings.GetString(k, "0")) + 1
    M_Settings.SetString k, CStr(n)
    GenApplicationNumber = ym & "-" & Format(n, "0000")
End Function

Public Function CurrentUserId() As String
    ' Windows ログオン名から user_id を解決
    Dim winUser As String: winUser = Environ("USERNAME")
    CurrentUserId = M_Settings.LookupUserIdByLogin(winUser)
End Function
```

### 2.8 `C_Application`（クラス）

```vb
Option Explicit

Public Id As String
Public ApplicationNumber As String
Public ApplicantUserId As String
Public RouteId As String
Public ApplicationType As String
Public Status As String
Public CurrentStepOrder As Long
Public ParentApplicationId As String
Public IsDeleted As Boolean
Public SubmittedAt As Variant
Public DecidedAt As Variant
Public ClosedAt As Variant
Public ClosedByUserId As String
Public CreatedAt As Date
Public UpdatedAt As Date

' Leave 詳細（取り回し用に展開）
Public LeaveType As String
Public TakeUnit As String
Public LeaveDateFrom As Date
Public LeaveDateTo As Date
Public LeaveTimeFrom As Variant
Public LeaveTimeTo As Variant
Public TotalMinutes As Long
Public TotalDays As Double
Public Reason As String
Public Remarks As String
```

---

## 3. ボタン配置とハンドラ

各シートに以下のボタンを配置し、対応する `M_Main` の Sub を呼ぶ:

| シート | ボタン | 呼び出し |
|---|---|---|
| 定型用紙 | [一時保存] | `M_Main.SaveDraft` |
| 定型用紙 | [提出する] | `M_Main.SubmitFromForm` |
| 申請リスト | [承認] | `M_Main.ApproveSelected` |
| 申請リスト | [差し戻し] | `M_Main.RejectSelected` |
| 申請リスト | [決裁] | `M_Main.DecideSelected` |
| 申請リスト | [取り消し] | `M_Main.CancelSelected` |
| 申請リスト | [クローズ] | `M_Main.CloseSelected` |
| 申請リスト | [印刷] | `M_Main.PrintSelected` |
| 申請リスト | [CSV エクスポート] | `M_Main.ExportCsv` |
| ダッシュボード | [環境設定メニュー] | `F_AdminMenu.Show` |

---

## 4. エラー処理方針

```vb
On Error GoTo Failed
' ... 処理 ...
Exit Sub
Failed:
    M_Logger.LogError "M_Workflow.Approve", Err.Number, Err.Description
    MsgBox "処理に失敗しました: " & Err.Description, vbCritical
End Sub
```

- すべての Public プロシージャは `On Error GoTo` で受ける
- ユーザー向けメッセージは日本語
- 例外は操作ログに `error` として記録
- 致命的エラーで Book を終了させない（保護解除中なら必ず再保護）

---

## 5. パフォーマンス最適化

```vb
Public Sub WithFastMode(callback As Object)
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    Application.EnableEvents = False
    On Error GoTo Restore
    ' callback 実行
Restore:
    Application.EnableEvents = True
    Application.Calculation = xlCalculationAutomatic
    Application.ScreenUpdating = True
End Sub
```

CSV エクスポート、初期インポート、一覧再描画時に使用。

---

## 6. テスト方針（VBA）

| テストレベル | 方法 | 対象 |
|---|---|---|
| 単体 | `RubberDuck` または手書きテストモジュール | `M_Util`, `M_Workflow`, `M_LeaveBalance` |
| 結合 | テスト用 .xlsm を準備し、シナリオを手動実行 | 提出→承認→決裁→クローズ |
| 受入 | 環境設定者によるチェックリスト | `requirements.md` の受入基準 |

---

## 7. ソースコード管理

- VBA ソースは `src/excel_vba/` にエクスポート（`.bas` / `.cls` / `.frm`）
- 取り込みは VBE「ファイル → ファイルのインポート」または以下のマクロ:

```vb
Sub ImportAllModules()
    Dim folder As String: folder = ThisWorkbook.Path & "\src\excel_vba\"
    Dim file As String
    file = Dir(folder & "*.bas")
    Do While file <> ""
        ThisWorkbook.VBProject.VBComponents.Import folder & file
        file = Dir
    Loop
    ' .cls, .frm も同様
End Sub
```

> 取り込みマクロ実行には「VBA プロジェクトオブジェクトモデルへのアクセスを信頼する」設定が必要。

---

*シート定義は `data-model.md`、画面詳細は `screen-design.md`、認証は `access-control.md` を参照。*
