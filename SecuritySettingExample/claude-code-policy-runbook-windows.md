# Claude Code 強制設定 フルセット 手順書（Windows / 安全運用）

> 同梱ファイル:
> - `managed-settings.json` … 強制(ロック)用。非 Admin は変更不可。
> - `claude-code-project-settings.json` … チーム配布用(リポジトリにコミット)。`.claude/settings.json` として保存。
> 対象読者: 端末をセットアップする管理者 / 情シス・セキュリティ担当
> 最終確認: 2026年6月時点の Claude Code 仕様にもとづく(週次更新のため導入前に公式ドキュメントで再確認)

---

## 0. 2層構成（VS Code と同じ考え方）

| ファイル | 役割 | VS Code での対応物 |
|----------|------|--------------------|
| `managed-settings.json`(+ レジストリ) | **強制(ロック)**。最上位の優先順位で、ユーザー/プロジェクト/CLI 引数のいずれでも上書き不可。 | ポリシー(ADMX / GPO) |
| `.claude/settings.json`(プロジェクト) | **配布**。リポジトリにコミットして全員に共通の安全側ガードレールを配る(強制ではない)。 | `.vscode/settings.json` |

Claude Code の設定優先順位(高→低)は: **managed settings → CLI 引数 → local(`.claude/settings.local.json`) → project(`.claude/settings.json`) → user(`~/.claude/settings.json`)**。managed が最上位で、**CLI 引数でも上書きできません**。

配列値(`permissions.deny` や `sandbox.network.allowedDomains` 等)はスコープ間で **連結＆重複排除でマージ**されます。つまり managed の deny は常に有効で、プロジェクト/ユーザーが消すことはできません。

---

## 1. 前提と効果

**必要な権限**: PC の **ローカル Admin**(または ドメイン GPO / Intune)。GitHub の組織権限は不要(`managed-settings.json` 配置とレジストリ書き込みは端末側の操作)。

**Windows でロックが効く理由**: 配置先(`C:\Program Files\ClaudeCode\` や `HKLM`)は **Admin でないと書き込めない**ため、非 Admin ユーザーは設定を変更できない。プロファイル切り替えや別ディレクトリ起動でも効く。

**確認**: セッション内で `/status` を実行 → 「Setting sources」行に読み込まれたソースが表示される。managed が効いていれば `(HKLM)` `(file)` `Enterprise managed settings (remote)` などの配信チャネルが括弧で示される。

---

## 2. サンドボックスの重大な注意（Windows 固有・必読）

Claude Code のサンドボックスは **macOS / Linux / WSL2 のみ対応で、ネイティブ Windows は非対応**です。これが今回の設計で最も重要な制約になります。

- **サンドボックスが無い(=ネイティブ Windows)と、deny ルールは Claude の組み込みツールしか守れず、Bash 経由はすり抜ける**。例: `Read(~/.ssh/**)` の deny は Read ツールを止めるが、`Bash(cat ~/.ssh/id_rsa)` は防げない(Bash パターンの deny でも、引用符トリック等で回避され得る)。OS レベルでファイル/ネットワークを横断的に強制するのがサンドボックスで、それが無ければ「部分的な緩和」にとどまる。
- したがって**安全な構成は「WSL2 の中で Claude Code を動かす」**(bubblewrap + socat を入れてサンドボックス有効)。
- 同梱の `managed-settings.json` では **`sandbox.failIfUnavailable` を `false`** にしてあります。これは「ネイティブ Windows でも(弱いが)起動はできる」安全側の既定です。
  - ★**全員が WSL2 に移行できたら `failIfUnavailable` を `true` に変更**してください。サンドボックスが起動できない場合にエラーで停止する「ハードゲート」になります。
  - ★ただし `true` のままネイティブ Windows で起動すると **Claude Code が起動を拒否**します(=事実上 WSL 強制)。移行前に `true` にしないこと。

---

## 3. WSL2 + サンドボックスのセットアップ（推奨構成）

1. 管理者 PowerShell で WSL2 を導入: `wsl --install`(再起動)。
2. WSL ディストリ内で依存パッケージを導入: `sudo apt install bubblewrap socat`(Debian/Ubuntu)。
3. WSL 内に Claude Code を導入し、`claude` を WSL 側で実行。
4. WSL からも Windows 側のポリシーを効かせるため、`managed-settings.json` の `wslInheritsWindowsSettings: true`(同梱済み)を使う。これにより WSL 上の Claude Code が `HKLM` レジストリ + `C:\Program Files\ClaudeCode\managed-settings.json` を読みます(Windows 側が優先)。
   - ※ HKCU ポリシーも WSL に効かせたい場合は、HKCU 側にも同フラグが必要。
5. セッション内 `/sandbox` の Dependencies タブで bubblewrap / socat / ripgrep / seccomp が利用可能かを確認。

---

## 4. managed-settings.json の配置（強制レイヤー）

配信方式は3通り。`HKLM\SOFTWARE\Policies\*` は従来 GPO 用に予約されているため、Intune の Settings Catalog では書けない点に注意(VS Code と同じ制約)。

### A. 単一 PC / 検証
- ファイルを **`C:\Program Files\ClaudeCode\managed-settings.json`** に配置(要 Admin)。
  - ★旧パス `C:\ProgramData\ClaudeCode\managed-settings.json` は **v2.1.75 で廃止**。必ず `C:\Program Files\ClaudeCode\` に置くこと。
- 複数チームで断片を配るなら、同じディレクトリの **`managed-settings.d\`** に `10-telemetry.json` `20-security.json` のように置く(英数字順にマージ、後勝ち。配列は連結)。

### B. レジストリ（MDM / OS ポリシー、ファイルより上位）
- **`HKLM\SOFTWARE\Policies\ClaudeCode`** に値名 **`Settings`**(REG_SZ または REG_EXPAND_SZ)を作成し、`managed-settings.json` の中身(JSON 文字列)を格納。
- ユーザー単位は `HKCU\SOFTWARE\Policies\ClaudeCode`(ポリシー優先度は最低、Admin ソースが無い時のみ)。
- PowerShell 例:
  ```powershell
  $regpath = "HKLM:\SOFTWARE\Policies\ClaudeCode"
  if (-not (Test-Path $regpath)) { New-Item -Path $regpath -Force | Out-Null }
  $json = Get-Content "C:\path\to\managed-settings.json" -Raw
  Set-ItemProperty -Path $regpath -Name "Settings" -Value $json -Type String -Force
  ```

### C. ドメイン GPO / Intune（複数台）
- **GPO**: グループポリシー設定(プリファレンス)のファイル配置アクションで `C:\Program Files\ClaudeCode\managed-settings.json` を配布。または上記レジストリ値を GPO で配信。
- **Intune**: `HKLM\Policies` が予約のため Settings Catalog 不可 → ① Win32 アプリでファイルを配置、または ② SYSTEM 実行の PowerShell 修復スクリプトで上記レジストリ値/ファイルを設定。
- Anthropic が Jamf / Kandji / Intune / Group Policy 向けの**配布テンプレート**を公開: `github.com/anthropics/claude-code/tree/main/examples/mdm`(これをベースに調整するのが早い)。

### 優先順位（managed の中）
server-managed(Anthropic 管理コンソール) > MDM/OS ポリシー(HKLM 等) > ファイル(`managed-settings.d/*.json` + `managed-settings.json`) > HKCU。**いずれか1ソースのみ採用**(階層をまたいでマージはしない。ファイル層内ではドロップインと本体をマージ)。

---

## 5. managed-settings.json の中身（各設定の意味）

| キー | 効果 |
|------|------|
| `permissions.defaultMode: "default"` | 未マッチのツールは毎回確認(自動許可しない)。 |
| `permissions.disableBypassPermissionsMode: "disable"` | **`--dangerously-skip-permissions`(全権限バイパス)を封じる**。VS Code の一括自動承認ロックに相当。 |
| `permissions.allowManagedPermissionRulesOnly: true` | **ユーザー/プロジェクトが allow/ask/deny を定義できない**。managed のルールだけが効く(最も厳格)。※プロジェクト独自の deny も使いたい場合は `false` に。`false` でも managed の deny は常に有効で deny は allow に優先。 |
| `permissions.allow / ask / deny` | deny → ask → allow の順で**最初に一致したルールが勝つ**。安全な読み取り系を allow、書き込み・commit を ask、破壊的コマンド・機微ファイルを deny。 |
| `disableAutoMode: "disable"` | auto モードの起動を禁止(Shift+Tab サイクルから除外)。 |
| `sandbox.enabled: true` | bash サンドボックス有効(macOS/Linux/WSL2)。 |
| `sandbox.failIfUnavailable: false` | サンドボックス不可時も起動(警告のみ)。**WSL2 移行後に `true`(ハードゲート)へ。** |
| `sandbox.autoAllowBashIfSandboxed: true` | サンドボックス内の bash を自動承認(承認疲れ対策。隔離されているので安全)。 |
| `sandbox.allowUnsandboxedCommands: false` | `dangerouslyDisableSandbox` の抜け穴を無効化(必ずサンドボックス内 or `excludedCommands` のみ)。 |
| `sandbox.filesystem.denyRead/denyWrite` | OS レベルで機微パスの読み書きを遮断(Bash 含む全コマンドに有効)。 |
| `sandbox.network.allowedDomains + allowManagedDomainsOnly: true` | **許可ドメインのみ送信可**。user/project のドメインは無視(managed のみ)。 |
| `strictPluginOnlyCustomization: true` | skills/agents/hooks/MCP を **user/project 由来から遮断**し、plugin か managed 由来のみに(サプライチェーン対策。v2.1.82+)。 |
| `strictKnownMarketplaces: []` | プラグインマーケットプレイスの追加を**全面禁止**(空配列=ロックダウン)。許可制にするには許可元を列挙。 |
| `allowManagedMcpServersOnly: true` + `allowedMcpServers: []` | MCP サーバーを managed 許可リストのみに限定し、許可リスト空=**MCP を実質禁止**。使う場合は `allowedMcpServers` に `{ "serverName": "github" }` 形式で列挙。 |
| `wslInheritsWindowsSettings: true` | WSL 上の Claude Code が Windows 側のポリシーチェーンを読む(WSL 運用に必須)。 |

### 任意の追加ハードニング（必要に応じ managed に追記）
- `forceLoginMethod`: `"claudeai"` か `"console"` でログイン方式を固定(★値を誤ると全員ログイン不可になるので、自組織の認証方式を確認してから設定)。
- `forceLoginOrgUUID`: 自社の Anthropic 組織 UUID に限定(UUID は要確認)。
- `disableRemoteControl: true` / `disableAgentView: true`: リモート操作・バックグラウンドエージェントを無効化。
- `disableSkillShellExecution: true`: スキル/コマンド内のインラインシェル実行を無効化。
- `claudeMd`: 組織共通の CLAUDE.md を注入(managed のみ honored)。
- `companyAnnouncements`: 起動時の社内アナウンス表示。
- `minimumVersion` / `autoUpdatesChannel: "stable"`: バージョンの下限固定・安定チャネル追従。

---

## 6. 検証チェックリスト

- [ ] `/status` の「Setting sources」に `(file)` または `(HKLM)` が出る。
- [ ] `claude --dangerously-skip-permissions` が**拒否される**(`disableBypassPermissionsMode` 有効)。
- [ ] deny したファイル(例 `.env`)を Read させようとして**ブロックされる**。
- [ ] WSL2 で `/sandbox` が有効(Dependencies が揃っている)。
- [ ] WSL からも Windows 側ポリシーが効いている(`wslInheritsWindowsSettings`)。
- [ ] ユーザーが `~/.claude/settings.json` を編集しても managed が優先される。

---

## 7. ロールバック

- ファイル方式: `C:\Program Files\ClaudeCode\managed-settings.json`(および `managed-settings.d\` の該当ファイル)を削除。
- レジストリ方式: `Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\ClaudeCode" -Name "Settings"`。
- GPO/Intune: 該当ポリシー/スクリプトを未構成・削除に変更して再配信。

---

## 8. チューニングの指針

- **段階導入**: まず deny を緩めに + フック(PreToolUse)で監査ログを取り、実際の利用を観測してから締める(approval fatigue と生産性低下を避ける)。
- **WSL 移行後に強化**: `sandbox.failIfUnavailable: true` でサンドボックス必須化。
- **プロジェクト独自ルールも使いたい**場合: `allowManagedPermissionRulesOnly` を `false`(managed の deny は引き続き有効)。
- **MCP を使う**場合: `allowedMcpServers` に許可サーバーを列挙、または `deniedMcpServers` で個別ブロック。

---

## 参考リンク（公式）

- Claude Code settings(スコープ・優先順位・全設定キー) — https://code.claude.com/docs/en/settings
- Claude Code sandboxing(プラットフォーム対応・依存・filesystem/network) — https://code.claude.com/docs/en/sandboxing
- Claude Code permissions(ルール構文・managed-only 設定) — https://code.claude.com/docs/en/permissions
- MDM 配布テンプレート(Jamf / Kandji / Intune / Group Policy) — https://github.com/anthropics/claude-code/tree/main/examples/mdm
