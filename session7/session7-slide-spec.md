# 第7回「ユースケース」スライド文案 v4 ― 8つの型を3つに削減

**テーマ**: Claude Codeはコードを書く道具ではなく、仕事の型を実装する道具
**★ 核フレーズ（覚えて帰る一言）**: 「コードではなく『型』を実装する」 ← v2追加：表紙・S9・S11の3箇所で同じ文言を使う
**Key Takeaway**: Claude Codeの真価は「コード生成」ではなく「開発ワークフロー全体の自動化」にある
**実演**: コードベース探索 → バグ発見 → Issue登録（1本×8分）
**構成**: 前半6回の復習 → 3つのレイヤー → 5つのユースケース → ワークフロー全体像 → 実演
**参加者環境**: GitHubではなく**GitLab**を使用。GitLab CI/CDベースでユースケースを構成

---

## S1: 表紙

- タイトル: Claude Code 勉強会 第7回
- メイン: ユースケース
- サブタイトル: コードを書く道具ではなく、仕事の型を実装する道具
- 今日のテーマ: 後半「活用を広げる」フェーズの第1回
- **★ 覚えて帰る一言:「コードではなく『型』を実装する」** ← v2追加：核フレーズ
- ナビゲーション: ●●●●●●●○○○

---

## S1.5: 今日の用語ミニ辞典 ← v3追加（優先度4）

タイトル: 今日の用語ミニ辞典 ― 迷ったらここに戻る
サブタイトル: 今日のスライドで初めて出てくるカタカナ語を1行で説明

5語（ゴールド強調 ＋ 白文字説明）：

- **● Plan Mode**　＝　コードを変更せず読み取りだけで分析するモード（Shift+Tab×2）
- **● /rewind**　＝　修正前の状態に巻き戻す機能（試行錯誤を安全に）
- **● SubAgents**　＝　専門役を並列に動かす分身（独立コンテキスト）
- **● Routines**　＝　人がいなくても自動でClaude Codeが動く設定（研究プレビュー）
- **● Skill Creator**　＝　Skillを自動で作って評価するメタSkill

> 第7回は「3つのレイヤー」「Skillの3つの型（残り5型は参考）」など概念用語が連発するため、必ず最初に置く。

---

## S2: 前回の振り返り + 今日の位置（0-2分）

タイトル: 前半6回で揃った土台と、今日からの後半
サブタイトル: 3原則は「守り」。ここからは3原則を前提に「攻め」に転じる

左カラム：前半6回の地図（ICE背景カード）
- 第1回 セキュリティ ― 3原則の導入
- 第2回 環境（分ける）― DevContainer / Sandbox
- 第3回 基本ガード（防ぐ）― settings.json 1ファイル
- 第4回 自動化・共有 ― Skills / MCP / コマンド
- 第5回 AIとの仕事の型 ― バイブ→SDDへの成熟
- 第6回 ログ/トレース（残す）― Langfuse × Claude Code

右カラム：今日の位置づけ（NAVY背景カード）
- 前半6回は「**安全に使う土台**」
- 今日からは「**実際に何ができるか**」
- Claude Codeを**開発ワークフロー全体**に組み込む具体的なユースケースを見る

下部:
> 今日のゴール: Claude Codeを「コード生成ツール」ではなく「**開発ワークフローの自動化基盤**」として捉え直す。

---

## S3: Claude Codeの使い方 ― 3つのレイヤー（2-4分）★重要

タイトル: Claude Codeの使い方 ― 3つのレイヤー
サブタイトル: 対話型の補助から、ワークフロー全体の自動化へ

3段階の階段図（左=低い / 右=高い）:

**レイヤー1: 対話型（ターミナルで直接使う）** ← ネイビー背景
- 「このバグを直して」「テストを書いて」「リファクタリングして」
- 人間が指示→Claude Codeが実行→人間がレビュー
- 第5回のSDD的な使い方。**ほとんどの人がここで止まっている**

**レイヤー2: Skills & カスタムコマンド（再利用可能な型を作る）** ← レッド背景
- 第4回で学んだSkills / コマンドで「自分の仕事の型」を定義
- `/feature-spec` `/refactor-plan` `/generate-docs` のようにワークフローを部品化
- GitLab MRコメント解決など**GitLab固有のワークフローもSkillで自作可能**
- Claude Code公式ドキュメント: 「Claude Code handles the tedious tasks that eat up your day」

**レイヤー3: Routines & SubAgents（人間がいなくても動く）** ← グリーン背景
- **Routines（研究プレビュー）**: スケジュール / API / GitLabイベントでClaude Codeを自律実行
- SubAgents でタスクを並列分担（セキュリティチェック・テスト・ドキュメント更新）
- 第6回のLangfuseトレースで結果を監視・評価
- **ノートPCを閉じていても動く** ― Anthropicクラウド上で実行

下部:
> 今日はレイヤー2と3の具体的なユースケースを見る。
> **「人間がターミナルに座っている時だけ動く」から脱却する。**

---

## S4: ユースケース① コードベース探索 & デバッグ（4-6分）

タイトル: UC①: コードベース探索 & デバッグ
サブタイトル: 新しいプロジェクトに参加した日から、Claude Codeはペアプログラマーになる

左カラム：コードベース探索（3ステップ）
- **ステップ1: 全体像を掴む**
  - 「give me an overview of this codebase」でプロジェクト構造・アーキテクチャパターンを一望
  - 「what are the key data models?」「how is authentication handled?」で深掘り
- **ステップ2: 関連コードを見つける**
  - 「find the files that handle user authentication」→ 該当ファイル群を特定
  - 「trace the login process from front-end to database」→ 実行フローを追跡
  - Code Intelligenceプラグインで「Go to Definition」「Find References」がClaude Codeからも使える
- **ステップ3: Plan Modeで安全に調査**
  - `Shift+Tab×2` でPlan Mode ON → **読み取り専用で分析**（コードを変更しない）
  - 複雑なリファクタリングの事前調査、影響範囲の特定に最適
  - 「I need to refactor our auth system to use OAuth2. Create a detailed migration plan.」

右カラム：デバッグ（3ステップ）
- **ステップ1: エラーを共有する**
  - 「I'm seeing this error when I run npm test」→ スタックトレースを渡すだけでOK
  - スクリーンショットをドラッグ&ドロップしてエラー画面を共有することも可能
- **ステップ2: 原因を特定する**
  - Claude Codeがコードベースを横断して原因を追跡
  - 再現条件（間欠的か恒常的か）を伝えるとさらに精度が上がる
- **ステップ3: 修正を適用・検証する**
  - 「suggest a few ways to fix the @ts-ignore in user.ts」→ 選択肢を提示
  - 修正適用後に「run tests for the refactored code」で即座に検証
  - **Checkpoint**: 修正前の状態に戻せる。`/rewind` で安全に試行錯誤

下部:
> Claude Codeの最も日常的なユースケース。
> **「コードを書く前に、まずコードを読み解く」ところからClaude Codeが使える。**

---

## S5: ユースケース② データ変換 & 分析（6-8分）

タイトル: UC②: データ変換 & 分析
サブタイトル: CSV/Excel/JSON/DBを読み込み、変換・集計・可視化を自然言語で指示する

左カラム：Claude Codeでできるデータ処理
- **データクリーニング**: CSVの重複削除、フォーマット統一、欠損値処理をClaude Codeに自然言語で指示
  - 例: 「この売上CSVの日付列をYYYY-MM-DD形式に統一し、重複行を削除して」
- **データ変換**: JSON→CSV、Excel→DB、複数CSVのマージ等をPythonスクリプトとして自動生成・実行
  - 例: 「この3つのCSVをcustomer_idで結合し、月別売上集計テーブルを作って」
- **分析 & 可視化**: pandas + matplotlib/plotlyで集計・グラフ生成をJupyter Notebookで実行
  - 例: 「四半期別の売上トレンドと前年同期比をグラフで見せて」
- Claude Code公式: 「Researchers and data scientists at Anthropic use Claude Code extensively for data exploration, analysis, and visualization tasks」

右カラム：金融業務での活用例
- **市況データの加工**: yfinance等で取得した株価データを整形→移動平均・ボラティリティ計算
- **レポート自動生成**: データ分析結果をMarkdown/docx/pptxに自動変換（anthropics/skillsのdocument-skills活用）
- **パイプライン化**: 探索的分析コードを本番品質のデータパイプラインに変換
- Jupyter Notebookと並行作業: VS Code上でNotebookを開きながらClaude Codeが分析コードを書く

下部:
> Claude Codeの強みは「**コマンドラインで何でもできる**」こと。
> データの取得→加工→分析→レポート生成まで、ターミナルから一気通貫で実行できる。

---

## S6: ユースケース③ Skill Creator ― Skillを作り・テストし・育てる（8-10分）★実演対象

タイトル: UC③: Skill Creator ― 「型を作る型」
サブタイトル: Skillの作成→テスト→評価→改善のサイクルをClaude Code自身が回す

左カラム：Skill Creatorとは（anthropics/skills公式のメタSkill）
- **「Skillを作るSkill」** ― Claude Codeが対話しながらSkillを自動生成・テスト・改善する
- `/skill-creator` を実行するだけで、以下のサイクルが回る:
  1. **意図の確認**: 「何をするSkillか？」「いつ発動すべきか？」「出力形式は？」をインタビュー
  2. **ドラフト作成**: SKILL.mdのフロントマター+本文を自動生成
  3. **テストケース生成**: 2-3個のリアルなプロンプトを提案 → evals/evals.jsonに保存
  4. **並列評価**: SubAgentで「Skillあり」「Skillなし」を同時実行 → 結果を比較
  5. **改善**: 評価結果をもとにSkillを書き直し → 再テスト → ユーザーが満足するまで繰り返し
  6. **description最適化**: 発動精度を上げるためにdescriptionを自動調整

右カラム：Skill Creatorの実演イメージ
- **実演シナリオ**: 「コードレビューチェックリストSkill」を作る
  - 「うちのチームのレビュー基準を教えて」→ インタビュー
  - SKILL.md自動生成（Reference型）
  - テストケース2つで「Skillあり/なし」比較
  - 「セキュリティ観点が足りない」→ 修正 → 再テスト
- **ワークスペース構造**:
  ```
  review-checklist/
  ├── SKILL.md              # 自動生成されたSkill
  └── evals/
      └── evals.json        # テストケース
  review-checklist-workspace/
  ├── iteration-1/
  │   ├── eval-0/
  │   │   ├── with_skill/   # Skillありの出力
  │   │   └── without_skill/ # Skillなしの出力
  │   └── eval-1/
  └── iteration-2/          # 改善後の再テスト
  ```

下部:
> **第4回で学んだ「型を資産化する」を、Claude Code自身が自動で回す。**
> Skill Creatorは「Skillを作るのが面倒」というハードルを根本的に解消する。

---

## S7: ユースケース④ Skills活用 ― まずは3つの型から（9-11分）★実演候補

タイトル: **UC④: Skills活用 ― まずはこの3つから**
サブタイトル: Skillには8つの型があるが、最初に押さえるべきは3つ。残り5型は必要になったら参照する

★ **v4変更点**：8つの型テーブル（4行×8型）を **3つの主要型** に削減。残り5型は「他にもある」と一言触れるのみ（詳細はGitHubリファレンス参照）

---

### 上段：今日覚える3つの型（大きめのカード3枚）

**🟦 ① Reference（参照）**
- 用途：**コーディング規約・スタイルガイド** などの知識注入
- 特徴：インラインで読み込まれ、会話に自動適用される（タスク指示なし）
- 例：`api-conventions`、`team-coding-style`

**🟧 ② Task（タスク実行）**
- 用途：**デプロイ・コミット・Issue起票** などのアクション
- 特徴：`/deploy`、`/fix-issue` のように手動起動。`$ARGUMENTS` で引数を受け取る
- 例：`deploy`、`fix-issue`、`glab-mr-comments`（GitLab MRコメント取得）

**🟩 ③ Document（文書操作）**
- 用途：**docx / pdf / pptx / xlsx** の生成・編集・解析
- 特徴：anthropics/skills 公式。Claude.ai の裏側でも本番運用されている品質
- 例：`docx`、`pdf`、`pptx`、`xlsx`

---

### 下段：他に5型あるが、必要になったら参照すれば良い

> 公式リポジトリ anthropics/skills には他に5型（Visual Output / Dynamic Context / Forked Subagent / Background Knowledge / Meta Skill）が定義されている。
> **本編では深追いしない** ― 必要になったタイミングで GitHub の公式ドキュメントで型を選べばよい。
> → 詳しくは FAQ Q2-1（実際にはどの型から作り始めるべき？）

| 残り5型（参考） | 一言で |
|---|---|
| Visual Output | scripts でHTML/グラフを生成して見せる |
| Dynamic Context | 環境変数やシェル出力を Skill 内容にリアルタイム注入 |
| Forked Subagent | 重い調査をサブエージェントに分離してコンテキスト汚染を防ぐ |
| Background Knowledge | ユーザーからは見えない、Claude だけが参照する知識 |
| Meta Skill | Skill 自体を自動生成・テスト・評価する Skill（=Skill Creator） |

---

### 公式Skills の入れ方（最小コマンド）

```
/plugin marketplace add anthropics/skills
/plugin install document-skills@anthropic-agent-skills
```

→ これだけで `docx / pdf / pptx / xlsx` がすぐ使える状態になる。

下部:
> **まず3つ。Reference で規約を、Task で繰り返し作業を、Document で文書を。**
> これだけで開発ワークフローの大半が Skill 化できる。残り5型は必要になってから。

---

## S8: ユースケース⑤ SubAgents & Routines（10-12分）

タイトル: UC⑤: SubAgents & Routines ― 並列処理と自律実行
サブタイトル: 単一セッションの限界を超え、タスクを分割して専門エージェントに委任する

左カラム：SubAgents（第4回の延長）
```
Main Agent
  ├─ SubAgent A: セキュリティレビュー（Read-only）
  ├─ SubAgent B: テスト生成＋実行
  └─ SubAgent C: ドキュメント更新
```
- 各SubAgentは**独立コンテキストウィンドウ**で動作（トークン節約）
- 結果はMain Agentに報告のみ（相互通信なし）
- **大半のユースケースで十分**。Agent Teamsは第8回で詳しく

右カラム：Routines（新機能・研究プレビュー）
- **定義**: プロンプト + リポジトリ + コネクタを保存し、自動実行するClaude Code設定
- **3つのトリガー**:
  1. **スケジュール**: 毎日9時 / 毎週月曜 / 毎時 etc
  2. **API**: HTTPエンドポイントにPOST → オンデマンド実行
  3. **GitLab/GitHubイベント**: MR作成 / Issue作成 / リリース etc
- 実行はAnthropicのクラウド上 → **ノートPCを閉じていても動く**
- 例: 夜間にデプロイ後のスモークテスト → エラーログスキャン → Slackに結果投稿

下部:
> SubAgentsは「1セッション内の並列処理」。Routinesは「セッションを超えた自律実行」。
> **組み合わせれば、開発ワークフローの大部分を自動化できる。**

---

## S9: 全体像 ― 前半6回の知識が繋がる（12-13分）★核心

タイトル: 開発ワークフロー全体像 ― 全7回の知識が繋がる
サブタイトル: 各ユースケースが、これまで学んだどの機能で実現されているか

大きなフロー図（横5ステップ + 各ステップに紐づく回の番号）:

**① Issue/要件定義** → **② 実装** → **③ レビュー** → **④ マージ&デプロイ** → **⑤ 運用&改善**

各ステップの下に対応する機能:
1. Issue/要件: CLAUDE.md（第3回）、SDD Spec（第5回）、Plan Mode（本回UC①）
2. 実装: コードベース探索（本回UC①）、Skills（第4回+本回UC④）、SubAgents（本回UC⑤）、DevContainer（第2回）
3. レビュー: Plan Mode（本回UC①）、Hooks（第3回）、Skill Creator（本回UC③）で品質基準をSkill化
4. マージ&デプロイ: Routines（本回UC⑤）、Prompt CMS（第6回）
5. 運用&改善: Langfuseトレース（第6回）、データ分析（本回UC②）、LLM-as-a-Judge（第6回）

下部（ネイビー帯）:
> **これが「仕事の型を実装する」ということ。**
> Claude Codeは①〜⑤の全ステップに関わる。コードを書くのはそのうちの②だけ。

---

## S10: 実演の導入（13-15分→実演8分）

タイトル: ここからは実演
サブタイトル: コードベース探索 → バグ発見 → Issue登録をClaude Codeで一気に

実演の概要（ステップ3つ）:

**ステップ1（3分）: コードベース探索**
- プロジェクトルートで `claude` を起動
- 「give me an overview of this codebase」で全体像を把握
- 「find the files that handle authentication」で関連コードを特定
- Plan Mode（`Shift+Tab×2`）で安全に調査する様子を見せる

**ステップ2（2分）: バグ / 改善点を発見**
- 「suggest improvements for the error handling in this module」
- Claude Codeが問題点を指摘 → 影響範囲を説明
- 「これはIssueとして登録すべき内容ですね」

**ステップ3（3分）: Issue登録**
- 「create a GitLab issue summarizing this bug with reproduction steps」
- Claude Codeがタイトル・説明・再現手順・期待される挙動を整形してIssue起票
- 人間はレビューして確認するだけ

下部:
> 見せたいのは「**コードを読む → 問題を見つける → 記録する**」が全てターミナル内で完結すること。
> コードベース探索が最も日常的なClaude Codeの使い方。

---

## S11: 振り返り（Key Takeaway）

タイトル: 今日の振り返り

★ **核フレーズ（最上部・大きく強調）**:
> **★「コードではなく『型』を実装する」** ← v2追加：表紙と同じ文言

大きな一文（ネイビー背景帯）:
> **Claude Codeはコードを書く道具ではなく、仕事の型を実装する道具。**
> 3つのレイヤー（対話→コマンド→CI/CD統合）で開発ワークフロー全体を自動化する。

**3原則の現在地（最下部・ゴールド帯）** ← v3追加（優先度6）:
> 3原則の現在地：  01 分ける ✅    02 防ぐ ✅    03 残す ✅    （3原則そろった → 今日から後半「活用フェーズ」）

チェック項目（5つ）:
- □ Claude Codeの3つのレイヤー（対話/コマンド/Routines）を知った
- □ コードベース探索・デバッグでPlan Modeを活用できることを理解した
- □ データ変換・分析もターミナルから自然言語で実行できる
- □ Skill Creatorで「Skillを作る→テスト→改善」のサイクルを自動で回せる
- □ Skillの3つの主要型（Reference / Task / Document）を知り、公式Skills+自作のハイブリッドで武器を揃えられる

---

## S12: 次回予告

- タイトル: 次回 ― 第8回「Agent Teams」
- テーマ: SubAgentsは道具、Agent Teamsは組織。役割設計と評価がセット
- 取り扱う内容:
  - Agent Teams のP2Pメッセージング
  - 役割設計（FE/BE/Test等）
  - タスクリスト共有と自律的タスク選択
  - コスト管理と使い分けの判断基準
- ナビゲーション: ●●●●●●●●○○

---

## デザインノート

- 配色: Midnight Executive（NAVY / ICE / RED / WHITE）第1回〜第6回と統一
- S3の3段階階段図: レイヤー1=NAVY / レイヤー2=RED / レイヤー3=GREEN
- S4, S7にYAMLコードブロック（CODE_BG #0F1434背景）
- S9の全体フロー図が本回の核心 ― 5ステップを横に並べ、各ステップの下に回番号バッジ

---

## 30分タイムライン

| 時間 | 内容 | スライド |
|------|------|---------|
| 0-2分 | 前回振り返り + 今日の位置 | S2 |
| 2-4分 | 3つのレイヤー | S3 ★重要 |
| 4-6分 | UC① コードベース探索 & デバッグ | S4 |
| 6-8分 | UC② データ変換 & 分析 | S5 |
| 8-10分 | UC③ Skill Creator（Skillを作り・テストし・育てる） | S6 ★実演対象 |
| 10-11分 | UC④ Skills活用（まずは3つの型 ― Reference/Task/Document） | S7 |
| 11-12分 | UC⑤ SubAgents & Routines | S8 |
| 12-13分 | 全体フロー図（前半6回と接続） | S9 ★核心 |
| 13-15分 | 実演導入 | S10 |
| 15-23分 | **実演**（GitLab MRコメント一括解決） | - |
| 23-28分 | 質疑応答 | - |
| 28-30分 | 振り返り + 次回予告 | S11-S12 |

---

## 情報ソース

### Anthropic公式Skillsリポジトリ
- **anthropics/skills**: https://github.com/anthropics/skills（Star 117k+）
  - Claude Code Plugin としてインストール可能（`/plugin marketplace add anthropics/skills`）
  - **document-skills**: docx / pdf / pptx / xlsx（Claude.aiの裏側でも使われている本番品質）
  - **example-skills**: Creative（アート・音楽・デザイン）/ Technical（Webテスト・MCP生成）/ Enterprise（コミュニケーション・ブランディング）
  - **skill-creator**: Skillを自動生成・テスト・評価するメタSkill（第4回の延長で活用可能）
  - **mcp-builder**: MCPサーバーを自動生成するSkill
  - **Agent Skills仕様はオープン標準**: OpenAI Codex CLIでも同じSKILL.md形式。ベンダーロックインなし
  - **SkillsMP（コミュニティ）**: https://skillsmp.com ― 800,000+のSkillsが検索可能
  
### Claude Code公式ドキュメント
- **GitLab CI/CD連携**: https://code.claude.com/docs/en/gitlab-ci-cd
  - 公式サポート。.gitlab-ci.ymlにジョブ追加で@claudeメンション対応
  - Anthropic API / AWS Bedrock / Google Vertex AI 対応
  - MR作成・コードレビュー・Issue→実装・パフォーマンス分析
- **Routines**: https://code.claude.com/docs/en/routines（研究プレビュー）
  - スケジュール / API / GitHubイベントで自律実行
  - ドキュメント更新・デプロイ検証・障害対応自動化
- **Common Workflows**: https://code.claude.com/docs/en/common-workflows
  - コードベース探索・デバッグ・リファクタリング・テスト生成・PR/MR作成
- **Skills**: https://code.claude.com/docs/en/skills
  - カスタムコマンド作成。GitLab固有ワークフローもSkillで自作可能
- **Best Practices**: https://code.claude.com/docs/en/best-practices
  - /fix-issue Skillの例。TDD、Plan Mode、コンテキスト管理

### GitLab × Claude Code コミュニティ事例
- **GitLab MRコメント解決Skill**: GitLabにはGitHubの/pr-comments相当がない → カスタムSkillで自作（GITLAB_API_TOKEN + REST API）
- **GitLab Duo Agent Platform**: GitLabのエージェントプラットフォームにClaude外部統合可能
- **GitLab Issue #557820**: Claude Code + Codex CLI のGitLab CI/CD公式統合のPRD（機能開発中）

### 既存スライドA（未消化分）
- A-S15: SubAgents vs Agent Teams → S8
- A-S16: エージェント使い分けガイド → S8
- A-S21-23: Claude Code Actions → S4, S5（GitHub→GitLab読み替え）

**A消化追加: 5枚（A-S15, S16, S21, S22, S23）→ A消化率: 約85%（39/46枚）**
