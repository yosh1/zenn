---
title: "GitHub Copilot の Agent Mode・CLI・Coding Agent・SDK を使い分けて、開発の進め方を組み直した話"
emoji: "🤖"
type: "tech"
topics: ["GitHubCopilot", "GitHub", "VSCode", "AI", "SDK"]
published: true
publication_name: "preferred"
---

## はじめに

2026年に入ってから、GitHub Copilot はもはやコード補完ツールではなくなった。

Agent Mode が VS Code と JetBrains の両方で GA になり、Copilot CLI はターミナル上のエージェント環境に進化し、Coding Agent は Issue を渡すだけで PR を作ってくる。SDK でアプリに組み込むこともできるようになった。便利そうなのはわかるけど、どこから手をつけるか迷う——そんな状態だった。

実際にプロジェクトで使い分けてみて、自分なりの使い所が見えてきたので整理してみる。

## エージェント機能の全体像

まず、2026年4月時点で使えるエージェント系の機能をざっと並べておく。

| 機能 | 動作環境 | 特徴 | 向いている作業 |
|------|----------|------|----------------|
| **Agent Mode** | VS Code / JetBrains | IDE 内でファイル作成・編集・テスト実行まで自律的に行う | 機能実装、リファクタリング、バグ修正 |
| **Copilot CLI** | ターミナル | 自然言語でタスクを指示、`/fleet` で並列実行も可能 | スクリプト作成、環境構築、ドキュメント生成 |
| **Coding Agent** | GitHub（クラウド） | Issue を割り当てるだけで、独立した VM 上でコード変更＋PR 作成 | 定型的な修正、小〜中規模の機能追加 |
| **Copilot SDK** | 任意のアプリ | Copilot のエージェント実行ループをアプリに組み込める | 社内ツール、業務特化の自動化 |

得意領域が違うので、タスクの性質で使い分けるのが肝になる。

## Agent Mode — IDE の中で完結するもう一人の開発者

### 基本の使い方

![VS Code の Copilot Agent Mode](https://code.visualstudio.com/assets/blogs/2025/02/24/agent-mode.png)
*出典: [Introducing GitHub Copilot agent mode (preview) - VS Code Blog](https://code.visualstudio.com/blogs/2025/02/24/introducing-copilot-agent-mode)*

VS Code のチャットパネルで「Agent」モードを選んで、やりたいことを伝えるだけ。Agent Mode はどのファイルを触るべきか自分で判断して、コード変更からターミナルコマンドの実行まで自律的に繰り返す。

たとえば「ユーザープロフィールページに、最終ログイン日時を表示する機能を追加して」と投げると、こんな流れで進む。

1. 関連するファイル（モデル、コントローラー、ビュー）を特定
2. データベースのマイグレーションファイルを生成
3. 各ファイルにコードを追加
4. テストを実行して動作確認

### Plan Mode で先に設計を見る

Agent Mode で最初に覚えたいのが **Plan Mode**。`/plan` と打つか「まず計画を立てて」と伝えると、コード変更に入る前に「何をどう変えるか」の設計図を出してくれる。

```
/plan ユーザープロフィールページに最終ログイン日時を表示する機能を追加
```

![Agent Mode の内部構造](https://code.visualstudio.com/assets/blogs/2025/02/24/diagram.png)
*出典: [Introducing GitHub Copilot agent mode (preview) - VS Code Blog](https://code.visualstudio.com/blogs/2025/02/24/introducing-copilot-agent-mode)*

これが地味に効く。Agent Mode は優秀だけど、こちらの意図と微妙にずれた方向に進むことがある。先に計画を見てから実行に移す癖をつけるだけで、手戻りがだいぶ減った。

### カスタムインストラクションで精度を上げる

プロジェクトのルートに `.github/copilot-instructions.md` を置くと、コード生成の指針を渡せる。

```markdown
## コーディング規約
- TypeScript を使用し、`any` 型は禁止
- コンポーネントは関数コンポーネントのみ
- テストは Vitest で記述
- CSS は Tailwind CSS を使用

## プロジェクト構成
- `src/components/` - UIコンポーネント
- `src/hooks/` - カスタムフック
- `src/lib/` - ユーティリティ関数
```

これだけで生成されるコードの質がぐっと安定する。チーム開発ならリポジトリに入れておけば全員が同じ恩恵を受けられる。

### Agent Mode が特に効くケース

使い込んでみて、特に助かったのは以下のパターンだ。

**リファクタリング**

「この関数を小さく分割して、それぞれにテストを書いて」みたいな指示が得意。複数ファイルにまたがる変更を、整合性を保ちながら進めてくれる。

**テストの追加**

「このモジュールのユニットテストを書いて」と投げるだけで、テストファイルの作成からケースの実装、実行確認まで一気に進む。カバレッジを上げたいときに重宝する。

**エラーの調査と修正**

スタックトレースを貼って「このエラーを修正して」と頼むと、セマンティックコード検索で関連コードを拾ってきて、原因の特定から修正まで進めてくれる。キーワード一致ではなく概念的な関連で探すので、「ログインのバグ」と伝えるだけで認証ミドルウェアやセッション管理のコードにたどり着く。

## Copilot CLI — ターミナルで動くエージェント環境

### GA になった Copilot CLI

2026年2月に GA になった Copilot CLI は、もうターミナルでの質問応答ツールとは呼べない。計画を立てて、コードを書いて、レビューして、セッション間で記憶を引き継ぐ。ターミナル上のフルスペックなエージェント環境だ。

インストールは簡単。

```bash
# npm でインストール
npm install -g @githubnext/github-copilot-cli

# または brew
brew install gh
gh extension install github/gh-copilot
```

### 自然言語でコマンドを組み立てる

やりたいことを伝えるとコマンドを提案してくれる。

```bash
# 例：直近1週間で変更されたファイルの一覧を取得
gh copilot suggest "直近1週間で変更されたファイルの一覧を表示"
```

複雑なコマンドを組み立てるときに便利だけど、ここで止まるのはもったいない。

### CLI のエージェントモード

Copilot CLI でもエージェントモードが使える。ターミナル上でタスクを指示すると、ファイルの読み書きやコマンド実行を自律的に回しながらタスクを片付けてくれる。

```bash
# エージェントモードでタスクを実行
ghcs "このプロジェクトにESLintとPrettierをセットアップして、既存コードをフォーマットして"
```

### /plan で見通しを立てる

VS Code の Agent Mode と同じく、CLI でも `/plan` が使える。

```bash
ghcs "/plan Next.js プロジェクトに Storybook を導入する"
```

計画を見て問題なければ実行に進む。特にインフラ系や CI/CD パイプラインの構築みたいな影響範囲が広い作業では、必ず Plan を確認してから進めるようにしている。

### MCP サーバーとの連携

Copilot CLI は MCP（Model Context Protocol）サーバーをサポートしている。外部ツールやサービスをつなげられる。

```json
// .github/copilot-mcp.json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

たとえばデータベースの MCP サーバーをつないでおけば、「ユーザーテーブルのスキーマを確認して、それに合わせた CRUD API を作って」みたいな指示が通る。

### `/fleet` で並列実行する

CLI の中でも特に面白いのが `/fleet` コマンド。タスクを投げると、オーケストレーターが作業を分解して複数のサブエージェントに並列で振り分けてくれる。

```bash
copilot /fleet "APIモジュールのドキュメントを作成して。
- docs/authentication.md (認証フロー)
- docs/endpoints.md (エンドポイント仕様)
- docs/errors.md (エラーコード)
- docs/index.md (上記3つへのリンク、他のタスク完了後に実行)"
```

![Copilot CLI の画面](https://github.blog/wp-content/uploads/2026/02/554396112-02305897-302e-435b-862b-616faea6a7f5.png)
*出典: [Run multiple agents at once with /fleet in Copilot CLI - GitHub Blog](https://github.blog/ai-and-ml/github-copilot/run-multiple-agents-at-once-with-fleet-in-copilot-cli/)*

依存関係を明示しておけば、独立したタスクは同時に走り、依存するものは順番に処理される。ドキュメント生成みたいな「量はあるけど独立した作業」にはかなり効く。

### CLI が特に活きる場面

**環境構築の自動化**

新しいプロジェクトのセットアップや、既存プロジェクトへのツール導入。Docker Compose の設定から CI/CD のパイプライン構築まで、ターミナルで完結する作業をそのまま任せられる。

**書き捨てスクリプト**

「デプロイ用のスクリプトを書いて」「ログをパースして日別のエラー数を集計するスクリプト」みたいな一発もの。こういうのこそ AI に投げたい作業だ。

**git 操作**

複雑な git コマンドを調べる手間が省ける。「main ブランチとの差分で、テストファイルだけ一覧表示して」がそのまま通る。

## Coding Agent — Issue から PR を自動生成する

### 仕組み

Coding Agent は GitHub 上で動くクラウドベースのエージェント。Issue に Copilot をアサインすると、こんな流れで PR を作ってくれる。

1. 専用の VM が起動し、リポジトリをクローン
2. GitHub のコード検索を使った RAG でコードベースを分析
3. 必要なコード変更を実装
4. ドラフト PR を作成

全部バックグラウンドで非同期に進むので、自分は別の作業に集中できる。

### 使い方

![GitHub Copilot Coding Agent](https://github.blog/wp-content/uploads/2025/05/PadawayBlog_Assignee_006.jpg)
*出典: [GitHub Copilot: Meet the new coding agent - GitHub Blog](https://github.blog/news-insights/product-news/github-copilot-meet-the-new-coding-agent/)*

GitHub の Issue 画面で Assignees に `Copilot` を追加するだけ。VS Code のチャットから「この Issue を Copilot に任せて」と指示してもいい。

コツは **Issue を具体的に書くこと**。

```markdown
## やること
- `src/components/Header.tsx` のナビゲーションに「お知らせ」リンクを追加
- リンク先は `/notifications` 
- 未読のお知らせがある場合はバッジを表示する
- バッジの数値は `useNotifications` フックから取得

## 参考
- 既存のナビゲーション項目の実装を参考にすること
- スタイリングは Tailwind CSS を使用
```

![Coding Agent へのフィードバック](https://github.blog/wp-content/uploads/2025/05/copilot1.png)
*出典: [GitHub Copilot: Meet the new coding agent - GitHub Blog](https://github.blog/news-insights/product-news/github-copilot-meet-the-new-coding-agent/)*

曖昧な指示だと的外れな PR が上がってくるけど、ここまで具体的に書けばかなり精度の高い変更が出てくる。

### Jira 連携（2026年3月〜）

2026年3月に Jira との連携がパブリックプレビューで追加された。Jira の Issue を Coding Agent に割り当てると、GitHub リポジトリに対してドラフト PR が自動生成される。すでに Jira でタスク管理しているチームなら、ワークフローを変えずに AI の自動化を取り入れられる。

### Coding Agent に向いているタスク

経験上、安定していい結果が出るのはこのあたり。

- **定型的な修正** — 依存パッケージのアップデート、非推奨 API の置き換え
- **小規模な機能追加** — 新しいエンドポイントや UI コンポーネントの追加
- **ドキュメントの更新** — README やコメントの更新、型定義の追加

逆に、設計判断が要る大きな変更や、アーキテクチャを組み替えるようなリファクタリングには向かない。「何をすればいいか明確なタスク」を確実にこなすのが得意、という理解でいい。

## Copilot SDK — アプリにエージェントを埋め込む

まだテクニカルプレビューだけど、Copilot SDK を使うと CLI の裏側で動いているのと同じ実行ループ（計画→ツール呼び出し→マルチターン実行）に、自分のアプリからアクセスできる。

社内向けの AI アシスタント GUI を作ったり、特定の業務フローに特化した自動化ツールを組んだり。Copilot の頭脳を借りつつ、インターフェースは自由に設計できるのが面白い。

MCP サーバーとの統合もサポートされていて、GitHub の MCP サーバーをつなげばリポジトリや Issue、PR の操作までエージェントに任せられる。プレビュー段階なので本番投入は早いけど、社内ツールを作りたい人は触っておいて損はないと思う。

## どう使い分けるか

日々の開発では、だいたいこんな基準で振り分けている。

### 判断フロー

```
タスクを受け取る
  │
  ├─ IDE でコードを見ながら進めたい？
  │    └─ Yes → Agent Mode
  │
  ├─ ターミナルで完結する作業？
  │    └─ Yes → Copilot CLI（並列なら /fleet）
  │
  ├─ 明確に定義された Issue として切り出せる？
  │    └─ Yes → Coding Agent
  │
  ├─ 自前のアプリに AI を組み込みたい？
  │    └─ Yes → Copilot SDK
  │
  └─ 上記に当てはまらない
       └─ まず Agent Mode で /plan から始める
```

### 組み合わせの例

たとえば「API に新しいエンドポイントを追加する」というタスクなら、こう進める。

1. **Coding Agent** に Issue をアサインして、基本的な CRUD エンドポイントの雛形を作ってもらう
2. 生成された PR を確認しつつ、**Agent Mode** でビジネスロジックやバリデーションを追加
3. **Copilot CLI** でテスト用のシードデータ投入スクリプトを作成

1つのタスクでも、工程ごとにツールを変えるとそれぞれの得意なところを引き出せる。

## モデルの使い分け

2026年の Copilot は複数のモデルを選べる。Claude Opus 4.6、Claude Sonnet 4.6、GPT-5.3-Codex、Gemini 3 Pro など。

試した範囲での感触はこんな感じ。

| 用途 | おすすめモデル | 理由 |
|------|---------------|------|
| 設計・アーキテクチャの相談 | Claude Opus 4.6 | 論理的な構成力と長いコンテキストの把握が強い |
| 日常的なコード実装 | Claude Sonnet 4.6 | 速度と品質のバランスがよく、ストレスなく使える |
| 高速な補完・軽微な修正 | Claude Haiku 4.5 | 応答が速く、Premium Request の消費も少ない |

![VS Code のモデルピッカー](https://code.visualstudio.com/assets/docs/copilot/language-models/model-dropdown-change-model.png)
*出典: [AI language models in VS Code - VS Code Docs](https://code.visualstudio.com/docs/copilot/customization/language-models)*

モデルの切り替えは VS Code のチャットパネルや CLI の `--model` フラグから。タスクの重さに応じて変えると、Premium Request の消費を抑えつつ品質を保てる。

## 実践で得た Tips

**copilot-instructions.md を育てる**

プロジェクト固有のルールを書き溜めていくと、Copilot の出力が回を追うごとに的確になる。「使ってみて微妙だった点」を都度追記するのが一番手っ取り早い。

**確認してから実行、を徹底する**

Agent Mode も CLI も、変更を確認してから適用する運用が安全。特に本番に影響するインフラ系のコマンドは、必ず Plan Mode を通してから実行している。

**Issue は「実装仕様書」として書く**

Coding Agent の精度は Issue の書き方に直結する。「〇〇を改善して」ではなく、変更対象のファイルパスや期待する振る舞いまで書く。これだけでドラフト PR の質が全然違う。

**セマンティック検索を活かす**

「認証周りの処理をすべて見せて」みたいな概念的な指示を出すと、キーワード検索では引っかからないコードまで拾ってくれる。コードベースの全体像を掴むのにも使える。

**小さいところから始める**

いきなり大きなタスクを任せるより、テスト追加やドキュメント更新から。Copilot の癖がわかってきたら、徐々に任せる範囲を広げていくのが無理がない。

## まとめ

Agent Mode は IDE での対話的な開発に、CLI はターミナル作業の自動化に、Coding Agent は明確なタスクの非同期処理に、SDK はアプリへの組み込みに。使い分けを意識するだけで、開発の進め方がだいぶ変わった。

これらを組み合わせて自分なりのワークフローを組み立てていくのが、Copilot を使いこなす面白さだと思う。まずは普段のタスクに「どの機能が向いてるか」を考えるところから試してみてほしい。
