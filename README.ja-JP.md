# graphify

[English](README.md) | [简体中文](README.zh-CN.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

[![CI](https://github.com/safishamsi/graphify/actions/workflows/ci.yml/badge.svg?branch=v4)](https://github.com/safishamsi/graphify/actions/workflows/ci.yml)
[![PyPI](https://img.shields.io/pypi/v/graphifyy)](https://pypi.org/project/graphifyy/)
[![Downloads](https://static.pepy.tech/badge/graphifyy/month)](https://pepy.tech/project/graphifyy)
[![Sponsor](https://img.shields.io/badge/sponsor-safishamsi-ea4aaa?logo=github-sponsors)](https://github.com/sponsors/safishamsi)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Safi%20Shamsi-0077B5?logo=linkedin)](https://www.linkedin.com/in/safi-shamsi)

**AIコーディングアシスタント向けのスキル。** Claude Code、Codex、OpenCode、Cursor、Gemini CLI、GitHub Copilot CLI、Aider、OpenClaw、Factory Droid、Trae、Hermes、Google Antigravity で `/graphify` と入力するだけで、ファイルを読み込んでナレッジグラフを構築し、あなたが気づいていなかった構造を返します。コードベースをより速く理解し、アーキテクチャ上の意思決定の「なぜ」を見つけ出します。

完全にマルチモーダル対応。コード、PDF、Markdown、スクリーンショット、図、ホワイトボード写真、他言語の画像、または動画・音声ファイルまで――graphify はそれらすべてから概念と関係性を抽出し、1 つのグラフに接続します。動画はコーパスから導出したドメイン認識プロンプトを使って Whisper でローカルに文字起こしされます。23 言語をサポート（22 言語は tree-sitter AST、Dart は regex 方式）: Python、JS、TS、Go、Rust、Java、C、C++、Ruby、C#、Kotlin、Scala、PHP、Swift、Lua、Zig、PowerShell、Elixir、Objective-C、Julia、Vue、Svelte、Dart。

> Andrej Karpathy は論文、ツイート、スクリーンショット、メモを放り込む `/raw` フォルダを持っています。graphify はまさにその問題への答えです――生ファイルを読むのに比べて1クエリあたりのトークン数が 71.5 倍少なく、セッションをまたいで永続化され、見つけたものと推測したものを正直に区別します。

```
/graphify .                        # どのフォルダでも動作 - コードベース、メモ、論文、なんでも
```

```
graphify-out/
├── graph.html       インタラクティブなグラフ - ノードをクリック、検索、コミュニティでフィルタ
├── GRAPH_REPORT.md  ゴッドノード、意外なつながり、推奨される質問
├── graph.json       永続化されたグラフ - 数週間後でも再読み込みなしでクエリ可能
└── cache/           SHA256 キャッシュ - 再実行時は変更されたファイルのみ処理
```

グラフに含めたくないフォルダを除外するには `.graphifyignore` ファイルを追加します：

```
# .graphifyignore
vendor/
node_modules/
dist/
*.generated.py
```

構文は `.gitignore` と同じです。パターンは graphify を実行したフォルダからの相対パスに対してマッチします。

## 仕組み

graphify は 3 パスで動作します。まず、決定論的な AST パスがコードファイルから構造（クラス、関数、インポート、コールグラフ、docstring、根拠コメント）を LLM なしで抽出します。次に、ビデオおよびオーディオファイルがコーパスのゴッドノードから導出したドメイン対応プロンプトを使って faster-whisper でローカル転写されます――転写結果はキャッシュされるため再実行時は即時処理されます。最後に、Claude サブエージェントがドキュメント、論文、画像、転写テキストに対して並列に実行され、概念、関係性、設計の根拠を抽出します。結果は NetworkX グラフにマージされ、Leiden コミュニティ検出でクラスタリングされ、インタラクティブ HTML、クエリ可能な JSON、平易な言葉の監査レポートとしてエクスポートされます。

**クラスタリングはグラフトポロジベース――埋め込みは使いません。** Leiden はエッジ密度によってコミュニティを見つけます。Claude が抽出する意味的類似性エッジ（`semantically_similar_to`、INFERRED とマーク）は既にグラフに含まれているため、コミュニティ検出に直接影響します。グラフ構造そのものが類似性シグナルであり――別途の埋め込みステップやベクターデータベースは不要です。

すべての関係は `EXTRACTED`（ソースから直接見つかった）、`INFERRED`（合理的な推論、信頼度スコア付き）、`AMBIGUOUS`（レビュー対象としてフラグ付け）のいずれかでタグ付けされます。何が見つかったもので何が推測されたものか、常に分かります。

## インストール

**必要なもの:** Python 3.10+ および以下のいずれか： [Claude Code](https://claude.ai/code)、[Codex](https://openai.com/codex)、[OpenCode](https://opencode.ai)、[Cursor](https://cursor.com)、[Gemini CLI](https://github.com/google-gemini/gemini-cli)、[GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli)、[Aider](https://aider.chat)、[OpenClaw](https://openclaw.ai)、[Factory Droid](https://factory.ai)、[Trae](https://trae.ai)、Hermes、または [Google Antigravity](https://antigravity.google)

```bash
pip install graphifyy && graphify install
```

> **公式パッケージ：** PyPI パッケージ名は `graphifyy` です（`pip install graphifyy` でインストール）。PyPI の `graphify*` という名前の他のパッケージはこのプロジェクトと無関係です。公式リポジトリは [safishamsi/graphify](https://github.com/safishamsi/graphify) のみです。CLI とスキルコマンドは引き続き `graphify` です。

### プラットフォームサポート

| プラットフォーム | インストールコマンド |
|----------|----------------|
| Claude Code (Linux/Mac) | `graphify install` |
| Claude Code (Windows) | `graphify install`（自動検出）または `graphify install --platform windows` |
| Codex | `graphify install --platform codex` |
| OpenCode | `graphify install --platform opencode` |
| GitHub Copilot CLI | `graphify install --platform copilot` |
| Aider | `graphify install --platform aider` |
| OpenClaw | `graphify install --platform claw` |
| Factory Droid | `graphify install --platform droid` |
| Trae | `graphify install --platform trae` |
| Trae CN | `graphify install --platform trae-cn` |
| Gemini CLI | `graphify install --platform gemini` |
| Hermes | `graphify install --platform hermes` |
| Cursor | `graphify cursor install` |
| Google Antigravity | `graphify antigravity install` |

Codex ユーザーは並列抽出のために `~/.codex/config.toml` の `[features]` の下に `multi_agent = true` も必要です。Factory Droid は並列サブエージェントディスパッチに `Task` ツールを使用します。OpenClaw、Aider、Hermes は逐次抽出を使用します（並列エージェントサポートはこれらのプラットフォームではまだ初期段階です）。Trae は並列サブエージェントディスパッチに Agent ツールを使用し、PreToolUse フックを**サポートしていません**――AGENTS.md が常時有効のメカニズムです。

次に、AI コーディングアシスタントを開いて入力します：

```
/graphify .
```

注意：Codex はスキル呼び出しに `/` ではなく `$` を使用するため、代わりに `$graphify .` と入力してください。

### アシスタントに常にグラフを使わせる（推奨）

グラフを構築した後、プロジェクトで一度だけ以下を実行します：

| プラットフォーム | コマンド |
|----------|---------|
| Claude Code | `graphify claude install` |
| Codex | `graphify codex install` |
| OpenCode | `graphify opencode install` |
| GitHub Copilot CLI | `graphify copilot install` |
| Aider | `graphify aider install` |
| OpenClaw | `graphify claw install` |
| Factory Droid | `graphify droid install` |
| Trae | `graphify trae install` |
| Trae CN | `graphify trae-cn install` |
| Cursor | `graphify cursor install` |
| Gemini CLI | `graphify gemini install` |
| Hermes | `graphify hermes install` |
| Google Antigravity | `graphify antigravity install` |

**Claude Code** は 2 つのことを行います：Claude にアーキテクチャの質問に答える前に `graphify-out/GRAPH_REPORT.md` を読むように指示する `CLAUDE.md` セクションを書き込み、すべての Glob と Grep 呼び出しの前に発火する **PreToolUse フック**（`settings.json`）をインストールします。ナレッジグラフが存在する場合、Claude は次のメッセージを見ます：_"graphify: Knowledge graph exists. Read graphify-out/GRAPH_REPORT.md for god nodes and community structure before searching raw files."_ ――これにより Claude はすべてのファイルを grep するのではなく、グラフを介してナビゲートします。

**Codex** は `AGENTS.md` に書き込み、すべての Bash ツール呼び出しの前に発火する **PreToolUse フック**を `.codex/hooks.json` にもインストールします――Claude Code と同じ常時有効のメカニズムです。

**OpenCode** は `AGENTS.md` に書き込み、bash ツール呼び出しの前に発火し、グラフが存在する場合にツール出力へグラフのリマインダーを注入する **`tool.execute.before` プラグイン**（`.opencode/plugins/graphify.js` + `opencode.json` 登録）もインストールします。

**Cursor** は `alwaysApply: true` を付けた `.cursor/rules/graphify.mdc` を書き込みます――Cursor はこれをすべての会話に自動的に含めるため、フックは不要です。

**Gemini CLI** はスキルを `~/.gemini/skills/graphify/SKILL.md` にコピーし、`GEMINI.md` セクションを書き込み、`read_file` と `list_directory` ツール呼び出しの前に発火する `BeforeTool` フックを `.gemini/settings.json` にインストールします――Claude Code と同じ常時有効のメカニズムです。

**Aider、OpenClaw、Factory Droid、Trae、Hermes** は同じルールをプロジェクトルートの `AGENTS.md` に書き込みます。これらのプラットフォームはツールフックをサポートしていないため、AGENTS.md が常時有効のメカニズムです。スキルファイルをプラットフォームのグローバルスキルディレクトリにもコピーするには、`graphify install --platform <platform>` を別途実行してください。

**Google Antigravity** は `.agent/rules/graphify.md`（常時有効のルール）と `.agent/workflows/graphify.md`（`/graphify` をスラッシュコマンドとして登録）を書き込みます。Antigravity にはフックに相当する機能がないため、ルールが常時有効のメカニズムです。

**GitHub Copilot CLI** はスキルを `~/.copilot/skills/graphify/SKILL.md` にコピーします。セットアップには `graphify copilot install` を実行してください。

アンインストールは対応するアンインストールコマンドで行います（例：`graphify claude uninstall`）。

**常時有効 vs 明示的トリガー――何が違うのか？**

常時有効のフックは `GRAPH_REPORT.md` を表面化します――これはゴッドノード、コミュニティ、意外なつながりを 1 ページにまとめた要約です。アシスタントはファイル検索の前にこれを読み、キーワードマッチではなく構造に基づいてナビゲートします。これで日常的な質問のほとんどをカバーできます。

`/graphify query`、`/graphify path`、`/graphify explain` はさらに深く踏み込みます：生の `graph.json` をホップごとに辿り、ノード間の正確なパスをトレースし、エッジレベルの詳細（関係タイプ、信頼度スコア、ソース位置）を表面化します。一般的なオリエンテーションではなく、特定の質問をグラフから答えさせたいときに使います。

こう考えてください：常時有効のフックはアシスタントに地図を与え、`/graphify` コマンドはその地図を正確にナビゲートさせます。

<details>
<summary>手動インストール（curl）</summary>

```bash
mkdir -p ~/.claude/skills/graphify
curl -fsSL https://raw.githubusercontent.com/safishamsi/graphify/v4/graphify/skill.md \
  > ~/.claude/skills/graphify/SKILL.md
```

`~/.claude/CLAUDE.md` に追加します：

```
- **graphify** (`~/.claude/skills/graphify/SKILL.md`) - any input to knowledge graph. Trigger: `/graphify`
When the user types `/graphify`, invoke the Skill tool with `skill: "graphify"` before doing anything else.
```

</details>

## `graph.json` を LLM と一緒に使う

`graph.json` は一度にまとめてプロンプトへ貼り付けるためのものではありません。実用的なワークフローは次のとおりです：

1. まず `graphify-out/GRAPH_REPORT.md` で全体像を把握します。
2. `graphify query` を使って、答えたい特定の質問に対する小さめのサブグラフを取得します。
3. コーパス全体を渡す代わりに、その絞り込んだ出力をアシスタントに渡します。

例えば、プロジェクトで graphify を実行した後：

```bash
graphify query "show the auth flow" --graph graphify-out/graph.json
graphify query "what connects DigestAuth to Response?" --graph graphify-out/graph.json
```

出力にはノードラベル、エッジタイプ、信頼度タグ、ソースファイル、ソース位置が含まれます。そのため LLM にとって適切な中間コンテキストブロックになります：

```text
Use this graph query output to answer the question. Prefer the graph structure
over guessing, and cite the source files when possible.
```

アシスタントがツール呼び出しや MCP をサポートしている場合は、テキストを貼り付ける代わりにグラフを直接使ってください。graphify は `graph.json` を MCP サーバーとして公開できます：

```bash
python -m graphify.serve graphify-out/graph.json
```

これにより、アシスタントは `query_graph`、`get_node`、`get_neighbors`、`get_community`、`god_nodes`、`graph_stats`、`shortest_path` といった反復クエリに構造化されたグラフアクセスを利用できます。

> **WSL / Linux 向けの注意：** Ubuntu には `python` ではなく `python3` が入っています。PEP 668 の競合を避けるためプロジェクトの venv にインストールし、`.mcp.json` にはフルパスの venv を指定してください：
> ```bash
> python3 -m venv .venv && .venv/bin/pip install "graphifyy[mcp]"
> ```
> ```json
> { "mcpServers": { "graphify": { "type": "stdio", "command": ".venv/bin/python3", "args": ["-m", "graphify.serve", "graphify-out/graph.json"] } } }
> ```
> また、PyPI パッケージ名は `graphifyy`（y が 2 つ）です —— `pip install graphify` は無関係の別パッケージをインストールします。

## 使い方

```
/graphify                          # カレントディレクトリで実行
/graphify ./raw                    # 特定のフォルダで実行
/graphify ./raw --mode deep        # より積極的な INFERRED エッジ抽出
/graphify ./raw --update           # 変更されたファイルのみ再抽出し、既存グラフにマージ
/graphify ./raw --directed          # 有向グラフを構築（エッジの方向を保持：source→target）
/graphify ./raw --cluster-only     # 既存グラフのクラスタリングを再実行（再抽出なし）
/graphify ./raw --no-viz           # HTML をスキップ、レポート + JSON のみ生成
/graphify ./raw --obsidian                          # Obsidian ボールトも生成（オプトイン）
/graphify ./raw --obsidian --obsidian-dir ~/vaults/myproject  # ボールトを特定のディレクトリに書き込み

/graphify add https://arxiv.org/abs/1706.03762        # 論文を取得、保存、グラフを更新
/graphify add https://x.com/karpathy/status/...       # ツイートを取得
/graphify add https://... --author "Name"             # 元の著者をタグ付け
/graphify add https://... --contributor "Name"        # コーパスに追加した人をタグ付け

/graphify query "アテンションとオプティマイザを結ぶものは？"
/graphify query "アテンションとオプティマイザを結ぶものは？" --dfs   # 特定のパスをトレース
/graphify query "アテンションとオプティマイザを結ぶものは？" --budget 1500  # N トークンで上限設定
/graphify path "DigestAuth" "Response"
/graphify explain "SwinTransformer"

/graphify ./raw --watch            # ファイル変更時にグラフを自動同期（コード：即時、ドキュメント：通知）
/graphify ./raw --wiki             # エージェントがクロール可能な wiki を構築（index.md + コミュニティごとの記事）
/graphify ./raw --svg              # graph.svg をエクスポート
/graphify ./raw --graphml          # graph.graphml をエクスポート（Gephi、yEd）
/graphify ./raw --neo4j            # Neo4j 用の cypher.txt を生成
/graphify ./raw --neo4j-push bolt://localhost:7687    # 実行中の Neo4j インスタンスに直接プッシュ
/graphify ./raw --mcp              # MCP stdio サーバーを起動

# git フック - プラットフォーム非依存、コミット時とブランチ切り替え時にグラフを再構築
graphify hook install
graphify hook uninstall
graphify hook status

# 常時有効のアシスタント指示 - プラットフォーム固有
graphify claude install            # CLAUDE.md + PreToolUse フック（Claude Code）
graphify claude uninstall
graphify codex install             # AGENTS.md（Codex）
graphify codex uninstall
graphify opencode install          # AGENTS.md + tool.execute.before プラグイン（OpenCode）
graphify opencode uninstall
graphify cursor install            # .cursor/rules/graphify.mdc（Cursor）
graphify cursor uninstall
graphify gemini install            # GEMINI.md + BeforeTool フック（Gemini CLI）
graphify gemini uninstall
graphify copilot install           # スキルファイル（GitHub Copilot CLI）
graphify copilot uninstall
graphify aider install             # AGENTS.md（Aider）
graphify aider uninstall
graphify claw install              # AGENTS.md（OpenClaw）
graphify claw uninstall
graphify droid install             # AGENTS.md（Factory Droid）
graphify droid uninstall
graphify trae install              # AGENTS.md（Trae）
graphify trae uninstall
graphify trae-cn install           # AGENTS.md（Trae CN）
graphify trae-cn uninstall
graphify hermes install            # AGENTS.md（Hermes）
graphify hermes uninstall
graphify antigravity install       # .agent/rules + .agent/workflows（Google Antigravity）
graphify antigravity uninstall

# ターミナルから直接グラフをクエリ（AI アシスタント不要）
graphify query "アテンションとオプティマイザを結ぶものは？"
graphify query "認証フローを表示" --dfs
graphify query "CfgNode とは？" --budget 500
graphify query "..." --graph path/to/graph.json
graphify path "DigestAuth" "Response"       # 2ノード間の最短パス
graphify explain "SwinTransformer"           # ノードの平易な説明

# ターミナルからコンテンツを追加してグラフを更新
graphify add https://arxiv.org/abs/1706.03762          # 論文を取得、./raw に保存、グラフを更新
graphify add https://... --author "Name" --contributor "Name"

# 増分更新とメンテナンス
graphify watch ./src                         # コード変更時に自動再構築
graphify update ./src                        # コードファイルのみ再抽出、LLM 不要
graphify cluster-only ./my-project           # 既存の graph.json でクラスタリングのみ再実行
graphify benchmark                           # 生コーパスを読む場合と比較したトークン削減量を測定
graphify benchmark path/to/graph.json        # 特定のグラフファイルをベンチマーク

# グラフのフィードバックループのため Q&A 結果を保存
graphify save-result --question "Q" --answer "A" --type code
```

あらゆるファイルタイプの組み合わせで動作します：

| タイプ | 拡張子 | 抽出方法 |
|------|-----------|------------|
| コード | `.py .ts .js .jsx .tsx .go .rs .java .c .cpp .cc .cxx .h .hpp .rb .cs .kt .kts .scala .php .blade.php .swift .lua .toc .zig .ps1 .ex .exs .m .mm .jl .vue .svelte .dart` | tree-sitter による AST（Dart は regex）+ コールグラフ + docstring/コメントの根拠; `.blade.php` は Blade ディレクティブ（`@include`）、Livewire コンポーネント、`wire:click` バインディングも抽出 |
| ドキュメント | `.md .txt .rst` | Claude による概念 + 関係性 + 設計根拠 |
| Office | `.docx .xlsx` | Markdown に変換した後 Claude で抽出（`pip install graphifyy[office]` が必要） |
| 論文 | `.pdf` | 引用マイニング + 概念抽出 |
| 画像 | `.png .jpg .jpeg .webp .gif .svg` | Claude Vision - スクリーンショット、図、任意の言語 |
| ビデオ / オーディオ | `.mp4 .mov .mkv .webm .avi .m4v .mp3 .wav .m4a .ogg` | faster-whisper でローカル転写し、Claude の抽出パイプラインに投入（`pip install graphifyy[video]` が必要） |
| YouTube / URL | 任意のビデオ URL | yt-dlp で音声をダウンロードし、同じ Whisper パイプラインで処理（`pip install graphifyy[video]` が必要） |

## 動画・音声コーパス

コードやドキュメントと一緒に動画・音声ファイルをコーパスフォルダに入れるだけで、graphify が自動的に取り込みます：

```bash
pip install 'graphifyy[video]'   # 初回のみ
/graphify ./my-corpus            # 見つかった動画・音声ファイルを転写
```

YouTube 動画（または公開されている任意の動画 URL）を直接追加することもできます：

```bash
/graphify add <video-url>
```

yt-dlp が音声のみをダウンロードし（高速・小容量）、Whisper がローカルで転写し、その転写結果は他のドキュメントと同じ抽出パイプラインに投入されます。転写結果は `graphify-out/transcripts/` にキャッシュされるため、再実行時にはすでに転写済みのファイルはスキップされます。

技術的な内容の精度を上げるには、より大きなモデルを使用してください：

```bash
/graphify ./my-corpus --whisper-model medium
```

音声がマシンの外に出ることはありません。転写はすべてローカルで実行されます。

## 得られるもの

**ゴッドノード** - 最高次数の概念（すべてが接続するもの）

**意外なつながり** - 複合スコアでランク付け。コード-論文のエッジはコード-コードよりも高くランクされます。各結果には平易な英語の理由が含まれます。

**推奨される質問** - グラフがユニークに答えられる 4〜5 の質問

**「なぜ」** - docstring、インラインコメント（`# NOTE:`、`# IMPORTANT:`、`# HACK:`、`# WHY:`）、ドキュメントからの設計根拠が `rationale_for` ノードとして抽出されます。コードが何をするかだけでなく――なぜそのように書かれたか。

**信頼度スコア** - すべての INFERRED エッジには `confidence_score`（0.0〜1.0）があります。何が推測されたかだけでなく、モデルがどれだけ確信していたかもわかります。EXTRACTED エッジは常に 1.0 です。

**意味的類似性エッジ** - 構造的接続のないクロスファイル概念リンク。互いを呼び出さずに同じ問題を解いている 2 つの関数、同じアルゴリズムを記述しているコード内のクラスと論文内の概念など。

**ハイパーエッジ** - ペアワイズエッジでは表現できない 3+ ノードを接続するグループ関係。共有プロトコルを実装するすべてのクラス、認証フロー内のすべての関数、論文セクションから 1 つのアイデアを形成するすべての概念など。

**トークンベンチマーク** - 実行ごとに自動的に出力されます。混合コーパス（Karpathy リポジトリ + 論文 + 画像）で、生ファイルを読むのに比べて 1 クエリあたり **71.5 倍** 少ないトークン。最初の実行で抽出とグラフ構築を行います（これにはトークンがかかります）。以降のクエリはすべて生ファイルではなくコンパクトなグラフを読みます――ここで節約が複利的に効いてきます。SHA256 キャッシュにより、再実行時は変更されたファイルのみ再処理されます。

**自動同期** (`--watch`) - バックグラウンドターミナルで実行し、コードベースが変更されるとグラフが自動的に更新されます。コードファイルの保存は即座の再構築をトリガーします（AST のみ、LLM なし）。ドキュメント/画像の変更は、LLM の再パスのために `--update` を実行するよう通知します。

**Git フック** (`graphify hook install`) - post-commit と post-checkout フックをインストールします。コミットごと、ブランチ切り替えごとにグラフが自動的に再構築されます。再構築が失敗した場合、フックは非ゼロコードで終了するため、git がエラーを表面化し、静かに続行することはありません。バックグラウンドプロセスは不要です。

**Wiki** (`--wiki`) - コミュニティごとおよびゴッドノードごとの Wikipedia スタイルの Markdown 記事と、`index.md` エントリポイント。任意のエージェントを `index.md` に向ければ、JSON をパースする代わりにファイルを読むことでナレッジベースをナビゲートできます。

## 実例

| コーパス | ファイル数 | 削減率 | 出力 |
|--------|-------|-----------|--------|
| Karpathy リポジトリ + 論文5本 + 画像4枚 | 52 | **71.5x** | [`worked/karpathy-repos/`](worked/karpathy-repos/) |
| graphify ソース + Transformer 論文 | 4 | **5.4x** | [`worked/mixed-corpus/`](worked/mixed-corpus/) |
| httpx（合成 Python ライブラリ） | 6 | ~1x | [`worked/httpx/`](worked/httpx/) |
| 小規模なドキュメントパイプライン（Python + markdown） | 7 | — | [`worked/example/`](worked/example/) —— 自分で実行できるスターターコーパス |

トークン削減はコーパスサイズに応じてスケールします。6 ファイルはいずれにせよコンテキストウィンドウに収まるため、そこでのグラフの価値は圧縮ではなく構造的明瞭さです。52 ファイル（コード + 論文 + 画像）では 71 倍以上が得られます。各 `worked/` フォルダには生の入力ファイルがあり（`worked/karpathy-repos/` を除く — こちらは自分で clone・ダウンロードする外部の GitHub リポジトリと arXiv 論文で構成されています、詳細は同フォルダの README を参照。`worked/example/` も除く — こちらは自分で実行するためのスターターコーパス）、実際の出力（`GRAPH_REPORT.md`、`graph.json`）から数字を検証できます。

## プライバシー

graphify はドキュメント、論文、画像の意味的抽出のために、ファイル内容を AI コーディングアシスタントの基盤モデル API に送信します――Anthropic（Claude Code）、OpenAI（Codex）、またはプラットフォームが使用するプロバイダーです。コードファイルは tree-sitter AST を介してローカルで処理されます――コードに関してはファイル内容がマシンから出ることはありません。テレメトリ、利用追跡、分析は一切ありません。ネットワーク呼び出しは抽出中のプラットフォームのモデル API への呼び出しのみで、あなた自身の API キーを使用します。

## 技術スタック

NetworkX + Leiden（graspologic） + tree-sitter + vis.js。意味的抽出は Claude（Claude Code）、GPT-4（Codex）、またはプラットフォームが実行するモデルを介して行われます。動画の文字起こしは faster-whisper + yt-dlp（オプション、`pip install graphifyy[video]`）。Neo4j は不要、サーバーも不要、完全にローカルで実行されます。

## graphify を基盤にした構築 — Penpax

[**Penpax**](https://safishamsi.github.io/penpax.ai) は graphify の上に構築されたエンタープライズレイヤーです。graphify がファイルフォルダをナレッジグラフに変換するのに対し、Penpax は同じグラフをあなたの仕事全体に —— 継続的に —— 適用します。

| | graphify | Penpax |
|---|---|---|
| 入力 | ファイルフォルダ | ブラウザ履歴、会議、メール、ファイル、コード —— すべて |
| 実行 | オンデマンド | バックグラウンドで継続実行 |
| 対象範囲 | プロジェクト単位 | 仕事全体 |
| クエリ | CLI / MCP / AI スキル | 自然言語、常時稼働 |
| プライバシー | デフォルトでローカル | 完全オンデバイス、クラウド不使用 |

弁護士、コンサルタント、経営者、医師、研究者など —— 何百もの会話やドキュメントにまたがる仕事をしていて、それを完全には再構築できない人たちのために作られています。

**無料トライアルは近日公開予定。** [ウェイトリストに参加する →](https://safishamsi.github.io/penpax.ai)

## 次に作っているもの

graphify はグラフレイヤーです。Penpax はその上に構築する常時稼働レイヤーで、会議、ブラウザ履歴、ファイル、メール、コードを 1 つの継続的に更新されるナレッジグラフに接続するオンデバイスのデジタルツインです。クラウド不使用、あなたのデータで学習することもありません。[ウェイトリストに参加する。](https://safishamsi.github.io/penpax.ai)

## スター履歴

[![Star History Chart](https://api.star-history.com/svg?repos=safishamsi/graphify&type=Date)](https://star-history.com/#safishamsi/graphify&Date)

<details>
<summary>コントリビューション</summary>

**実例** は最も信頼を築くコントリビューションです。実際のコーパスで `/graphify` を実行し、出力を `worked/{slug}/` に保存し、グラフが正しく捉えたもの・間違えたものを評価する正直な `review.md` を書き、PR を提出してください。

**抽出バグ** - 入力ファイル、キャッシュエントリ（`graphify-out/cache/`）、何が見逃された/捏造されたかを添えて issue を開いてください。

モジュールの責任と言語の追加方法については [ARCHITECTURE.md](ARCHITECTURE.md) を参照してください。

</details>
