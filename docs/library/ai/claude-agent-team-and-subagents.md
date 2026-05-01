# Claude エージェントチームとサブエージェント 完全ガイド

> 調査日: 2026-04-29  
> 対象: Claude Code v2.1.x 以降 / Claude Agent SDK

---

## 目次

1. [概要](#1-概要)
2. [マルチエージェントシステムの3つのレイヤー](#2-マルチエージェントシステムの3つのレイヤー)
3. [サブエージェント詳解](#3-サブエージェント詳解)
4. [組み込みエージェントタイプ](#4-組み込みエージェントタイプ)
5. [カスタムサブエージェントの作成](#5-カスタムサブエージェントの作成)
6. [Agent SDK によるプログラム的なエージェント構築](#6-agent-sdk-によるプログラム的なエージェント構築)
7. [エージェントチーム（実験的機能）](#7-エージェントチーム実験的機能)
8. [エージェント間コミュニケーション](#8-エージェント間コミュニケーション)
9. [実行モード：フォアグラウンド vs バックグラウンド](#9-実行モードフォアグラウンド-vs-バックグラウンド)
10. [ワークツリー分離](#10-ワークツリー分離)
11. [パーミッションモード](#11-パーミッションモード)
12. [ツール使用とMCPサーバー](#12-ツール使用とmcpサーバー)
13. [ユースケースとパターン](#13-ユースケースとパターン)
14. [制限事項と注意点](#14-制限事項と注意点)
15. [比較表：サブエージェント vs エージェントチーム vs Managed Agents](#15-比較表サブエージェント-vs-エージェントチーム-vs-managed-agents)
16. [ベストプラクティス](#16-ベストプラクティス)

---

## 1. 概要

Claude のマルチエージェントシステムは、**オーケストレーター（指揮者）** が複数の **サブエージェント（作業者）** に役割を分担させ、複雑なタスクを効率的に処理する仕組みです。

### なぜマルチエージェントが必要か

- **コンテキストウィンドウの限界**: 大規模なコードベースや長い処理は1つのコンテキストに収まらない
- **並列処理**: 独立したタスクを同時に実行してスループットを向上させる
- **専門化**: 各エージェントが特定の役割（セキュリティレビュー、テスト作成など）に特化できる
- **コンテキスト汚染の防止**: 詳細なログや大量の出力をサブエージェントのコンテキストに留め、メインの会話をクリーンに保つ

> Anthropic の社内研究によれば、マルチエージェントシステムは単一の Opus 4 モデルと比較して研究タスクで **90.2% の性能向上** を達成しています。

---

## 2. マルチエージェントシステムの3つのレイヤー

| レイヤー | 種別 | 特徴 |
|---------|------|------|
| **レイヤー 1** | サブエージェント（同一セッション内） | Agent ツールで起動 / 親へ最終メッセージのみ返す / 1段階のネストのみ（サブエージェントはさらに生成不可） |
| **レイヤー 2** | エージェントチーム（実験的、複数セッション） | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` で有効化 / ピアツーピアのメールボックス通信 / 共有タスクリストで協調動作 |
| **レイヤー 3** | Managed Agents（APIレベル、リサーチプレビュー） | Anthropic がサンドボックスとセッションを管理 / REST API でオーケストレーター・ワーカーを定義 / イベントストリーミングで通信 |

---

## 3. サブエージェント詳解

### サブエージェントとは

サブエージェントは、メインエージェントが `Agent` ツールを使って起動する **専門化された別のエージェントインスタンス** です。

各サブエージェントは以下を持つ独立した環境で動作します：
- 独自のシステムプロンプト
- 制限されたツールセット
- 独立したパーミッション設定
- 新規のコンテキストウィンドウ（会話履歴なし）

### コミュニケーションの仕組み

```
メインエージェント
    │
    │  Agent ツール呼び出し
    │  （プロンプト文字列のみ渡せる）
    ▼
サブエージェント
    │
    │  最終テキストレスポンスのみ返す
    │  （ツール呼び出し結果や中間出力は返らない）
    ▼
メインエージェント（結果受信）
```

**重要**: 親からサブエージェントへの唯一のチャネルは **Agent ツールのプロンプト文字列** です。必要なファイルパス、エラーメッセージ、決定事項はすべてこのプロンプトに含める必要があります。

### サブエージェントの作成方法（3種類）

| 方法 | 説明 |
|------|------|
| **組み込みエージェント** | `general-purpose`, `Explore`, `Plan` など、常に利用可能 |
| **ファイルシステムベース** | `.claude/agents/` または `~/.claude/agents/` に Markdown ファイルを配置 |
| **プログラム的（SDK）** | `query()` 関数の `agents` パラメータで定義 |

---

## 4. 組み込みエージェントタイプ

| エージェント | モデル | ツール | 用途 |
|------------|--------|--------|------|
| `Explore` | Haiku（高速） | 読み取り専用（Write/Edit なし） | ファイル検索、コードベース探索 |
| `Plan` | 継承 | 読み取り専用（Write/Edit なし） | プランモードでのコードベース調査 |
| `general-purpose` | 継承 | 全ツール | 複雑なマルチステップタスク |
| `claude-code-guide` | Haiku | 限定的 | Claude Code の機能・ドキュメントへの質問 |
| `statusline-setup` | Sonnet | 限定的 | ステータスラインの設定（`/statusline` コマンド） |

### Explore サブエージェントの詳細

`Explore` は最も頻繁に使われる組み込みエージェントです。

- **コスト**: Haiku モデルのため低コスト
- **速度**: 最速
- **制約**: Write/Edit ツールへのアクセスなし（読み取り専用）
- **粒度指定**: `quick`（ピンポイント検索）、`medium`（バランス型）、`very thorough`（網羅的）

**無効化する場合**:
```json
{
  "permissions": {
    "deny": ["Agent(Explore)"]
  }
}
```

### Plan サブエージェントの詳細

`Plan` は **プランモード**（`/plan` コマンドまたは `permissionMode: plan`）で使用されます。

- **目的**: 無限ネストを防ぐためのアーキテクチャ上の工夫
- **動作**: プランモード中にコードベースを理解するためのリサーチを読み取り専用で実行
- `Explore` との違い: `Explore` は常に Haiku を使うが、`Plan` はメイン会話のモデルを継承する

---

## 5. カスタムサブエージェントの作成

### ファイル配置

```
プロジェクトレベル: .claude/agents/<name>.md
ユーザーグローバル: ~/.claude/agents/<name>.md
```

### Markdown ファイルの構造

```yaml
---
name: code-reviewer
description: >
  コードの品質、セキュリティ、保守性をレビューする専門エージェント。
  コードレビューが必要なときに使用。
tools: Read, Grep, Glob
model: sonnet
memory: project
permissionMode: acceptEdits
isolation: worktree
background: true
effort: high
color: blue
maxTurns: 50
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
---

あなたはシニアのコードレビュアーです。セキュリティ、パフォーマンス、
ベストプラクティスの観点から詳細なレビューを行います。
```

### フロントマターフィールド一覧

| フィールド | 必須 | 説明 |
|-----------|------|------|
| `name` | Yes | エージェントの識別子 |
| `description` | Yes | いつこのエージェントを使うかの自然言語説明 |
| `tools` | No | 利用可能なツールのホワイトリスト |
| `disallowedTools` | No | 除外するツールのブラックリスト |
| `model` | No | `sonnet`, `opus`, `haiku`, `inherit`, またはフルモデルID |
| `memory` | No | `user`, `project`, `local` で永続メモリを有効化 |
| `permissionMode` | No | パーミッションモード |
| `isolation` | No | `worktree` でgit worktree分離を有効化 |
| `background` | No | `true` でバックグラウンド実行 |
| `effort` | No | `low`, `medium`, `high`, `xhigh`, `max` |
| `maxTurns` | No | 最大ターン数 |
| `color` | No | UI表示色 |
| `mcpServers` | No | このエージェント専用のMCPサーバー |
| `hooks` | No | ライフサイクルフック |
| `skills` | No | 読み込むスキル名のリスト |
| `initialPrompt` | No | エージェント起動時に自動投入される最初のプロンプト |

### 優先順位（高 → 低）

1. Managed Settings（組織全体）
2. `--agents` CLIフラグ（現在のセッションのみ）
3. `.claude/agents/`（プロジェクトレベル）
4. `~/.claude/agents/`（ユーザーグローバル）
5. プラグインの `agents/` ディレクトリ

---

## 6. Agent SDK によるプログラム的なエージェント構築

### インストール

```bash
# TypeScript（Claude Code バイナリを内包）
npm install @anthropic-ai/claude-agent-sdk

# Python
pip install claude-agent-sdk
```

**認証**: `ANTHROPIC_API_KEY` 環境変数を設定。Amazon Bedrock、Google Vertex AI、Microsoft Azure にも対応。

### 基本的な使い方

```python
# Python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async for message in query(
    prompt="auth.py のバグを見つけて修正してください",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Bash"]
    ),
):
    if hasattr(message, "result"):
        print(message.result)
```

```typescript
// TypeScript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "auth.ts のバグを見つけて修正してください",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

### カスタムサブエージェントを持つオーケストレーター

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async for message in query(
    prompt="認証モジュールのセキュリティ問題をレビューしてください",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Grep", "Glob", "Agent"],
        agents={
            "code-reviewer": AgentDefinition(
                description="コード品質・セキュリティ・保守性のレビュー専門家",
                prompt="あなたはセキュリティ、パフォーマンス、ベストプラクティスの専門家です。",
                tools=["Read", "Grep", "Glob"],
                model="sonnet",
            ),
            "test-writer": AgentDefinition(
                description="テストコード作成専門家",
                prompt="あなたはテスト駆動開発の専門家です。",
                tools=["Read", "Write", "Bash"],
                model="haiku",
            ),
        },
    ),
):
    if hasattr(message, "result"):
        print(message.result)
```

### 動的エージェント生成（ファクトリーパターン）

```python
def create_security_agent(security_level: str) -> AgentDefinition:
    is_strict = security_level == "strict"
    return AgentDefinition(
        description="セキュリティコードレビュアー",
        prompt=f"あなたは{'厳格な' if is_strict else 'バランスの取れた'}セキュリティレビュアーです。",
        tools=["Read", "Grep", "Glob"],
        model="opus" if is_strict else "sonnet",  # 高リスクタスクには高性能モデル
    )
```

### セッション再開

```python
# 1回目の実行でセッションIDを取得
session_id = None
async for message in query(prompt="最初のタスク", options=...):
    if hasattr(message, "type") and message.type == "system:init":
        session_id = message.session_id

# 2回目の実行で再開
async for message in query(
    prompt="続きのタスク",
    options=ClaudeAgentOptions(resume=session_id, ...)
):
    ...
```

### `ClaudeAgentOptions` 主要フィールド

| フィールド | 説明 |
|-----------|------|
| `allowedTools` | 使用可能なツールのリスト |
| `agents` | カスタムサブエージェントの辞書 |
| `mcpServers` | インラインMCPサーバー設定 |
| `permissionMode` | パーミッションモード |
| `resume` | 再開するセッションID |
| `systemPrompt` | システムプロンプトの上書き |
| `hooks` | ライフサイクルコールバック |
| `settingSources` | 読み込む設定ソース |

---

## 7. エージェントチーム（実験的機能）

### 有効化

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### サブエージェントとの違い

| 項目 | サブエージェント | エージェントチーム |
|------|----------------|------------------|
| セッション数 | 1つのセッション内 | 複数の独立したセッション |
| 通信 | 一方向（最終結果のみ） | ピアツーピア（メールボックス） |
| タスク管理 | なし | 共有タスクリスト |
| 状態 | 一般公開（GA） | 実験的（オプトイン必須） |

### エージェントチームの構成

```
チームリード（オーケストレーター）
    ├── チームメイト A（独立したセッション）
    ├── チームメイト B（独立したセッション）
    └── チームメイト C（独立したセッション）
```

- **チームリード**: オーケストレーター、タスクを配分・監督
- **チームメイト**: ワーカー、独立したコンテキストを持つ
- **共有タスクリスト**: `~/.claude/tasks/{team-name}/` に保存
- **チーム設定**: `~/.claude/teams/{team-name}/config.json`

### 推奨チームサイズ

- **3〜5人のチームメイト** が最適
- **1人あたり5〜6タスク** でコンテキスト切り替えなしに高生産性を維持

---

## 8. エージェント間コミュニケーション

### サブエージェントの通信（一方向）

```
メインエージェント ──プロンプト──▶ サブエージェント
メインエージェント ◀──最終結果── サブエージェント
```

- 中間のツール呼び出し結果は渡らない
- 最終メッセージのみが親に返る

### エージェントチームの通信（双方向）

`SendMessage` ツールによるメッセージタイプ：

| タイプ | 説明 |
|--------|------|
| `message` | 特定のチームメイトへの直接メッセージ |
| `broadcast` | 全チームメイトへの一斉送信（コストに注意） |
| `shutdown_request` / `shutdown_response` | グレースフルシャットダウン |
| `plan_approval_response` | プラン承認ゲートメカニズム |

**自動配信**: メッセージは自動配信され、リードがポーリングする必要なし。  
**アイドル通知**: チームメイトが完了すると自動的にリードに通知。  
**自動再開**: 停止中のサブエージェントが `SendMessage` を受け取るとバックグラウンドで自動再開。

---

## 9. 実行モード：フォアグラウンド vs バックグラウンド

### フォアグラウンド（デフォルト）

- メインの会話をブロックして完了を待つ
- パーミッション確認や `AskUserQuestion` がユーザーに通知される
- 結果が次の処理に必要な場合に使用

### バックグラウンド

- メインの会話と並行して実行（ノンブロッキング）
- 起動前にすべてのパーミッションを事前承認
- 実行中のパーミッション確認は自動拒否
- `AskUserQuestion` によるユーザー質問は失敗（エージェントは継続）

**有効化方法**:
```yaml
# フロントマターで指定
background: true
```
または、Claudeに「バックグラウンドで実行して」と指示、または `Ctrl+B` で切り替え。

**全バックグラウンドタスクの無効化**:
```bash
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1
```

### フォークモード（実験的）

`CLAUDE_CODE_FORK_SUBAGENT=1`（v2.1.117以降）を設定すると：
- すべてのサブエージェント起動がバックグラウンドで実行
- フォークは **親の会話履歴を継承**（通常のサブエージェントは新規コンテキスト）
- 親のプロンプトキャッシュを再利用（コスト削減）

---

## 10. ワークツリー分離

### 概要

`isolation: worktree` を設定すると、サブエージェントは独立した git worktree のコピーで作業します。

```yaml
---
name: refactor-worker
description: コードを分離環境でリファクタリング
isolation: worktree
background: true
---
```

### 動作

- メインのチェックアウトとは別の worktree でファイルを変更
- 他の並列サブエージェントとの競合を防止
- 変更がなければ worktree は自動クリーンアップ
- 変更があればブランチ名と worktree パスが結果に含まれる

### 推奨パターン

```
background: true + isolation: worktree
```

この組み合わせが、同一コードベースで複数エージェントを並列実行する際の **推奨パターン** です。

---

## 11. パーミッションモード

| モード | 動作 |
|--------|------|
| `default` | 標準パーミッション確認（インタラクティブプロンプト） |
| `acceptEdits` | ファイル編集・一般的なファイルシステム操作を自動承認。シェルコマンドは確認あり |
| `auto` | バックグラウンド分類器がコマンドを評価。ほとんどの操作でプロンプトなし |
| `dontAsk` | パーミッション確認を自動拒否（明示的に許可されたツールは動作する） |
| `bypassPermissions` | すべてのプロンプトをスキップ。一部の保護ディレクトリは例外 |
| `plan` | 読み取り専用モード（ファイル変更不可） |

### 継承ルール

- 親が `bypassPermissions` または `acceptEdits` を使用している場合、サブエージェントの設定より優先
- 親が `auto` モードの場合、サブエージェントの `permissionMode` フロントマターは無視される

---

## 12. ツール使用とMCPサーバー

### 利用可能な組み込みツール

| ツール | 用途 |
|--------|------|
| `Read` | ファイル読み取り |
| `Write` | 新規ファイル作成 |
| `Edit` | 既存ファイルの精密な編集 |
| `Bash` | シェルコマンド実行 |
| `Monitor` | バックグラウンドプロセスの監視・イベントストリーム |
| `Glob` | パターンによるファイル検索 |
| `Grep` | 正規表現によるファイル内容検索 |
| `WebSearch` | ウェブ検索 |
| `WebFetch` | ウェブページの取得・解析 |
| `AskUserQuestion` | ユーザーへの確認（選択肢付き） |
| `Agent` | サブエージェントの起動 |

### ツール制限の設定

```yaml
# ホワイトリスト（指定ツールのみ使用可）
tools: Read, Grep, Glob

# ブラックリスト（指定ツールを除外）
disallowedTools: Bash, Write

# スポーンできるサブエージェントを制限
tools: Agent(worker, researcher), Read, Bash
```

**両方指定した場合**: `disallowedTools` を先に適用し、残ったツールに対して `tools` を解決。

### MCP サーバーのスコープ

```yaml
mcpServers:
  - playwright:           # このサブエージェント専用（インライン定義）
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github                # 既存のサーバーへの参照
```

サポートトランスポート: `stdio`, `http`, `sse`, `ws`

---

## 13. ユースケースとパターン

### パターン 1: 並列コードレビュー

```
PRレビューをするエージェントチームを作成。3つのレビュアーを起動:
- セキュリティの観点でチェックするエージェント
- パフォーマンスの影響をチェックするエージェント
- テストカバレッジを検証するエージェント
```

3つのサブエージェントが同時実行、それぞれ異なる視点でレビュー。リードが知見を統合。

### パターン 2: コンテキスト汚染の防止

```
テストスイートを実行して、失敗したテストとエラーメッセージのみ報告してください
```

詳細なテストログ（数千行）をサブエージェントのコンテキストに留め、関連するサマリーのみ返す。

### パターン 3: 競合仮説によるデバッグ

```
5つのエージェントチームメイトを使って異なる仮説を並行調査。
お互いの理論を反証し合って科学的な議論をしてください。
```

逐次調査では確証バイアスの問題がある。複数の調査者が互いの理論を積極的に反証することで、より正確に根本原因に収束する。

### パターン 4: チェーン型サブエージェント

```
code-reviewer サブエージェントでパフォーマンス問題を特定し、
その結果を optimizer サブエージェントに渡して修正してください
```

各サブエージェントが完了したら結果を返し、Claudeが次のエージェントに関連コンテキストを渡す逐次チェーン。

### パターン 5: 大規模コードベースの分割調査

```python
# オーケストレーターがモジュールごとにサブエージェントを起動
orchestrator = client.beta.agents.create(
    name="セキュリティ監査リード",
    tools=[{"type": "agent_toolset_20260401"}],
    callable_agents=[
        {"type": "agent", "id": reviewer_agent.id},
        {"type": "agent", "id": test_writer_agent.id},
    ],
)
```

### パターン 6: コストコントロール（モデルルーティング）

- `Explore`: Haiku（読み取り専用ファイル検索）→ 低コスト
- セキュリティレビュー: Opus（高精度が必要）→ 高コスト
- 通常のサブエージェント: `inherit`（メイン会話と同じモデル）→ バランス

### パターン 7: 永続メモリを持つサブエージェント

```yaml
memory: project  # .claude/agent-memory/<name>/ に保存
```

コードベースのパターン、デバッグの知見、アーキテクチャの決定が会話をまたいで蓄積される。`MEMORY.md`（最初の200行または25KB）が起動時にインジェクトされる。

---

## 14. 制限事項と注意点

### サブエージェントの制限

| 制限 | 詳細 |
|------|------|
| **ネスト不可** | サブエージェントは自分でサブエージェントを生成できない（`Agent` ツールを `tools` に含めないこと） |
| **コンテキストリセット** | 各起動時に新鮮なコンテキスト（前回の呼び出しの状態は引き継がれない） |
| **Windowsの制限** | プロンプトが長すぎるとコマンドライン文字数制限（8,191文字）で失敗。複雑な指示はファイルシステムベースのエージェントを使用 |
| **セッション再読み込み** | `.claude/agents/` のファイルはセッション開始時のみ読み込まれる。実行中の追加には `/agents` コマンドまたは再起動が必要 |
| **コスト** | マルチエージェントシステムは単一エージェントと比較して **10〜15倍のトークン** を使用 |

### エージェントチームの制限（実験的機能）

| 制限 | 詳細 |
|------|------|
| **セッション再開不可** | `/resume` と `/rewind` はインプロセスのチームメイトを復元しない |
| **タスクステータスの遅延** | チームメイトが完了マークを付け忘れることがある |
| **シャットダウン遅延** | チームメイトは現在のツール呼び出し完了後にシャットダウン |
| **1セッション1チーム** | 新しいチームを起動する前に既存チームをクリーンアップ必要 |
| **ネスト不可** | チームメイトは自分のチームを作れない |
| **リード固定** | チームのライフタイム中はリードが変更不可 |
| **ターミナル要件** | 分割ペインには tmux または iTerm2 が必要（VS Code統合ターミナル、Windows Terminal、Ghostty は非対応） |
| **フロントマター制限** | チームメイトとして実行時、サブエージェント定義の `skills` と `mcpServers` フロントマターは適用されない（プロジェクト・ユーザー設定から読み込まれる） |

---

## 15. 比較表：サブエージェント vs エージェントチーム vs Managed Agents

| 項目 | サブエージェント | エージェントチーム | Managed Agents (API) |
|------|----------------|------------------|---------------------|
| **通信** | 一方向（最終メッセージのみ） | 双方向ピアツーピア + 共有タスクリスト | セッションスレッド + イベントストリーミング |
| **ネスト** | 1段階のみ | ネスト不可 | 1段階のみ |
| **コンテキスト分離** | 呼び出しごとに新規 | 各チームメイトが独立 | 各スレッドが独立 |
| **スコープ** | 単一セッション内 | 複数の独立したセッション | APIレベル（Anthropicがホスト） |
| **並列性** | 複数バックグラウンドサブエージェント | 複数チームメイトの同時実行 | セッションあたり複数スレッド |
| **インフラ** | 自前（Claude Code内） | 自前（複数Claude Codeセッション） | Anthropicがサンドボックス管理 |
| **状態** | 一般公開（GA） | 実験的（オプトイン） | リサーチプレビュー（要申請） |

---

## 16. ベストプラクティス

### インライン処理が適切なケース

- 頻繁な往復や反復的な改良が必要なタスク
- 複数フェーズが大きなコンテキストを共有（計画 → 実装 → テスト）
- クイックかつターゲットを絞った変更
- レイテンシーが重要な場合（サブエージェントは新規起動のため時間がかかる）

### サブエージェントが適切なケース

- 詳細な出力（テスト実行、ログ分析、ドキュメント取得）をメインコンテキストから隔離したい
- 特定のツール制限やパーミッションを強制したい
- 自己完結したタスクでサマリーを返せる
- 並列化可能な独立したタスク

### エージェントチームが適切なケース

- チームメイトが知見を共有し、互いに挑戦し、自律的に調整する必要がある
- 複数の独立したドメイン（フロントエンド、バックエンド、テスト）にまたがる作業
- 競合する仮説で並列調査を行うデバッグ
- 複数の視点が品質を高める調査タスク

### 避けるべきアンチパターン（Anthropicの経験より）

1. **過度なサブエージェント生成**: 単純な検索に50個のサブエージェントを生成するなど
2. **存在しないソースの際限ない検索**: ウェブを探し続けても見つからないソースへの執着
3. **過度な進捗報告**: 実際の作業を邪魔する過剰なエージェント間の更新
4. **同一ファイルへの並列編集**: worktree 分離なしに同じファイルを複数エージェントが変更

### 並列化の原則

| ケース | 推奨 |
|--------|------|
| 独立したタスク | 並列実行 |
| 依存関係のあるタスク | 逐次実行 |
| 同一ファイルの編集 | worktree 分離なしに並列化しない |
| 読み取り専用タスク | 常に並列化安全 |

### Anthropicからの推奨

> 「シンプルなプロンプトから始め、包括的な評価で最適化し、シンプルな解決策が不十分な場合のみマルチステップのエージェントシステムを追加すること。複雑さは、それが明らかに結果を改善する場合のみ追加を検討してください。」
> — Anthropic「Building Effective Agents」

---

## 参考リソース

- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Subagents in the SDK - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Agent SDK overview - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/overview)
- [Multiagent sessions - Claude API Docs](https://platform.claude.com/docs/en/managed-agents/multi-agent)
- [Building Effective AI Agents - Anthropic Research](https://www.anthropic.com/research/building-effective-agents)
- [How we built our multi-agent research system - Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Building agents with the Claude Agent SDK - Anthropic Engineering](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
