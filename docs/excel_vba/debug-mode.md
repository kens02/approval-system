# Excel VBA 暫定ツール デバッグモード設計書

**バージョン**: 1.0  
**作成日**: 2026-05-01

---

## 1. 目的

開発・動作確認時に、運用時の保護機構（パスワード認証・シート非表示・メール送信）が
妨げとなって検証効率が下がるのを避けるため、**運用時の保護をまとめて緩和するデバッグモード** を提供する。

| 項目 | 通常モード | デバッグモード |
|---|---|---|
| ファイル閲覧パスワード | Excel 標準（**変更しない**） | Excel 標準（**変更しない**） |
| 決裁パスワード | 要求 | **要求しない**（自動 OK） |
| 環境設定パスワード | 要求 | **要求しない**（自動 OK） |
| 共通シート保護パスワード | 自動適用 | **保護を解除し、再保護しない** |
| `非常に隠す` シート | 隠したまま | **手動で表示可** |
| メール送信 | Outlook/CDO で実送信 | **ダイアログ表示のみ**（送信しない） |
| 操作ログ | 通常通り記録 | 記録（ただし `is_debug=true` 列を立てる） |
| デバッグモード切替 | - | **環境設定パスワードで認証** して切替 |

> ファイル閲覧パスワード（Excel 標準のブック開封パスワード）は **VBA で制御できない仕様** のため、デバッグモードでも変更しない。設定済みなら開封時に毎回入力が必要。

---

## 2. デバッグモード切り替え方法

### 2.1 切り替えフロー

```
[F_AdminMenu]
   └─→ 「デバッグモードを切り替える」を選択
          ↓
   [F_PasswordPrompt(環境設定パスワード)]
          ↓ 認証成功
   [F_DebugToggle ダイアログ]
      ┌────────────────────────────────────┐
      │  現在のモード: 通常モード           │
      │                                    │
      │  ☐ デバッグモードを有効にする       │
      │                                    │
      │  ［ 適用 ］  ［ キャンセル ］      │
      └────────────────────────────────────┘
          ↓ 適用
   [全シートの保護を解除]
   [非表示シートを表示]
   [環境設定シートに debug.enabled=TRUE を保存]
   [タイトルバー・ステータスバーに警告表示]
```

### 2.2 切り替え時のシート操作

**デバッグモード ON** に切り替わった瞬間に実行する処理:

1. すべてのシートを `Worksheet.Unprotect` で保護解除
2. `xlSheetVeryHidden` のシートを `xlSheetVisible` に変更
3. 環境設定シートに `debug.enabled = TRUE` を保存
4. ウィンドウタイトルに `[デバッグモード]` を付与
5. ステータスバーに警告: 「デバッグモード中です（保護無効・メール無効）」
6. 全シートの A1 セル左上に黄色の警告マーカーを表示（任意）

**デバッグモード OFF** に切り替わった瞬間の処理:

1. 環境設定シートに `debug.enabled = FALSE` を保存
2. すべてのシートを `M_Sheet.EnsureProtections` で再保護
3. 運用上隠すべきシート（環境設定など）を `xlSheetVeryHidden` に戻す
4. ウィンドウタイトルから `[デバッグモード]` を除去
5. ステータスバーをクリア
6. 警告マーカーを除去

### 2.3 起動時の挙動

`Workbook_Open` 時に `debug.enabled` を読み、その状態を引き継ぐ。
ただし起動の都度、ステータスバーに状態を表示する。

```
通常モード起動時:    [ステータスバー無表示]
デバッグモード起動時: 「⚠ デバッグモード中（パスワード/メール無効）」
```

---

## 3. 各機能のデバッグモード時の振る舞い

### 3.1 認証 (`M_Auth.RequireAuth`)

```vb
Public Function RequireAuth(level As AuthLevel) As Boolean
    ' デバッグモードなら無条件 True
    If M_Debug.IsDebugMode Then
        AuthSession.AuthorizedAt = Now
        AuthSession.DecisionAuthorized = True
        AuthSession.AdminAuthorized = True
        RequireAuth = True
        Exit Function
    End If
    
    ' 通常モード処理（既存）
    ...
End Function
```

> デバッグモード自体の切替認証は **必ず通常ロジックで認証** する（後述の `EnableDebugMode` で個別に処理）。

### 3.2 シート保護 (`M_Sheet.EnsureProtections`)

```vb
Public Sub EnsureProtections()
    If M_Debug.IsDebugMode Then Exit Sub   ' デバッグ中は何もしない
    
    Dim ws As Worksheet
    For Each ws In ThisWorkbook.Worksheets
        If Not ws.ProtectContents Then
            ws.Protect Password:=ResolveProtectionPwd(ws.Name), _
                       UserInterfaceOnly:=True
        End If
    Next ws
End Sub

Public Sub UnprotectFor(ws As Worksheet, ByRef oldState As Boolean)
    If M_Debug.IsDebugMode Then
        oldState = False         ' 元から解除されているとして扱う
        Exit Sub
    End If
    ' 通常モード処理（既存）
    ...
End Sub

Public Sub ReprotectIf(ws As Worksheet, oldState As Boolean)
    If M_Debug.IsDebugMode Then Exit Sub  ' 再保護しない
    ' 通常モード処理（既存）
    ...
End Sub
```

### 3.3 シート可視性

デバッグモード ON では、「非常に隠す」状態のシートをすべて `xlSheetVisible` に設定する。
ユーザーは VBE を経由せずに各シートを確認できる。

| シート | 通常モード | デバッグモード |
|---|---|---|
| 定型用紙 | 表示 | 表示 |
| 申請リスト | 表示 | 表示 |
| 決裁ルート | 表示 | 表示 |
| 環境設定 | **非常に隠す** | **表示** |
| ユーザーマスタ | 表示 | 表示 |
| メールテンプレート | 表示 | 表示 |
| 操作ログ | 表示（保護） | 表示（保護なし） |
| メール送信ログ | 表示（保護） | 表示（保護なし） |

> デバッグモード ON 中は、ユーザーが手動で `非常に隠す` を解除/再設定しても影響しない（OFF 時に運用既定状態へ復元）。

### 3.4 メール送信 (`M_Notification`)

```vb
Private Sub SendOne(toAddr As String, subject As String, body As String, isHtml As Boolean)
    If M_Debug.IsDebugMode Then
        M_Debug.ShowMailDialog toAddr, subject, body, isHtml
        Exit Sub                  ' 実送信はしない
    End If
    
    ' 通常モード処理（既存）
    Select Case M_Settings.GetString("mail.method")
        Case "outlook": SendViaOutlook toAddr, subject, body, isHtml
        Case "cdo":     SendViaCdo toAddr, subject, body, isHtml
    End Select
End Sub
```

メール送信ダイアログ:

```
┌────────────────────────────────────────────────────────┐
│  [DEBUG] メール送信プレビュー                          │
├────────────────────────────────────────────────────────┤
│  宛先  : suzuki@demo                                   │
│  件名  : 【承認依頼】休暇申請 - 山田 太郎              │
│  形式  : プレーンテキスト                              │
│  ────────────────────────────────────────             │
│  〔本文〕                                              │
│  ┌──────────────────────────────────────────────┐     │
│  │ 山田 太郎 さんから承認依頼が届いています。   │     │
│  │ 申請番号: 2026-05-0001                       │     │
│  │ 取得日  : 2026-05-10 09:00～11:00            │     │
│  │ ファイル: \\fileserver\share\承認決裁\...   │     │
│  └──────────────────────────────────────────────┘     │
│                                                        │
│  [ 次へ ]   [ すべて閉じる ]   [ ログ出力 ]            │
└────────────────────────────────────────────────────────┘
```

- **次へ**: 同一トリガで複数受信者がいる場合、次の受信者のプレビューへ
- **すべて閉じる**: 残りのダイアログを表示せず終了
- **ログ出力**: メール送信ログには `status=debug_skipped` で記録

### 3.5 操作ログ

デバッグモード中も操作ログは記録するが、**`is_debug` 列に `TRUE`** を立てて区別する。  
本番運用時のログと混同しないよう、後段で集計やインポート時にフィルタできるようにする。

> Web 版インポート時、`is_debug=true` の行は既定で **除外** する（`--include-debug-logs` で含めるオプションを提供）。

### 3.6 メール送信ログ

```
| log_id | application_id | event_type | to_email | subject | sent_at | status | error_message |
| GUID   | UUID           | submitted  | s@d      | 【…】   | now     | debug_skipped | (空) |
```

- `status` に `debug_skipped` という新しい値を追加

---

## 4. M_Debug モジュール（実装詳細）

```vb
' modules/M_Debug.bas
Option Explicit

Private Const KEY_DEBUG_ENABLED As String = "debug.enabled"
Private Const TITLE_SUFFIX As String = " [デバッグモード]"

Public Function IsDebugMode() As Boolean
    IsDebugMode = CBool(M_Settings.GetString(KEY_DEBUG_ENABLED, "FALSE"))
End Function

Public Sub EnableDebugMode()
    ' このプロシージャは通常認証を強制する（デバッグモード有効時はこの認証もスキップしない）
    If Not RequireAdminAuth_Strict() Then Exit Sub
    
    M_Settings.SetString KEY_DEBUG_ENABLED, "TRUE"
    UnprotectAllSheets
    ShowAllSheets
    ApplyDebugUI True
    M_Logger.AppendDebug "debug_mode_enabled"
    
    MsgBox "デバッグモードを有効化しました。" & vbCrLf & _
           "・パスワード認証を全てスキップします" & vbCrLf & _
           "・シート保護を解除します（再保護しません）" & vbCrLf & _
           "・隠しシートをすべて表示します" & vbCrLf & _
           "・メール送信はダイアログ表示のみとなります", vbInformation
End Sub

Public Sub DisableDebugMode()
    If Not RequireAdminAuth_Strict() Then Exit Sub
    
    M_Settings.SetString KEY_DEBUG_ENABLED, "FALSE"
    HideOperationalSheets
    M_Sheet.EnsureProtections
    ApplyDebugUI False
    M_Logger.AppendDebug "debug_mode_disabled"
    
    MsgBox "デバッグモードを解除しました。", vbInformation
End Sub

Public Sub ShowMailDialog(toAddr As String, subject As String, body As String, isHtml As Boolean)
    Dim f As F_DebugMailPreview
    Set f = New F_DebugMailPreview
    f.SetData toAddr, subject, body, isHtml
    f.Show vbModal
End Sub

Private Function RequireAdminAuth_Strict() As Boolean
    ' デバッグモード切替時は M_Auth.IsDebugMode のショートサーキットを使わず、必ず認証を取る
    Dim plain As String
    plain = F_PasswordPrompt.Prompt("環境設定パスワード（デバッグモード切替）")
    If plain = "" Then Exit Function
    
    Dim salt As String, expected As String, actual As String
    salt = M_Settings.GetString("password.salt")
    expected = M_Settings.GetString("password.admin")
    actual = M_Util.HashPassword(plain, salt)
    RequireAdminAuth_Strict = (actual = expected)
    
    If Not RequireAdminAuth_Strict Then
        MsgBox "パスワードが違います", vbExclamation
    End If
End Function

Private Sub UnprotectAllSheets()
    Dim ws As Worksheet
    For Each ws In ThisWorkbook.Worksheets
        On Error Resume Next
        ws.Unprotect Password:=COMMON_SHEET_PROTECTION_PWD
        ws.Unprotect Password:=M_Settings.GetString("password.admin_protect", COMMON_SHEET_PROTECTION_PWD)
        On Error GoTo 0
    Next ws
End Sub

Private Sub ShowAllSheets()
    Dim ws As Worksheet
    For Each ws In ThisWorkbook.Worksheets
        ws.Visible = xlSheetVisible
    Next ws
End Sub

Private Sub HideOperationalSheets()
    On Error Resume Next
    ThisWorkbook.Worksheets("環境設定").Visible = xlSheetVeryHidden
    On Error GoTo 0
End Sub

Private Sub ApplyDebugUI(enabled As Boolean)
    Dim w As Window: Set w = ThisWorkbook.Windows(1)
    If enabled Then
        If InStr(w.Caption, TITLE_SUFFIX) = 0 Then w.Caption = w.Caption & TITLE_SUFFIX
        Application.StatusBar = "⚠ デバッグモード中（パスワード認証/メール送信が無効です）"
    Else
        w.Caption = Replace(w.Caption, TITLE_SUFFIX, "")
        Application.StatusBar = False
    End If
End Sub
```

---

## 5. F_AdminMenu への追加

```
┌──────────────────────────────────────────┐
│   環境設定メニュー                       │
├──────────────────────────────────────────┤
│   ...（既存メニュー）                     │
│   ─────────────────────────────         │
│   ▼ 開発・動作確認                       │
│   デバッグモード切替                      │
│      現在: [ 通常 ]                      │
│      [ 切り替える ]                       │
│   ─────────────────────────────         │
│              [ 閉じる ]                  │
└──────────────────────────────────────────┘
```

`[切り替える]` 押下で `M_Debug.EnableDebugMode` または `DisableDebugMode` を呼び出す。

---

## 6. F_DebugMailPreview UserForm

### 6.1 レイアウト

```
┌────────────────────────────────────────────────────────────┐
│  [DEBUG] メール送信プレビュー                  [_][□][×] │
├────────────────────────────────────────────────────────────┤
│  宛先  : [_______________________________________]         │
│  件名  : [_______________________________________]         │
│  形式  : ○プレーン  ○HTML                                │
│  ────────────────────────────────────────                  │
│  本文  :                                                   │
│  ┌──────────────────────────────────────────────────┐     │
│  │                                                  │     │
│  │   （長文表示・折り返し・スクロール可）            │     │
│  │                                                  │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  [ クリップボードへコピー ]   [ 閉じる ]                   │
└────────────────────────────────────────────────────────────┘
```

### 6.2 主要プロシージャ

```vb
' F_DebugMailPreview
Public Sub SetData(toAddr As String, subject As String, body As String, isHtml As Boolean)
    Me.txtTo.Text = toAddr
    Me.txtSubject.Text = subject
    Me.txtBody.Text = body
    Me.optHtml.Value = isHtml
    Me.optPlain.Value = Not isHtml
End Sub

Private Sub btnCopy_Click()
    Dim s As String
    s = "宛先: " & Me.txtTo.Text & vbCrLf & _
        "件名: " & Me.txtSubject.Text & vbCrLf & _
        String(40, "-") & vbCrLf & Me.txtBody.Text
    With CreateObject("htmlfile")
        .parentWindow.clipboardData.setData "text", s
    End With
End Sub
```

---

## 7. 終了時のデバッグモード扱い

`Workbook_BeforeClose` でデバッグモードのまま閉じることを許容する。
ただしユーザーへ警告を出す:

```
[Workbook_BeforeClose]
    ↓
If IsDebugMode Then
    MsgBox "デバッグモードのまま閉じます。" & vbCrLf & _
           "次回起動時もデバッグモードで開きます。", vbInformation
End If
```

---

## 8. 開発体制との整合

| 工程 | デバッグモード活用 |
|---|---|
| VBA コード取り込み（`.bas`/`.cls`/`.frm` インポート） | 取り込み直後にデバッグモード ON にして検証 |
| シート構成の追加・編集 | デバッグモード ON で保護解除済みのまま編集可能 |
| 業務シナリオテスト（提出→承認→決裁→クローズ） | メールはダイアログで内容確認、パスワード入力スキップ |
| 配布前 | デバッグモード OFF にし、`Workbook_Open` で正しく起動することを確認 |

---

## 9. 制約と注意点

| 項目 | 注意 |
|---|---|
| デバッグモード ON のまま配布禁止 | 出荷前チェックリストに「デバッグモード OFF」を必須項目とする |
| 操作ログの混在 | `is_debug=true` 行はインポート除外がデフォルト。集計時も除外する |
| 認証ショートサーキット | デバッグモード切替自体は **必ず認証**（`RequireAdminAuth_Strict`）を通す |
| ファイル閲覧パスワード | デバッグモードでも変更しない（VBA から制御不可な仕様） |
| メール送信ログ | `debug_skipped` を多発させないため、デバッグ中は不要なら `M_Settings.GetBool("debug.skip_mail_log")` でログ出力を抑止可（任意） |

---

## 10. 受入チェック（デバッグモード）

- [ ] 環境設定パスワードでデバッグモードに切り替えできる
- [ ] デバッグモードでは決裁ボタン押下時にパスワード入力が求められない
- [ ] デバッグモードでは環境設定シートが表示される（手動で再非表示にも可）
- [ ] デバッグモードではすべてのシート保護が解除されている
- [ ] デバッグモードではメール送信時にダイアログが表示され、実送信されない
- [ ] デバッグモードを OFF にすると保護が再適用される
- [ ] デバッグモードを OFF にすると環境設定シートが再非表示化される
- [ ] ウィンドウタイトル/ステータスバーで状態が判別できる
- [ ] 操作ログに `is_debug=TRUE` で記録される
- [ ] デバッグモードのまま閉じても警告が表示される

---

*デバッグモードはあくまで開発・動作確認用。配布前に必ず OFF にすること。*
