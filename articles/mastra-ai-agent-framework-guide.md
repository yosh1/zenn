---

title: "Mastra入門 〜AIエージェント開発ツールの概要と使い方〜"
emoji: ":desktop_computer:" 
type: "tech" 
topics: ["AIエージェント", "AI", "Mastra", "TypeScript", "Node.js"]
published: true

---

## Mastraとは

Mastra（マストラ）は、**AIエージェント開発のためのオープンソースフレームワーク**です。TypeScript（JavaScript）で実装され、大規模言語モデル（LLM）を活用したAIアプリケーションや機能を効率的に構築できます。

![mastra.jpg](/images/mastra.jpg)

例えば、対話型の「エージェント」（自律的にタスクを実行するAIシステム）をシンプルなコードで実装でき、ローカル環境やクラウド上で動作させることが可能です。

### 主な特徴

Mastraでは、エージェントに**ツール**や**ワークフロー**などの機能を組み込むことで、言語モデルに外部の操作能力を与えることができます。エージェントはユーザーからの指示に応じて、これらのツールを活用したり、定義されたワークフロー（処理手順）に従ったりしながら、自律的に処理を進めていきます。

また、OpenAIのGPT-4やAnthropicのClaude、GoogleのGeminiなど**複数のLLMプロバイダに対応**しており、コードの設定を変更するだけでバックエンドのAIモデルを簡単に切り替えられます。これはVercel社のAI SDKをベースにしているおかげで、統一されたインターフェースで様々なモデルを扱えるためです。

### 主要コンポーネント

Mastraが提供する主なコンポーネントには以下の4つがあります。

- **agents**
  ![agents.png](/images/agents.png)
  - 言語モデルが一連の動作を選んでタスクをこなすための主体
  - ツールやワークフロー、知識ベースを組み合わせて動く
  - 必要に応じて自分で関数や外部APIを呼び出して情報を取得・処理

- **workflows**
  ![workflows.png](/images/workflows.png)
  - 安定したグラフベースの**状態管理システム**
  - 複数のステップにわたるタスクを定義できる
  - ループ処理や条件分岐、他のワークフローの組み込み、エラー時のリトライ、人間からの入力待ちなどが可能
  - 各ステップの動作履歴も標準で記録される

- **rag**
  ![rags.png](/images/rags.png)
  - 日本語では「検索強化型生成」とも呼ばれる機能
  - エージェントに外部の知識ベースを参照させる仕組み
  - 大量のドキュメントやデータをベクトルデータベースに登録
  - 質問に答える際に関連する情報を検索・取得して回答に反映

- **Ops**
  ![ops.png](/images/ops.png)
  - エージェントやワークフローから使える**機能拡張用の関数**
  - 各ツールは入力・出力の形式（スキーマ）を持つ
  - 実行時に特定の処理（外部APIサービス呼び出しやデータベース操作など）を行う

## インストールとセットアップ

### 準備するもの

- 開発マシンに **Node.js（v20以上）** が入っていること
- OpenAIやAnthropicなど **LLMプロバイダのAPIキー** を用意

### セットアップの手順

#### 1. プロジェクトを作成

ターミナルで次のコマンドを実行します。

```bash
npx create-mastra@latest
```

※ `npm create mastra@latest` や `yarn create mastra` でも同じようにプロジェクト作成ウィザードが起動します。

コマンドを実行すると対話形式で以下のような質問が表示されるので、順番に入力・選択してください。

- プロジェクト名: 好きな名前を入力（e.g. my-mastra-app）
- インストールするコンポーネント: **Agents**, **Tools**, **Workflows** から必要なものを選ぶ
- デフォルトで使うLLMプロバイダ: **OpenAI** （おすすめ）やAnthropicなど使いたいモデル提供元を選ぶ
- サンプルコードを含めるか: **Yes** を選ぶと、エージェントやツールのサンプル実装が自動で作られます

#### 2. APIキーの設定

プロジェクトフォルダ内の`.env`ファイルを開いて、自分のLLMプロバイダのAPIキーを設定します。

```plaintext
OPENAI_API_KEY=sk-XXXXXXXX...
```

※ 上記の`sk-...`の部分に実際の自分のAPIキー文字列を入力してください。

#### 3. 開発サーバーを起動

セットアップが完了したら、以下のコマンドでMastraの開発用サーバーを起動します。

```bash
npm run dev
```

初回実行時は依存関係のビルドなどで少し時間がかかる場合があります。サーバーが起動すると、Mastraによってエージェント用のREST APIエンドポイントがローカル環境（デフォルトではポート4111）に作られます。

> :bulb: **補足** `create-mastra`実行時にサンプルコードを含めた場合は、最初から簡単なエージェント実装が含まれています。その場合、APIキーを設定してサーバーを起動するだけでサンプルエージェントをすぐ試すことができます。

## 基本的な使い方

それでは、Mastraでエージェントを作って使う基本的な流れを見ていきます。
ここでは例として、exampleにある「指定した都市の現在の天気を教えてくれるエージェント」を見ていきます。

### 1. ツールを作成

まず、エージェントに外部の天気情報を取得させるための**ツール**を用意します。
ツールとは上述した通り外部機能を実行するための部品で、今回は**Open-Meteo**という無料の天気APIからデータを取得する関数をツール化します。

具体的なコードは省略しますが、`createTool`関数を使って「都市名」を入力すると「現在の気温や湿度など」を返すようなロジックを実装し、ツール名を`weatherTool`としてエクスポートします。

※このツール内部ではfetchを使ってOpen-Meteo APIにリクエストを送り、結果を加工して返す処理を行っています。

### 2. エージェントを定義

ツールが用意できたら、次に**エージェント**本体を定義します。Mastraでは`Agent`クラスを使って新しいエージェントを作れます。

以下は天気情報エージェントのコード例です。

```typescript
import { openai } from "@ai-sdk/openai";
import { Agent } from "@mastra/core/agent";
import { weatherTool } from "../tools/weather-tool";

export const weatherAgent = new Agent({
 name: "Weather Agent",
 instructions: `You are a helpful weather assistant that provides accurate weather information.

 Your primary function is to help users get weather details for specific locations. When responding:
 - Always ask for a location if none is provided
 - Include relevant details like humidity, wind conditions, and precipitation
 - Keep responses concise but informative

 Use the weatherTool to fetch current weather data.`,
 model: openai("gpt-4o-mini"),
 tools: { weatherTool },
});
```

このコードでは、`new Agent({...})`でエージェントを生成しています。
主な設定項目は以下の通りです。

- `name`: エージェントの名前（任意の識別用）
- `instructions`: エージェントに与えるシステムメッセージ（振る舞いに関する指示）
- `model`: 使う言語モデルを指定
- `tools`: エージェントが使えるツールをオブジェクトとして渡す

### 3. エージェントを登録

エージェントを定義できたら、最後にこのエージェントをMastraに登録します。プロジェクトのエントリーポイントである`src/mastra/index.ts`（または生成された場所）を開いて、次のようにエージェントを登録します。

```typescript
import { Mastra } from "@mastra/core";
import { weatherAgent } from "./agents/weather";

export const mastra = new Mastra({
 agents: { weatherAgent },
});
```

### 4. エージェントを実行

それでは、実際にエージェントを動かしてみましょう。開発サーバー(`npm run dev`)が起動した状態で、RESTエンドポイントにリクエストを送ります。

天気エージェントの場合、`/api/agents/weatherAgent/generate`がエンドポイントです。以下は`curl`コマンドを使って「ロンドンの天気」を尋ねる例です。

```bash
curl -X POST http://localhost:4111/api/agents/weatherAgent/generate \
 -H "Content-Type: application/json" \
 -d '{"messages": ["What is the weather in London?"]}'
```

上記のコマンドを実行すると、Mastra経由でエージェントにメッセージが渡されます。エージェントは内部で`weatherTool`を使ってロンドンの現在の天気データを取得し、それに基づいた回答を生成します。

> :bulb: **補足** Mastraのエージェントはプログラム内から直接呼び出すこともできます。例えばNode.jsコード上で `const agent = await mastra.getAgent("weatherAgent")` のように取得し、`agent.generate("What is the weather in London?")` と呼ぶことで、同様に応答を得ることも可能です。

## 応用と今後の可能性

Mastraを使うと、いろんな場面で**自分で考えて動くAIエージェント**を作れます。いくつかの例を見てみましょう。

### 具体的な応用例

- **カスタマーサポートの自動化**
  - 製品のFAQやナレッジベースと連携
  - 顧客からの問い合わせに自動で回答
  - RAG機能を使って社内ドキュメントから回答を検索

- **Web情報のクローリングと要約**
  - 指定したウェブサイトやニュースソースを自動で巡回
  - 情報を集めて、要点をまとめてレポート
  - 営業リストの連絡先収集や市場調査などに応用

- **ドキュメント生成・分析**
  - 会議の議事録や財務レポートを自動生成
  - 大量のテキストデータから重要ポイントを抜き出す
  - 音声認識と組み合わせて会議音声からリアルタイムに要約を作成

- **プログラミング支援**
  - コードの自動生成やバグ修正提案
  - コードレビューコメントの作成
  - 複数のファイルにまたがる変更やテストケースの生成

### 高度な機能

Mastraには高度な機能が備わっているので、さらに発展的な使い方ができます。

- エージェント同士が協力する **マルチエージェント** 構成
- タスクを一時停止・再開できる **ワークフローグラフ** の構築
- 対話履歴を長期記憶する **メモリ** 機能
- LLMの応答品質を評価する **Evals（自動評価）**

例えば、あるエージェントが判断に迷ったら別のエージェントにサブタスクを依頼するといった協力プレイも、ワークフローの設定次第で実現できます。

また、エージェントの動作はすべて記録（可視化）されるので、どのように判断・ツール実行したかを後から分析でき、**動作の追跡（オブザーバビリティ）**が高い点も開発運用上のメリットです。

### 今後の展望

Mastraは公開直後から開発者コミュニティで大きな注目を集めています。実際、Hacker Newsで話題になりGitHubのスター数がわずか数日で1,500から7,500以上に急増するといった盛り上がりも見せました。

裏側には、Reactフレームワーク「Gatsby」の開発者たちが参画していて、使いやすさや拡張性にも配慮されています。

今後も外部サービス連携の拡充やコミュニティ主導の拡張が進み、機能はさらに充実していくでしょう。

最近では、ChatGPTのような単一の対話AIから一歩進み、**自分で考えて動くAIエージェント** が次のトレンドになるとも言われています。Mastraのような開発フレームワークを使えば、そうした最先端のAIエージェントをいち早く手元で試し、独自のプロジェクトに取り入れることができます。

ぜひ本記事を参考に、MastraでAIエージェント開発の第一歩を踏み出してみてください。自分で考えて動くAIがもたらす新しい可能性を、初心者の方でも実感できるはずです。

---

私が代表を務める [エクステム株式会社](https://xtem.jp) では、このようなAIエージェントに関する研修やコンサルティングを行っています。

もしご興味のある企業がおられましたら、ぜひお問い合わせくださいませ。