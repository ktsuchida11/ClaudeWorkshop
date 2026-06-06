# VS Code ポリシー強制 設定手順書（Windows / Copilot 安全運用）

> 目的: 非 Admin ユーザーが変更できない形で、Copilot / エージェントのセキュリティ設定を「強制(ロック)」する
> 対象読者: 端末をセットアップする管理者 / 情報システム・セキュリティ担当
> 最終確認: 2026年6月時点の VS Code エンタープライズポリシー仕様にもとづく

---

## 0. 前提と効果

**必要な権限**
- PC の **ローカル Admin 権限**(または ドメイン GPO / Intune の管理権限)
- ※ GitHub の組織/アカウントレベルの権限は **不要**。これはあくまで端末側(VS Code)のポリシー。

**前提バージョン**
- ポリシー機構は VS Code 1.69 以降。Copilot/Chat 系ポリシー(`ChatToolsAutoApprove` 等)は比較的新しい版で追加されたため、**使用中の版の ADMX に該当ポリシーが含まれるかを必ず確認**する(手順2)。

**効果**
- ポリシー値は VS Code の設定優先順位の最上位で、**default / user / workspace のいずれの設定も上書き**する。
- UI 上は「組織によって管理されています」と表示され、ロックアイコンが付き、ユーザーは変更できない。
- **プロファイルを切り替えても、別の user-data-dir で起動しても効く**(ファイルロックでは塞げない抜け道をカバーできる)。
- 制約: **ロックできるのは ADMX に定義されたポリシー対応の設定のみ**。任意の settings.json キーをロックすることはできない。

---

## 1. 配布方式の選択(3パターン)

| 方式 | 用途 | 設定手段 |
|------|------|----------|
| **A. ローカルグループポリシー** | 単一 PC / 検証 | `gpedit.msc` + ADMX |
| **B. Active Directory GPO** | ドメイン参加端末を一括 | 中央ストアに ADMX + GPMC |
| **C. Intune(クラウド管理)** | MDM のみの環境 | PowerShell 修復スクリプト(SYSTEM 実行) |

> ★ Intune 注意: `HKLM\SOFTWARE\Policies\Microsoft\*` は従来 GPO 用に予約されているため、**Intune の Settings Catalog や ADMX 取り込みでは書き込めない**。Intune では SYSTEM で動く PowerShell 修復スクリプトを使う(手順5)。

---

## 2. 事前準備：ADMX / ADML の取得と確認

1. 使用中バージョンの **VS Code zip アーカイブ**をダウンロードして展開する。
2. 展開先の `policies` フォルダを開く。`vscode.admx` と `locales/<locale>/vscode.adml`(例 `en-US/vscode.adml`)がある。
3. `vscode.admx` をテキストで開き、**自分の版で使えるポリシー名**を確認する(版により増減する)。Copilot 系(`ChatToolsAutoApprove`, `ChatAgentMode`, `ChatHooks` 等)が含まれているかをここで確認。

---

## 3. 手順A：ローカルグループポリシー（1台 / 検証用）

1. `vscode.admx` を `C:\Windows\PolicyDefinitions` にコピー。
   `vscode.adml` を `C:\Windows\PolicyDefinitions\<locale>`(例 `C:\Windows\PolicyDefinitions\en-US`)にコピー。
   ※ PolicyDefinitions への書き込みには **Admin 権限が必要**。
2. `Windows + R` → `gpedit.msc` でローカルグループポリシーエディターを開く(UAC は「はい」)。
3. **「コンピューターの構成 > 管理用テンプレート > Microsoft VS Code」** を開く。
   (ポリシーは Computer と User の両方にあるが、**全ユーザーに強制するため Computer 側**で設定する。両方設定時は Computer が優先。)
4. 手順6の推奨ポリシーを設定する。
5. コマンドプロンプトで `gpupdate /force` を実行 → **VS Code を再起動**。
6. 手順8で検証する。

---

## 4. 手順B：Active Directory GPO（複数台を一括）

1. `vscode.admx` / `vscode.adml` を **中央ストア**
   `\\<ドメイン>\SYSVOL\<ドメイン>\Policies\PolicyDefinitions\`(および `<locale>` サブフォルダ)に配置。
2. グループポリシー管理コンソール(GPMC)で対象 OU に GPO を作成・リンク。
3. **「コンピューターの構成 > 管理用テンプレート > Microsoft VS Code」** で推奨ポリシーを設定。
4. 端末で `gpupdate /force`、または次回ポリシー反映を待つ。

---

## 5. 手順C：Intune（PowerShell 修復スクリプト）

`HKLM\...\Policies` は GPO 予約のため、**Intune Management Extension が SYSTEM 権限で実行する PowerShell 修復スクリプト**で設定する。

**検出スクリプト(例 `VSCode_Detection.ps1`)** — 値が正しければ Exit 0、違えば Exit 1:

```powershell
$regpath = "HKLM:\SOFTWARE\Policies\Microsoft\VSCode"
$ok = $true
try {
  $v = Get-ItemProperty -Path $regpath -ErrorAction Stop
  if ($v.UpdateMode -ne "manual") { $ok = $false }
  # 必要に応じて他の値も検査
} catch { $ok = $false }
if ($ok) { Write-Host "Compliant"; exit 0 } else { Write-Host "Non-compliant"; exit 1 }
```

**修復スクリプト(例 `VSCode_Remediation.ps1`)**:

```powershell
$regpath = "HKLM:\SOFTWARE\Policies\Microsoft\VSCode"
if (-not (Test-Path $regpath)) { New-Item -Path $regpath -Force | Out-Null }

# 許可拡張(JSON文字列)。コロンの前に半角スペースを入れること(実装上の注意)
$allowed = '{"github" :"stable", "ms-vscode" :true, "ms-python.python" :true, "blackboxapp" :false}'
Set-ItemProperty -Path $regpath -Name "AllowedExtensions" -Value $allowed -Type String -Force

# 更新モード: none / manual / start / default
Set-ItemProperty -Path $regpath -Name "UpdateMode" -Value "manual" -Type String -Force

# テレメトリ無効化
Set-ItemProperty -Path $regpath -Name "TelemetryLevel" -Value "off" -Type String -Force
```

> ★コツ(重要): Copilot 系のブール型ポリシー(`ChatToolsAutoApprove` 等)を Intune で設定する場合、**まず 1 台で手順A(gpedit)で設定 → レジストリ `HKLM\SOFTWARE\Policies\Microsoft\VSCode` を `reg query` で確認し、実際の値名・型(REG_SZ / DWORD)を採取 → それを修復スクリプトに反映**するのが確実。ブール系は DWORD(0=無効化)になることが多いが、版差があるため実値で確認する。

---

## 6. 推奨ポリシー（Copilot を安全に使うための設定）

> 用途は「エージェントは使うが、自動承認の暴走と拡張・更新を統制したい」を想定。
> ◎=強く推奨 / ○=任意の追加ハードニング。

| 区分 | ポリシー名 | 対応する設定 | 推奨値 | 効果 |
|------|-----------|--------------|--------|------|
| ◎ | `ChatToolsAutoApprove` | `chat.tools.global.autoApprove` | Disabled(false) | **一括自動承認をロックして無効化**。権限ピッカーから Bypass Approvals / Autopilot も非表示になる。最重要。 |
| ◎ | `AllowedExtensions` | `extensions.allowed` | 承認した発行元/拡張のみ許可(JSON) | 未審査拡張・サプライチェーン混入を防止。例: `{"github":"stable","ms-vscode":true,...}` |
| ◎ | `UpdateMode` | 更新動作 | `manual` または `none`(配布をMDM管理)/ `default`(最新追従) | 更新の制御。MDM で配信するなら manual/none。 |
| ◎ | `TelemetryLevel` | テレメトリ | `off` | プロジェクト/環境メタデータの外部送信を最小化。 |
| ○ | `EnableFeedback` | フィードバック送信 | `0` | フィードバック送信プロンプトを抑止し外向き通信を削減。 |
| ○ | `ChatHooks` | `chat.useHooks` | false(フック不要時) | フック(任意シェル実行)の悪用経路を遮断。 |
| ○ | `ChatAgentMode` | `chat.agent.enabled` | false(エージェント自体を禁止する場合のみ) | エージェントを完全禁止(ask/edit は可)。**エージェントを使う方針なら設定しない**。 |
| 参考 | MCP アクセス制御系 | `chat.mcp.access` 等 | 制限 / 無効 | 信頼できない MCP 経由のプロンプトインジェクション対策。版に該当ポリシーがあればロック。 |
| 参考 | Dev Tunnels 匿名アクセス無効化 | — | 有効化 | 自動承認と匿名トンネルの併用は RCE リスク。併せて適用推奨。 |

> サンドボックス系(`chat.tools.terminal.sandbox.enabled` / `chat.agent.sandbox.enabled`)は組織レベルで管理可能だが、**Windows ネイティブでは実効性がない**(macOS / Linux / WSL2 / Dev Container でのみ有効)。Windows ではサンドボックスではなく Dev Container での隔離を前提にすること。

---

## 7. レジストリ直接設定（参考）

| 項目 | 値 |
|------|----|
| キー(Computer) | `HKLM\SOFTWARE\Policies\Microsoft\VSCode` |
| キー(User) | `HKCU\SOFTWARE\Policies\Microsoft\VSCode`(両方ある場合 Computer 優先) |
| `AllowedExtensions` | REG_SZ / JSON 文字列(コロンの前に半角スペース) |
| `UpdateMode` | REG_SZ / `none`・`manual`・`start`・`default` |
| `TelemetryLevel` | REG_SZ / `off` |
| `EnableFeedback` | DWORD / `0` |
| Chat 系ブール | gpedit で設定後に実値を採取して反映(DWORD 想定) |

---

## 8. 検証チェックリスト

- [ ] VS Code 設定 UI で対象設定に **ロックアイコン / 「組織によって管理されています」** が表示される。
- [ ] その設定を **変更しようとしても変更できない**。
- [ ] **新規プロファイルを作成しても** ポリシーが効いている(プロファイル回避不可)。
- [ ] **別の `--user-data-dir` で起動しても** 効いている。
- [ ] レジストリ: `reg query "HKLM\SOFTWARE\Policies\Microsoft\VSCode"` で値が入っている。
- [ ] GPO 適用確認: `gpresult /h result.html` で対象ポリシーが適用済み。
- [ ] (反映されない場合)VS Code の **Window ログ**(`Show Window Log`)に syntax error が出ていないか確認。

---

## 9. ロールバック

- **ローカル GP**: gpedit で該当ポリシーを「未構成」に戻す → `gpupdate /force` → VS Code 再起動。
- **AD GPO**: GPO のリンク解除、または該当ポリシーを「未構成」に。
- **Intune**: 修復スクリプトを `Remove-ItemProperty`(または該当キー削除)に差し替えて再配信。
- **ADMX 撤去**: `C:\Windows\PolicyDefinitions`(または中央ストア)から `vscode.admx` / `vscode.adml` を削除。

---

## 10. トラブルシューティング

- **設定が反映されない**: ポリシー値の syntax error の可能性。VS Code の `Show Window Log` でエラーを確認(JSON 文字列の書式ミスが多い)。
- **gpedit で ADMX 重複エラー**: 一部のプレビュー版 ADMX で「Duplicate Policy Entry」が報告されている。安定版の ADMX を使い、古い `vscode.admx` / `.adml` のコピーを除去する。
- **Intune の Settings Catalog で設定できない**: `HKLM\...\Policies` が GPO 予約のため。PowerShell 修復スクリプト(SYSTEM 実行)で設定する。
- **ユーザーがすり抜ける**: それはポリシー未適用(settings.json 配布のみ)の状態。本手順のポリシー適用後はプロファイル/データディレクトリ変更でも回避できない。

---

## 参考リンク（公式ドキュメント中心）

- VS Code: Centrally manage VS Code settings with policies(ADMX/GPO/policy.json、ローカル設定手順) — https://code.visualstudio.com/docs/enterprise/policies
- VS Code: Manage AI settings in enterprise environments(`ChatToolsAutoApprove` / `ChatAgentMode` / `ChatHooks`) — https://code.visualstudio.com/docs/enterprise/ai-settings
- VS Code: Manage extensions in enterprise environments(`AllowedExtensions`) — https://code.visualstudio.com/docs/enterprise/extensions
- VS Code: VS Code for enterprise(Intune / GPO / MDM 概要) — https://code.visualstudio.com/docs/setup/enterprise
- 既知の不具合: ADMX 重複エントリエラー(#252268) — https://github.com/microsoft/vscode/issues/252268
