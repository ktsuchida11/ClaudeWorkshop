# 第3回「基本ガード」スライド文案 v2（確定版）

**テーマ**: 3原則の【02 防ぐ】の実践編
**Key Takeaway**: Permission で境界、Sandbox で OS 壁、Hooks で強制、ネットワーク allowlist で接続先限定。settings.json 1ファイルに全部書ける。
**実演**: 全部入れた状態で1つだけ見せる（8分）
**構成**: 個別機能を先に説明 → 最後に統合 settings.json を見せる流れ

---

## S1: 表紙

- タイトル: Claude Code 勉強会 第3回
- メイン: 基本ガード
- サブタイトル: settings.json で「防ぐ」を実装する
- 今日のテーマ: 3原則の【02 防ぐ】の実践編
- ナビゲーション: ●●●○○○○○○○

---

## S2: 前回の振り返り + 今日の位置（0-2分）

- 3原則を再掲、02「防ぐ」を赤枠ハイライト
- 今日のゴール:
  > 4つの防御レイヤー（Permission / Sandbox / Hooks / ネットワーク allowlist）を理解し、
  > 統合 settings.json をプロジェクトにコピーして使えるようになる

---

## S3: settings.json で「防ぐ」― 4つの防御レイヤー（2-4分）

4列カード:

| Permission | Sandbox | Hooks | ネットワーク allowlist |
|---|---|---|---|
| Allow/Denyで境界 | ファイル+ネットワーク分離 | ライフサイクルで強制 | 接続先を限定 |
| Claude Code が解釈 | **OS レベル強制** | **決定的(100%)** | **OS レベル強制** |

脅威カテゴリとの対応:

| 脅威 | 対応レイヤー |
|---|---|
| 入口（汚染を入れない） | Permission Deny + Hooks PreToolUse |
| 出口（意図しない操作） | Sandbox filesystem + Permission Deny |
| 境界（外に拡がらない） | **ネットワーク allowlist** + Sandbox |

下部: CLAUDE.md はこれらの「読み込み指針」。settings.json が実際の「壁」。

---

## S4: Permission ― Allow / Deny で行動範囲を定義（4-6分）

設定例:
```json
"permissions": {
  "allow": ["Bash(git *)", "Bash(npm run *)"],
  "deny":  ["Bash(rm -rf *)", "Bash(curl *)", "Read(.env*)"]
}
```

**重要ポイント（赤枠で強調）**:

> ⚠️ **Permission の Deny だけでは不十分な理由**
>
> `/sandbox` なしの場合：
> - Deny は Claude の**組み込みツール（Read/Edit）のみ**をブロック
> - **Bash コマンドは Deny を回避できる**（`cat .env` 等で読めてしまう）
> - **ファイルシステム制御（denyRead/allowRead）は Sandbox 有効時のみ動作**
>
> → **Permission と Sandbox は必ずセットで使う。片方だけでは穴がある。**

補足: Enterprise > Project > User の優先順位。Bypass モードでも Deny は有効（ただし Sandbox は無効化される → S5参照）

---

## S5: Sandbox ― OS レベルの壁（6-8分）

左：ファイルシステム制御
```json
"sandbox": {
  "enabled": true,
  "filesystem": {
    "denyRead": ["~/"],
    "allowRead": ["."],
    "allowWrite": ["/tmp"]
  }
}
```

**🔴 重要**: `sandbox.filesystem` の設定は **Sandbox が有効 (`"enabled": true`) でなければ一切機能しない**。settings.json にファイルシステム制御を書いても、`/sandbox` が無効なら**ただの飾り**になる。

右：ネットワーク allowlist
```json
"sandbox": {
  "network": {
    "allowedDomains": [
      "github.com",
      "registry.npmjs.org",
      "api.anthropic.com"
    ]
  }
}
```

許可ドメイン以外への接続をOSレベルでブロック。子プロセスも同じ制限を継承。

**Bypass モードに注意（赤枠ブロック）**:

> Bypass モードは全確認をスキップし、**Sandbox の制約も回避**して実行するモード。
> 便利だが、Sandbox + Permission の壁が全て無効化される。
>
> **対策**: `sandbox.failIfUnavailable: true` を設定すると、
> Sandbox が起動できない環境（Bypass含む）では **Claude Code 自体が停止**する。
> → エンタープライズ向けには必須設定。
>
> ```json
> {
>   "sandbox": {
>     "enabled": true,
>     "failIfUnavailable": true
>   }
> }
> ```

下部: 「Sandbox なしの Permission Deny = お願いレベル / Sandbox ありの Permission Deny = 法律レベル」

---

## S6: Hooks ― 100%実行保証のガードレール（8-10分）

- ライフサイクル: SessionStart → UserPromptSubmit → PreToolUse → PostToolUse → PreCompact → Stop
- exit code: 0 = 許可 / 2 = ブロック
- 4種ハンドラ: command / http / prompt / agent
- ユースケース: 品質ゲート / セキュリティ / 監査ログ
- httpハンドラ利用時は allowedHttpHookUrls で制限

---

## S7: CLAUDE.md ― エージェントの「行動指針」（10-11分）

- settings.json が壁なら、CLAUDE.md は地図
- 読み込み階層: グローバル → プロジェクト → サブディレクトリ → ローカル → rules/
- 書くべきこと: ビルドコマンド / スタイル規約 / 構造概要 / ワークフロー
- 100〜200行超えると遵守率低下 → 確実に守らせたいルールは settings.json に書く

---

## S8: 指示の追従性 ― 4つの階段（11-13分）

```
            ┌─────────────────────┐
            │ Sandbox（OS強制）    │ ← カーネルレベル
            ├─────────────────────┤
          │ Hooks（決定的）       │ ← exit 2 で100%ブロック
          ├───────────────────────┤
        │ Permission Deny         │ ← Claude Codeが解釈
        ├─────────────────────────┤   ⚠ Sandbox なしだと
      │ CLAUDE.md（助言的）       │     Bashはここを突破する
      └───────────────────────────┘
```

注釈: Permission Deny は Sandbox が有効でないと Bash に対して効かない。だから Sandbox が上にいる。
使い分け: コーディング規約 → CLAUDE.md / ツール使い方 → Skills（第4回） / セキュリティ → Hooks + Sandbox

---

## S9: 統合 settings.json ― ベストプラクティス版（13-15分）★重要

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)", "Bash(npm run *)", "Bash(npx jest *)"
    ],
    "deny": [
      "Bash(rm -rf *)", "Bash(curl *)", "Bash(wget *)",
      "Read(.env*)", "Read(**/.credentials*)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "filesystem": {
      "denyRead": ["~/"],
      "allowRead": ["."],
      "allowWrite": ["/tmp"]
    },
    "network": {
      "allowedDomains": [
        "github.com", "registry.npmjs.org",
        "api.anthropic.com"
      ]
    }
  },
  "hooks": {
    "PreToolUse": [{
      "type": "command",
      "command": ".claude/hooks/pretool-guard.sh"
    }]
  },
  "enableAllProjectMcpServers": false
}
```

カラーコード凡例: 🔴赤=Deny / 🟢緑=Allow / 🔵青=Sandbox / 🟡黄=Hooks
参考: Trail of Bits claude-code-config

---

## S10: 実演の導入 + 参考（15分）

統合 settings.json + /sandbox を全部入れた状態で Claude Code を動かす。
- rm -rf → Permission Deny + Sandbox でダブルブロック
- .env 読み込み → denyRead でOSレベルブロック
- 許可外ドメインへの curl → ネットワーク allowlist でブロック
- git push → Allow で通る

---

## S11: 振り返り（Key Takeaway）

> Permission で境界、Sandbox で OS 壁、Hooks で強制、ネットワーク allowlist で接続先限定。
> settings.json 1 ファイルに全部書ける。これをコピーして始めよう。

チェック項目:
- □ Permission Deny だけでは Bash を止められないことを理解した
- □ Sandbox + Permission のセット運用が必要と分かった
- □ Bypass モードの危険性と failIfUnavailable による対策を理解した
- □ 統合 settings.json をプロジェクトにコピーする準備ができた

---

## S12: 次回予告

第4回「機能2：自動化・共有」Skills / MCP / スラッシュコマンド / コンテキストエンジニアリング1
ナビゲーション: ●●●●○○○○○○

---

## デザインノート

- 配色: Midnight Executive（NAVY #1E2761 / ICE #CADCFC / RED #F96167）
- 第1回・第2回と完全に統一
- コードブロック背景: #0F1434（ダークネイビー系）
- 赤枠の警告ブロック: LIGHT_RED_BG #FDECEC + RED border
