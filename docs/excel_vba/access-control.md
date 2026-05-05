# Excel VBA 暫定ツール アクセス制御設計書

**バージョン**: 1.0  
**作成日**: 2026-05-01

---

## 1. 設計方針

本ツールは個人単位での運用を前提とするため、Web 版のような RBAC は実装せず、**3層のパスワード方式** で実用的な権限分離を行う。

### 基本原則

1. パスワードは **`環境設定` シートに集約管理**（VBA からのみ参照可能）
2. すべてのシートは **デフォルトで保護**（VBA 以外で編集不可）
3. 特権操作は VBA がシート保護を一時解除 → 処理 → 再保護する流れ
4. パスワードは **SHA-256 + salt でハッシュ化**して保存（平文禁止）
5. 認証成功はメモリ上にセッション保持し、Book を閉じるまで有効

---

## 2. 3 層パスワード制御

```
┌─────────────────────────────────────────────────────────┐
│ ① ファイル閲覧制限（Excel 標準のブック開封パスワード） │
│    Book を開く時に Excel が要求                          │
└─────────────────────────────────────────────────────────┘
                          ↓ 開封成功
┌─────────────────────────────────────────────────────────┐
│ ② 決裁パスワード制限                                   │
│    承認・決裁・差し戻し・取り消しの直前に VBA が要求    │
└─────────────────────────────────────────────────────────┘
                          ↓ 環境設定者の場合
┌─────────────────────────────────────────────────────────┐
│ ③ 環境設定パスワード制限                               │
│    環境設定・マスタ編集・クローズ・代行決裁時に要求     │
│    （①②を兼ねる最高権限）                              │
└─────────────────────────────────────────────────────────┘
```

### 2.1 各層の詳細

| 層 | 名称 | 用途 | 認証主体 | 保管場所 |
|---|---|---|---|---|
| ① | ファイル閲覧パスワード | Book を開く | Excel 本体 | **Bookのプロパティ**（VBA 管理外） |
| ② | 決裁パスワード | 承認/決裁/差戻/取消 | VBA `Auth` モジュール | `環境設定` シート（ハッシュ） |
| ③ | 環境設定パスワード | マスタ編集/クローズ/代行 | VBA `Auth` モジュール | `環境設定` シート（ハッシュ） + シート保護 |

---

## 3. シート保護の使い分け

### 3.1 シート保護パスワードの設計

| シート | シート保護パスワード | 解除方法 |
|---|---|---|
| `定型用紙` | 共通保護PW（VBA 内蔵定数） | VBA が起動時に自動 |
| `申請リスト` | 共通保護PW | 行追加・更新時のみ VBA で解除 |
| `休暇詳細` | 共通保護PW | VBA で解除 |
| `決裁ルート` | 環境設定パスワード | 環境設定者のみ VBA で解除 |
| `決裁ルートステップ` | 環境設定パスワード | 同上 |
| `ユーザーマスタ` | 環境設定パスワード | 同上 |
| `メールテンプレート` | 環境設定パスワード | 同上 |
| `休暇残管理` | 環境設定パスワード | 同上 |
| `環境設定` | 環境設定パスワード（**シート保護を二重に**） | 環境設定者のみ |
| `操作ログ` | 共通保護PW（**追記のみ**設定） | VBA で解除して append のみ |
| `メール送信ログ` | 共通保護PW（追記のみ） | 同上 |

> 「共通保護PW」は VBA モジュール内に定数で持つ難読化文字列とし、人間が知らずに済む状態にする。  
> 環境設定パスワードはユーザー入力を受け、VBA が一時的に保持する。

### 3.2 環境設定シートの二重保護

`環境設定` シートだけは **シート保護＋ブック構造保護＋セル単位の rights 制御** を多重化:

1. シートを `Worksheet.Protect Password:=adminPwd, UserInterfaceOnly:=True`
2. ハッシュ列（B列）を「Locked=True」+「Hidden=True」（数式バーにも表示しない）
3. シート自体を「非常に隠す（`xlSheetVeryHidden`）」状態にし、VBE からのみ表示可能とする

---

## 4. パスワード保管とハッシュ化

### 4.1 ハッシュ計算

```vb
Public Function HashPassword(plain As String, salt As String) As String
    Dim utf8 As Object: Set utf8 = CreateObject("System.Text.UTF8Encoding")
    Dim sha As Object: Set sha = CreateObject("System.Security.Cryptography.SHA256Managed")
    Dim bytes() As Byte
    bytes = sha.ComputeHash_2(utf8.GetBytes_4(salt & ":" & plain))
    HashPassword = ByteArrayToHex(bytes)
End Function
```

### 4.2 環境設定シートでの保管例

| setting_key | setting_value | is_secret |
|---|---|---|
| `password.salt` | `9f3ab2c1d4e5...` | TRUE |
| `password.decision` | `7a8b9c0d1e2f...` (SHA-256 hex) | TRUE |
| `password.admin` | `1a2b3c4d5e6f...` (SHA-256 hex) | TRUE |

### 4.3 認証フロー

```
ユーザーが決裁ボタン押下
   ↓
VBA: ShowPasswordPrompt("決裁パスワードを入力")
   ↓
入力された平文 plain ←━ ユーザー入力
   ↓
salt = GetSetting("password.salt")
expected = GetSetting("password.decision")
actual = HashPassword(plain, salt)
   ↓
If actual = expected Then
    AuthSession.DecisionAuthorized = True
    [操作続行]
Else
    [失敗カウント++、3回でロック（一時無効化）]
End If
```

---

## 5. セッション管理

### 5.1 認証セッション（メモリ上）

```vb
' modules/AuthSession.bas
Public Type TAuthSession
    DecisionAuthorized As Boolean
    AdminAuthorized    As Boolean
    AuthorizedAt       As Date
    FailCount          As Long
End Type
Public AuthSession As TAuthSession
```

- 認証は Book を開いている間のみ有効
- Book を閉じるとセッション破棄
- `AdminAuthorized = True` の場合、`DecisionAuthorized` も自動的に True とみなす（環境設定パスワードは決裁パスワードを兼ねる）

### 5.2 タイムアウト（推奨）

```vb
Public Function IsAuthorized(level As AuthLevel) As Boolean
    Dim TIMEOUT_MIN As Long: TIMEOUT_MIN = 30
    If DateDiff("n", AuthSession.AuthorizedAt, Now) > TIMEOUT_MIN Then
        ResetSession
        Exit Function
    End If
    ' レベルに応じて返却
End Function
```

30 分操作がなければ再認証を要求する（任意）。

### 5.3 失敗時のロック

3 回連続失敗で 5 分間ロック:
```
AuthSession.FailCount += 1
If FailCount >= 3 Then
    AuthSession.LockUntil = DateAdd("n", 5, Now)
End If
```

---

## 6. シート保護の解除・再保護パターン

VBA 内では以下のヘルパを提供する:

```vb
Public Sub WithUnprotect(ws As Worksheet, pwd As String, body As Object)
    Dim wasProtected As Boolean
    wasProtected = ws.ProtectContents
    If wasProtected Then ws.Unprotect Password:=pwd
    
    On Error GoTo ReProtect
    ' 呼び出し元で body を実行
    
ReProtect:
    If wasProtected Then
        ws.Protect Password:=pwd, UserInterfaceOnly:=True, _
                   AllowFormattingCells:=False
    End If
End Sub
```

> 例外発生時も必ず再保護されるよう `On Error GoTo` でハンドリング。

---

## 7. 操作ごとのパスワード要求マトリクス

| 操作 | ファイル閲覧 | 決裁PW | 環境設定PW |
|---|:-:|:-:|:-:|
| Book を開く | ✅ | - | - |
| 申請リスト閲覧 | ✅ | - | - |
| 新規申請作成 | ✅ | - | - |
| 申請提出 | ✅ | ✅ | - |
| 承認・合議 | ✅ | ✅ | - |
| 差し戻し | ✅ | ✅ | - |
| 決裁 | ✅ | ✅ | - |
| 取り消し（自分の申請） | ✅ | ✅ | - |
| クローズ処理 | ✅ | - | ✅ |
| 代行決裁 | ✅ | - | ✅ |
| ユーザーマスタ編集 | ✅ | - | ✅ |
| 決裁ルート編集 | ✅ | - | ✅ |
| メールテンプレート編集 | ✅ | - | ✅ |
| 環境設定編集 | ✅ | - | ✅ |
| 残日数手動修正 | ✅ | - | ✅ |
| パスワード変更 | ✅ | - | ✅（旧PWでの認証） |
| CSV エクスポート | ✅ | - | ✅ |
| ロック強制解除 | ✅ | - | ✅ |

---

## 8. パスワード変更フロー

```
[環境設定者が「パスワード変更」メニューを実行]
   ↓
[現在の環境設定パスワードを入力] ─→ 認証
   ↓
[変更対象を選択: 決裁PW / 環境設定PW]
   ↓
[新パスワード（2回）を入力]
   ↓
[VBA: 新 salt 生成 → 新ハッシュ計算]
   ↓
[環境設定シートを更新]
   ↓
[シート保護パスワード（環境設定PW変更時）も VBA で再保護]
```

> 環境設定パスワード変更時は、`環境設定` シート保護も新 PW で再 Protect する必要がある。

---

## 9. 監査・改ざん検知

### 9.1 操作ログのハッシュチェーン

`操作ログ` シートの各行に `hash` 列を持たせる:
```
hash[i] = SHA-256( hash[i-1] + log_id[i] + application_id[i] + action[i] + action_at[i] + assignee_user_id[i] )
```

起動時または環境設定者操作時にチェーン整合性を検証する。

### 9.2 検知時の動作

- ログに不整合があった場合、警告ダイアログを表示
- 環境設定者にメール通知（任意）
- 決裁機能を一時停止し、調査を促す

---

## 10. 限界事項（明示）

Excel VBA の性質上、以下は **完全には防げない**:

| リスク | 対策レベル | 補足 |
|---|---|---|
| Excel/VBA に詳しい者がパスワード回避 | △ | VBA プロジェクト保護＋シート保護でハードルを上げる程度 |
| バイナリ解析によるパスワード抽出 | × | サードパーティツールで突破可能（許容前提） |
| マクロ無効化による保護バイパス | △ | 起動時にシート上に大きな警告を表示 |
| 不正コピー | × | 共有フォルダ運用で検知のみ |

> 本ツールは **悪意のない一般ユーザーの誤操作を防ぐレベル** の保護を目的とする。  
> 真に機密性が必要な情報は本ツールに載せず、Web 版稼働を待つこと。

---

## 10.5 デバッグモードによる保護緩和

開発・動作確認時のみ、すべての保護機構をまとめて緩和する **デバッグモード** を提供する。

| 影響範囲 | デバッグモード時 |
|---|---|
| 決裁パスワード認証 | スキップ |
| 環境設定パスワード認証 | スキップ |
| 共通シート保護 | 解除したまま（再保護しない） |
| 隠しシート | すべて表示（手動で再非表示も可） |
| メール実送信 | 行わない（ダイアログ表示のみ） |
| 切替時の認証 | **環境設定パスワードを必ず要求**（ショートサーキット禁止） |

> 詳細は [debug-mode.md](debug-mode.md) を参照。  
> 配布物では必ずデバッグモード OFF にすること（操作ログの `is_debug` 列で確認可能）。

---

## 11. 推奨運用

- 環境設定パスワードは **環境設定者のみが知る**（口頭または暗号化された別経路で伝達）
- 決裁パスワードは **申請対象部門で共有可**（ホワイトボード等は不可、社内チャット等で限定的に共有）
- パスワードは **3か月ごとに変更**
- 退職・異動時には **必ず変更**
- 共有フォルダの ACL も併用し、対象部門のみアクセス可とする

---

*VBA 実装の詳細は `detail-design.md` の `Auth` モジュールを参照。*
