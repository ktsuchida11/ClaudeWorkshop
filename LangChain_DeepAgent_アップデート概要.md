# LangChain v1.3 & deepagents v0.6 調査レポート

> 調査日: 2026年5月15日 / 対象: Python パッケージ
> 主要情報源: 公式 GitHub releases、`docs.langchain.com` 公式ドキュメント、`blog.langchain.com`、PyPI

---

## 1. エグゼクティブサマリー

- **LangChain v1.3 のバージョン状況**: 2026年5月6日付で `langchain==1.3.0a1` / `1.3.0a2` のアルファタグが公開され、PyPI には `langchain-1.3.0` が掲出されている(なお公式ドキュメントの Changelog ページは 2026/4/7 の deepagents v0.5 までしか追従しておらず、v1.3 の独立ブログは未公開。中身は GitHub のコミットログと Deep Agents v0.6 ブログから読み取れる)。
- **v1.3 の最大の目玉**は `stream_events(version="v3")` プロトコルが `langchain-core` から `create_agent` まで配線され、エージェント/グラフが「コンテンツブロック中心」「型付きチャンネル別射影(`run.values` / `run.messages` / `run.lifecycle` / `run.subgraphs`)」のストリームを返すようになった点。
- **v1.2 で導入済みの基盤**(`extras` 属性によるプロバイダ固有ツール、`ProviderStrategy` の strict schema 対応)は維持。1.2 → 1.3 の差分は主に**ストリーミング刷新**と **HITL ミドルウェアへの `respond` 決定追加**、**ツール dispatch の perf 改善**。
- **破壊的変更は限定的**。v1.x 系は「2.0 まで破壊的変更なし」を公約しているため、移行のキモは `stream_events` の v1/v2 → v3 への切替判断、`langchain-classic` への退避済みレガシー API、Google GenAI 4.0 への移行、`langchain.hub` の非推奨化など。
- **deepagents v0.6**(2026年5月13日リリース、PyPI に `deepagents-0.6.1` 掲出)は「性能と長時間運用」をテーマにした大型リリース。**Code Interpreter (QuickJS) ミドルウェア / ハーネスプロファイル / ストリーミング v3 / DeltaChannel / ContextHubBackend** の5本柱を追加。
- deepagents は「Claude Code 風のエージェントハーネス」で、`pip install deepagents` で `create_deep_agent()` を呼ぶだけで、planning / 仮想ファイルシステム / sub-agents / 自動要約 を持つエージェントが立ち上がる。LangGraph 上に構築されコンパイル済みグラフを返すため、Studio・チェックポインタ・ストリーミングをそのまま使える。

---

## 2. LangChain v1.3 変更点

### 2.1 概要

LangChain v1.0(2025/10/20)で「create_agent + Middleware を中核に据え、レガシーは `langchain-classic` に退避」というメジャー再設計が完了した後、以下のリリースカデンスで進化している。

| バージョン | 日付 | 主要トピック |
|---|---|---|
| v1.0.0 | 2025/10/20 | 初メジャー。`create_agent`、Middleware、Content Blocks、`langchain-classic` 分離 |
| v1.1.0 | 2025/11/25 | Model Profiles、Summarization middleware、`SystemMessage` を `create_agent` に直接渡せるように、Model Retry middleware、OpenAI Content Moderation |
| `langchain-google-genai` v4.0.0 | 2025/12/8 | Google 統合 SDK へリライト。`langchain-google-vertexai` の一部を非推奨化 |
| v1.2.0 | 2025/12/15 | ツールの `extras` 属性、Anthropic Programmatic Tool Calling / Tool Search、`ProviderStrategy` の strict schema |
| `langgraph` v1.1 | 2026/3/10 | Type-safe streaming `version="v2"`、Type-safe invoke `version="v2"`、Pydantic/dataclass 自動コアース、サブグラフでの time travel 修正 |
| `deepagents` v0.5 | 2026/4/7 | Async subagents、PDF/audio/video の multi-modal `read_file` |
| `langchain` 1.2.16〜1.2.18 (patch) | 2026/4〜5月 | HITL ミドルウェアに `respond` decision、`ls_agent_type` タグ、tool-dispatch Sends の perf 改善、`langchain-classic` の Deprecation 整理(hub の deprecation、loads/dumps の制限) |
| **`langchain` 1.3.0a1/a2** | **2026/5/6** | **`stream_events(version="v3")` プロトコルを `langchain-core` に実装し、`create_agent` に配線** |
| `langchain-core` 1.3.3 | 2026/5 | `load()` を untrusted manifest に対して堅牢化、deprecation since の整合、CVE-2026-34070(path traversal) 修正のバックポート |

> 注: 「v1.3 とは何か」を一つの公式ブログにまとめた記事は本稿執筆時点で未公開。実体は **「v3 streaming を中心としたマイナーリリース」**であり、Deep Agents 0.6 ブログ(2026/5/13)が同じ仕組みを使うかたちでお披露目されている。

### 2.2 主要な変更点(マイグレーション観点)

#### 破壊的変更

v1.x 系は **「2.0 までは破壊的変更を入れない」** ことが LangChain 1.0 発表時に明文化されているため、v1.2 → v1.3 の純粋な breaking change は基本的に存在しない。実務上気をつけるべき「壊れる可能性」は以下の3点に集約される。

1. **`langchain.hub` の deprecation(`langchain-classic` 経由)**
   `langchain-classic` 1.0.7 で `hub` 機能が deprecated 化。`langchain` 0.3 系にもバックポートされている。プロンプト Hub を使うコードは将来削除に備えて移行が必要。
2. **`load()` / `loads()` / `dumps()` のセキュリティ厳格化**
   `langchain-core` 1.3.3 と関連 patch で「未検証マニフェストの逆シリアライズを制限」。社外/未検証 JSON から Runnable を読み込んでいる場合、ホワイトリスト指定が必須になる可能性がある(CVE-2026-34070 対応の一環)。
3. **`langchain-google-genai` 4.0.0 への移行**(v1.2 期間中の出来事だが未対応プロジェクトも多い)
   `langchain-google-vertexai` の一部 API が deprecated。Gemini と Vertex を統合 SDK に寄せる。

#### 非推奨化された API(v1.2 期間〜v1.3 で進行中のもの)

- `from langchain import hub` → `from langchain_classic import hub`(将来的に削除)
- `langchain.chains.*` 全般 → `langchain_classic.chains.*`(v1.0 以来)
- `langchain.retrievers.*`(`MultiQueryRetriever` 等)→ `langchain_classic.retrievers.*`
- `langgraph.prebuilt.create_react_agent` → `langchain.agents.create_agent`(v1.0 以来)
- `stream_events(version="v1")` は引き続き動くがメンテモード。v2 は安定、v3 は **beta**。

#### 新機能

- **`stream_events(version="v3")` プロトコル**(本リリースの目玉)
  - コンテンツブロック中心のストリーム。チャンク種別を「テキスト / 推論 / メディア / ツール呼び出し」と型で扱える。
  - チャンネル別の型付き射影:`run.values`(values モード相当)、`run.messages`(LLM 呼び出しごとに 1 つの `ChatModelStream`)、`run.lifecycle`、`run.subgraphs`。
  - **opt-in トランスフォーマ**: updates、custom events、checkpoints、tasks、debug を必要なものだけ subscribe。
  - **`create_agent` への配線**: ユーザは Agent から `stream_events(..., version="v3")` を呼ぶだけで上記が降りてくる。
  - **タイムアウト/エラーハンドラは Python 専用**、リトライポリシーは Py/TS 両方。
- **HITL middleware の `respond` decision**(`langchain` 1.2.17)
  従来の `approve` / `reject` / `edit` に加え、人間が**ツール呼び出しを介さずに直接最終応答を返す**ことが可能になった。承認フローで「これは私が答えるからツール実行はスキップ」というパターンが書きやすい。
- **`create_agent` の perf 改善**(`langchain` 1.2.17 / 1.3.0a1 へ向けて)
  `perf(langchain): stop inlining agent state into tool-dispatch Sends` — 大きな state を全ツール呼び出しに毎回インライン化する挙動を止め、長い state や大きなファイル state を持つエージェントでメモリ・トレース両面で軽くなる。
- **`state_schema` の優先順位修正**(`langchain` 1.3.0a2)
  `ordered schema resolution — list replaces set so state_schema wins`。複数ソースから schema が来る場合に、ユーザ指定の `state_schema` が確実に勝つように。

#### 改善・バグ修正

- `langchain-core` で `merge_lists` が並列ツール呼び出しを誤マージする問題を修正、空のツールチャンク ID を欠落として扱う、ツールスキーマが JSON シリアライズ不能な場合のエラー文言改善。
- `langchain-anthropic` 1.4.2、`langchain-openai` 1.2.1、`langchain-fireworks` 1.2.1 と各 provider が同期。Bedrock 用に `ChatAnthropicBedrock` ラッパ追加。
- セキュリティ: `urllib3 2.7.0` へ、path traversal の修正 (CVE-2026-34070)。

### 2.3 マイグレーションガイド

v1.2 で動いているコードはほぼそのまま v1.3 で動く。実務的な書き換えポイントを 5 つに絞って示す。

#### (a) ストリーミングを v2 → v3 に上げる

**Before(v1.2、`stream_events` v2):**

```python
from langchain.agents import create_agent

agent = create_agent(model="anthropic:claude-sonnet-4-5", tools=[...])

async for event in agent.astream_events(
    {"messages": [{"role": "user", "content": "..."}]},
    version="v2",
):
    if event["event"] == "on_chat_model_stream":
        chunk = event["data"]["chunk"]
        print(chunk.content, end="", flush=True)
```

**After(v1.3、`stream_events` v3):**

```python
from langchain.agents import create_agent

agent = create_agent(model="anthropic:claude-sonnet-4-5", tools=[...])

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "..."}]},
    version="v3",
)

# 型付き射影でテキストだけを購読
for message in stream.messages:
    for delta in message.text:
        print(delta, end="", flush=True)

# サブエージェント(deepagents 連携時)も型付きで購読可能
for subagent in stream.subagents:
    print(f"\n[{subagent.name}] {subagent.status}")
```

ポイントは **「`event["event"] == ...` の文字列マッチを書かなくて済む」**こと。UI 側で「テキストだけ」「推論ブロックだけ」「ツール呼び出しだけ」を購読する SSE / WebSocket エンドポイントの実装が大幅にシンプル化する。なお v2 は不変なので段階的移行で構わない。

#### (b) HITL Middleware の `respond` を使う

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(
    model="openai:gpt-5",
    tools=[refund_tool, send_email_tool],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "refund_tool": True,    # 承認必須
                "send_email_tool": True,
            },
            # v1.3 で追加: "respond" を選ぶと人間の発話で最終応答に置き換え
            decisions=["approve", "reject", "edit", "respond"],
        )
    ],
)
```

#### (c) `langchain.hub` の使用箇所を `langchain_classic` に逃がす

```python
# Before
from langchain import hub
prompt = hub.pull("rlm/rag-prompt")

# After (deprecated 経路を回避)
# pip install langchain-classic
from langchain_classic import hub
prompt = hub.pull("rlm/rag-prompt")
```

#### (d) `load()` を信頼できない入力に対して使っているケース

`langchain-core` 1.3.3 以降では `load()` が「allowed namespaces」のホワイトリストを期待するようになっており、社外から受け取った Runnable JSON を直接 `loads()` するコードはエラーになるか警告が出る。社内成果物のみに限定するか、明示的に `valid_namespaces=[...]` を渡す方針に切り替える。

#### (e) `agent.invoke` の戻り値の型を活用するなら LangGraph v1.1 の `version="v2"` も合わせる

```python
result = agent.invoke(
    {"messages": [{"role": "user", "content": "..."}]},
    version="v2",  # LangGraph 1.1+
)
# .value と .interrupts が型付きで取れる。Pydantic/dataclass に自動コアース。
final_state = result.value
pending = result.interrupts
```

---

## 3. deepagents v0.6 全体像

### 3.1 deepagents とは

**「バッテリー同梱型(batteries-included)のエージェントハーネス」**。Claude Code に着想を得て、「LLM がツールをループで呼ぶだけ」の浅い(shallow)エージェントを、長時間・複雑タスクに耐える「深い(deep)」エージェントに仕立てるための定型を、汎用ライブラリとして実装したもの。

Deep Agents は以下を **デフォルトで** 提供する:

- **Planning ツール** — `write_todos` / `read_todos` でタスクを TODO に分解し、進捗を追跡
- **仮想ファイルシステム** — `read_file` / `write_file` / `edit_file` / `ls` / `glob` / `grep`(state または LangGraph Store、もしくは外部バックエンドに格納)
- **シェル実行** — `execute`(サンドボックスバックエンド経由で安全に)
- **Sub-agents** — `task` ツールで隔離されたコンテキストを持つサブエージェントへ委譲
- **コンテキスト管理** — 会話の自動要約、大きなツール出力はファイルへオフロード
- **HITL** — CLI の `HITLMiddleware` でシェル等の機微なツール呼び出しの承認

LangGraph 上の状態機械にミドルウェアスタックを被せた構造で、すべての副作用(ファイル、シェル、サブエージェント)は **バックエンドプロトコル** 越しに抽象化されているため、State / Store / Composite / ContextHub / Modal / Daytona / Runloop 等に差し替え可能。

### 3.2 主要機能(v0.6 時点)

| カテゴリ | 機能 | 中身 |
|---|---|---|
| 計画 | `TodoListMiddleware` | `write_todos` でタスク分解、進捗追跡 |
| ファイルシステム | `read_file` / `write_file` / `edit_file` / `ls` / `glob` / `grep` | 仮想 FS。**v0.5 以降は PDF/audio/video/画像のマルチモーダル対応**(モデル profile で対応可否を確認) |
| 実行環境 | `execute` + sandbox backends | `langchain-modal` / `langchain-daytona` / `langchain-runloop`(v0.4 から)で外部サンドボックスに差し替え可能 |
| Sub-agents | **Inline** `SubAgent` / `CompiledSubAgent` / **`AsyncSubAgent`**(v0.5〜) | Async は LangSmith Deployment + Agent Protocol 準拠サーバを呼び、メイン会話をブロックせず並行実行 |
| ストリーミング | **`stream_events(version="v3")`**(v0.6) | サブエージェント別・チャンネル別の型付きストリーム |
| Code Interpreter | **`REPLMiddleware`**(v0.6、`deepagents[quickjs]`) | エージェントが QuickJS ランタイムで JS コードを書いて実行。ツールを「コードから」呼ぶ Programmatic Tool Calling と再帰ワークフローが可能に |
| ハーネスプロファイル | **Harness Profiles**(v0.6) | モデルごとの最適なシステムプロンプト・ツール記述・ミドルウェア構成を「名前付き・バージョン管理可能」にまとめる |
| 永続化 | **`DeltaChannel`**(v0.6) | チェックポイントを「フルスナップショット」ではなく「差分」で保存。10〜100倍のストレージ削減 |
| バックエンド | **`ContextHubBackend`**(v0.6) | LangSmith Context Hub をファイルシステムバックエンドに。スキル/メモリ/プロンプトを Git ライクにバージョン管理 |
| HITL | `HITLMiddleware` | CLI で承認/拒否、`respond` 等の決定(v1.3 LangChain と同期) |
| 自動要約 | `SummarizationMiddleware` | v0.4 から `wrap_model_call` で実行、`ContextOverflowError` でも自動発火 |

### 3.3 基本的な使い方

#### インストールと最小構成

```python
# pip install deepagents
# pip install langchain-anthropic  (またはお好みのプロバイダ)

from deepagents import create_deep_agent

agent = create_deep_agent()  # 最小: デフォルトモデル + すべてのバッテリー

result = agent.invoke(
    {"messages": [{"role": "user", "content":
        "LangGraph について調査して summary.md に要約を書いて"}]}
)
```

#### カスタムツール + モデル指定

```python
from deepagents import create_deep_agent
from langchain.chat_models import init_chat_model

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_deep_agent(
    model=init_chat_model("openai:gpt-5"),
    tools=[get_weather],
    system_prompt="You are a research assistant.",
)
```

#### Sub-agents(同期 + 非同期の混在)

```python
from deepagents import create_deep_agent, SubAgent, AsyncSubAgent

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[
        SubAgent(
            name="critic",
            description="Reviews drafts for factual accuracy.",
            prompt="You are a strict fact-checker.",
            tools=[search_tool],
        ),
        # v0.5 以降: 非同期(リモート Agent Protocol サーバへの fire-and-forget)
        AsyncSubAgent(
            name="researcher",
            description="Performs deep research on a topic.",
            url="https://my-agent-server.dev",
            graph_id="research_agent",
        ),
    ],
)
```

#### v0.6 の Code Interpreter ミドルウェア

```python
# pip install "deepagents[quickjs]"
from deepagents import create_deep_agent
from langchain_quickjs import REPLMiddleware

agent = create_deep_agent(
    model="baseten:zai-org/GLM-5",
    middleware=[REPLMiddleware()],
)
```

これにより、モデルは以下のような JavaScript コードを書き、QuickJS ランタイム内で `tools.task()` / `tools.fetchUrl()` 等を直接コールできる(モデルラウンドトリップを減らし、トークン消費を抑える Programmatic Tool Calling):

```javascript
const pages = await Promise.all(
  urls.map((url) => tools.fetchUrl({ url })),
);
const relevant = pages
  .filter((page) => page.includes("interpreter"))
  .slice(0, 3);
relevant.map((page) => page.slice(0, 500));
```

#### v0.6 の ContextHub バックエンド

```python
from deepagents import create_deep_agent
from deepagents.backends import (
    ContextHubBackend, CompositeBackend, StateBackend
)

# 全 FS を Context Hub に乗せる
agent = create_deep_agent(
    model="google_genai:gemini-3.1-pro-preview",
    backend=ContextHubBackend("my-agent"),
)

# あるいは /memories/ だけを Hub にして残りはスレッドスコープに
agent = create_deep_agent(
    model="google_genai:gemini-3.1-pro-preview",
    backend=CompositeBackend(
        default=StateBackend(),
        routes={"/memories/": ContextHubBackend("my-agent")},
    ),
)
```

`LANGSMITH_API_KEY` を設定するとリポジトリへの書き込みが Git ライクなコミットとして残り、staging→production のタグ付けも可能。

### 3.4 LangChain / LangGraph との関係

- **LangChain**: モデル抽象、ツール、メッセージ、`create_agent` などの「コア building blocks」を提供。Deep Agents は LangChain のチャットモデル(`init_chat_model`)・ツール・ミドルウェアをそのまま使う。
- **LangGraph**: 状態機械ランタイム。durable execution、ストリーミング、HITL、永続化、リプレイ、time travel を担う。Deep Agents の `create_deep_agent()` は **コンパイル済み LangGraph グラフ** を返すので、LangGraph の checkpointer、LangGraph Studio、`thread_id` ベースの再開、`stream_events` がすべてそのまま使える。
- **Deep Agents 自身**: LangChain/LangGraph の上に位置する「ハーネス層」。エージェントループのトポロジーは同じだが、「TODO 計画 + 仮想 FS + サブエージェント + 自動要約 + 良いシステムプロンプト」を**意見表明的に**プリセット。

公式の三層分類は次のとおり: **LangChain(フレームワーク) → LangGraph(ランタイム) → Deep Agents(ハーネス)**。Anthropic の Claude Agent SDK が比較対象として位置づけられている。

### 3.5 v0.6 の特徴(2026年5月13日リリース、`deepagents==0.6.1` が PyPI 最新)

公式ブログ「New in Deep Agents v0.6」が掲げる **5本柱**:

1. **Code Interpreter (`REPLMiddleware`)** — QuickJS をミドルウェアとして注入し、モデルが「コードを書いてツールを呼ぶ」Programmatic Tool Calling と再帰ワークフローを実現。中間結果をモデルコンテキストに戻さず in-memory state で保持できるため、トークン消費とラウンドトリップを大幅削減。Anthropic が API レベルで提供している PTC を、任意のモデル(オープンウェイト含む)で実現できる。
2. **Harness Profiles** — Kimi K2.6 / GLM 5.1 / DeepSeek V4 のようなオープンウェイトモデルでも、システムプロンプトとツール記述を「モデル別の方言」に合わせれば閉鎖モデル並みの性能が出る。LangChain 社のベンチでは harness 層の調整だけで `gpt-5.2-codex` が Terminal-Bench 2.0 で 52.8% → 66.5% に、tau2-bench で `gpt-5.3-codex` +20%、`opus-4.7` +10% を実測。プロファイルは diff/version/swap 可能な第一級概念に。
3. **Streaming v3** — LangChain v1.3 と同期した型付きストリーム。`stream.messages` / `stream.subagents` / `stream.subgraphs` を直接 iterate でき、Agent Streaming Protocol 準拠で SSE / WebSocket 越しの remote stream も同じプロトコル。`@langchain/react` `@langchain/vue` `@langchain/svelte` `@langchain/angular` の v1 フロント統合も同時公開。
4. **DeltaChannel** — チェックポイントを差分保存。長時間・大コンテキスト agent の保存容量を **5.27 GB → 129 MB(200ターン coding session 実測)** に削減。`langgraph >= 1.2` 必須、beta。`Annotated[list[str], DeltaChannel(reducer, snapshot_frequency=5)]` のように State に組み込む。
5. **`ContextHubBackend`** — LangSmith Context Hub をバックエンドに。プロンプト/スキル/メモリをバージョン管理し、「ある run の改善が次の run に引き継がれる」サイクルを実現。書き込みは Git コミット相当でレビュー・タグ付け可能。

これと同時に、Managed Deep Agents(LangSmith 上で運用されるホスト型 Deep Agents)も発表されている。

### 3.6 ユースケース

deepagents が刺さる代表的シナリオ:

- **ディープリサーチ系**: トピックを分解し、サブエージェントに調査を委譲、結果をファイルに書き出して合成 → 最終レポートを返す。`AsyncSubAgent` を使えば「ユーザに進捗を流しながら別エージェントが裏で重いリサーチ」が可能。
- **長時間コーディングエージェント**: ファイルシステム/シェル/sandbox を活用、TODO で進捗管理、長文脈は DeltaChannel で安価に永続化。`deepagents-cli` は Claude Code / Cursor 風のターミナルエージェントとして実装済み。
- **データ分析パイプライン**: `langchain-modal` / `langchain-daytona` / `langchain-runloop` のサンドボックスで Python/Notebook を回し、Code Interpreter で中間データを保持しながらモデルにはサマリだけ返す。
- **ドキュメント駆動の社内エージェント**: ContextHubBackend で会社のスキル/ポリシー/メモリを Git 的に管理し、staging→prod でプロモート。
- **企業 Due Diligence エージェント**: 公式 LangChain ブログ事例(2026/5/8、Parallel 社との協業)で「会社調査エージェント」が題材化されている。
- **マルチモーダル入力処理**: PDF/音声/動画を `read_file` で読ませて要約・抽出。対応モダリティはモデル profile で判定。

---

## 4. 参考リンク

### LangChain v1.x / v1.3
- 公式 Changelog(Python): https://docs.langchain.com/oss/python/releases/changelog
- LangChain v1 リリースノート: https://docs.langchain.com/oss/python/releases/langchain-v1
- LangChain v1 マイグレーションガイド: https://docs.langchain.com/oss/python/migrate/langchain-v1
- GitHub Releases(`langchain-ai/langchain`): https://github.com/langchain-ai/langchain/releases
- LangChain 1.0 / LangGraph 1.0 GA ブログ: https://blog.langchain.com/langchain-langgraph-1dot0/
- LangChain 1.0 Changelog アナウンス: https://changelog.langchain.com/announcements/langchain-1-0-now-generally-available
- LangGraph Pregel(`DeltaChannel` 解説): https://docs.langchain.com/oss/python/langgraph/pregel
- `langchain` PyPI: https://pypi.org/project/langchain/

### deepagents v0.6
- 公式ブログ「New in Deep Agents v0.6」: https://www.langchain.com/blog/deep-agents-0-6
- 「Delta Channels: Evolving our Runtime for Long-Running Agents」: https://www.langchain.com/blog/delta-channels-evolving-agent-runtime
- 公式オーバービュー: https://docs.langchain.com/oss/python/deepagents/overview
- API Reference: https://reference.langchain.com/python/deepagents
- Deep Agents 製品紹介: https://www.langchain.com/deep-agents
- GitHub: https://github.com/langchain-ai/deepagents
- PyPI: https://pypi.org/project/deepagents/
- Deep Agents v0.5 ブログ(async subagents の前段): https://blog.langchain.com/deep-agents-v0-5/
- Tuning Deep Agents across different models: https://www.langchain.com/blog/tuning-deep-agents-different-models
- Streaming Cookbook: https://github.com/langchain-ai/streaming-cookbook
- DeepWiki アーキ解説(参考): https://deepwiki.com/langchain-ai/deepagents

---

### 補足: 一次情報と速報情報の食い違いについて

- 公式ドキュメントの **Changelog ページは 2026/4/7 までしか追従しておらず**、Apr 7 の deepagents v0.5、Mar 10 の langgraph v1.1 までしか掲載がない(本稿執筆時点)。
- 一方、GitHub `langchain-ai/langchain` の Releases には **2026/5/6 付で `langchain==1.3.0a1` / `1.3.0a2`** が、PyPI には `langchain-1.3.0` がそれぞれ存在し、Deep Agents 0.6 ブログ(2026/5/13)が新ストリーミングを大々的に披露している。
- したがって **「v1.3 はストリーミング v3 を主目玉とするマイナー」** という整理が現時点でもっとも妥当だが、独立した v1.3 アナウンスブログは出ていないため、最終確認は GitHub の `release(langchain): 1.3.0` 系コミットと `docs.langchain.com` の今後の Changelog 更新で行うことを推奨する。
- 同様に deepagents は **PyPI に 0.6.1 が出ている** 一方、GitHub Releases ページのキャッシュは `0.4.2` を Latest と表示しているケースがある(Releases タブの自動更新遅延)。`pip index versions deepagents` で実際の最新を確認するのが確実。