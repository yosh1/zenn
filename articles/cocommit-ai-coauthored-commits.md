---

title: "自分のコミットの39%がClaudeとの共著だった。数えるツールを作って公開した"
emoji: "🤝"
type: "tech"
topics: ["ClaudeCode", "GitHub", "GitHubActions", "AI", "OSS"]
published: true

---

## 自分が実際どれだけAIと書いているか、知らなかった

Claude Code を毎日使っています。でも「どれくらい使っているか」と聞かれても、体感でしか答えられませんでした。

数えてみたら、こうでした。

```console
$ git log --format='%b' | grep -c 'Co-Authored-By: Claude'
960
```

このリポジトリ、1,107コミット中960件。**87%がClaudeとの共著**でした。

全リポジトリ横断で数えたら **26,198コミット中10,223件、39%**。この数字が思ったより大きくて、プロフィールに出したくなったので、ツールを作って公開しました。

https://github.com/yosh1/cocommit

![cocommitカード](/images/cocommit/card-dark.png)

## 既存ツールは「会話量」を数えていた

同じことを考えた人がいるはずだと思って探すと、それらしいものが複数ありました。`VibeTrace`、`cc-stats`、`agentcard`、`agent-wrapped`。星は一桁台のものが多く、決定版はまだない領域です。

ただ、見つけた6本すべてが**同じデータソース**を読んでいました。ローカルのセッションログ（`~/.claude/**/*.jsonl`）です。

つまり測っているのは「トークン数」「セッション数」「ツール呼び出し回数」。**AIとどれだけ会話したか**であって、**何を作ったか**ではない。

`Co-authored-by` トレーラの付いた実際のコミットを数えているものは、1本もありませんでした。

![比較図](/images/cocommit/compare.png)

この差は地味に効きます。「24億トークン使った」は誰にも検証できない自己申告で、PCを変えたら消えます（あるツールがわざわざ「クロスデバイス同期」を機能として売りにしているのが、その裏返しです）。

一方 `Co-authored-by` は **git の履歴に刻まれた事実**です。誰でも `git log` で確認できるし、GitHub のサーバーに残るのでマシンを変えても消えません。

正直に言うと、**向こうの数字のほうが派手**です。「24億トークン・$24,006相当」は「10,223コミット」よりインパクトがある。でも前者は検証できず、後者は git が証明してくれる。そこを取りました。

## 実装中にぶつかった3つの壁

### 1. `co-authored-by:` は公式ドキュメントに載っていない

GitHub の Search Commits API で、こう叩けます。

```console
$ gh api '/search/commits?q=author:yosh1+co-authored-by:claude&per_page=1' --jq '.total_count'
10223
```

動きます。が、**公式ドキュメントのどこにも書かれていません**。

本当に機能しているのか確かめるため、対照実験をしました。

| クエリ | total_count |
|---|---|
| `co-authored-by:noreply@anthropic.com` | 10,203 |
| `co-authored-by:copilot@github.com` | **29** |
| `zzznotaqualifier:noreply@anthropic.com`（存在しない修飾子） | **0** |
| 修飾子なし | 26,184 |

存在しない修飾子は 0 を返す。値を変えれば結果も変わる。**パーサに認識されている**ことは確かです。

ただし未記載ということは、無予告で壊れうるということ。そこで、壊れ方を2通り想定して検知するようにしました。

```js
// 共著コミットは全コミットの部分集合。総数と一致したら
// 修飾子が効いていない（クエリ全体がただの全文検索になった）
const unusable = (n) => n === 0 || n >= total;
```

「0件になる」だけでなく「**総数と同じ値が返る**」ケースを潰すのが重要でした。後者を見逃すと、フィルタが効いていないのに「共著率100%！」と誇らしげに表示してしまいます。検知したら `"Co-Authored-By: Claude"` の全文検索にフォールバックします。

### 2. private が数えられないと、数字が意味を失う

当初は github-readme-stats のような「URLを叩いたらSVGが返るサービス」にするつもりでした。が、自分のデータを見て方針を変えました。

```console
$ gh api '/search/commits?q=author:yosh1' --jq '.total_count'
26206   # private含む

$ gh api '/search/commits?q=author:yosh1+is:public' --jq '.total_count'
5687    # public のみ
```

共著コミットで見ると、この差はもっと極端です。

| 範囲 | 共著 | 全体 | 比率 |
|---|---|---|---|
| private含む | 10,223 | 26,198 | **39%** |
| public のみ | 124 | 5,687 | **2%** |

**仕事のコードはたいてい private にあります。** public 限定で集計すると、自分の場合 2% という、実態とかけ離れた数字になってしまう。

ここで Search Commits API の性質が効いてきます。他人の public コミットは検索できます（試しに `author:anuraghazra` を叩くと 49,512 件返ってきます）が、**他人の private は当然見えません**。

つまりホスト型サービスを作っても、提供できるのは「public限定の数字」だけ。自分にとって一番出したい数字が出せない。

なので **GitHub Actions 方式**にしました。各自が自分のリポジトリでワークフローを回し、自分のトークンで集計し、SVGを自分のリポジトリにコミットする。

- **private も数えられる**（これが本命）
- サーバー不要・運用コストゼロ
- 他人のPATを預からなくていい

「サービスとして公開する」より地味ですが、この用途ではこちらが正解でした。

### 3. GitHub の画像プロキシ（camo）の制約

README に貼った画像は、すべて `camo.githubusercontent.com` を経由します。実際にレスポンスヘッダを見ると、こうなっていました。

```
content-security-policy: default-src 'none'; img-src data:; style-src 'unsafe-inline'
```

この1行から制約が全部決まります。

| | 可否 |
|---|---|
| JavaScript | **不可**（`default-src 'none'`） |
| 外部フォント | **不可**（Webフォント読み込みはブロック） |
| 外部画像 | **不可**（`img-src data:` のみ） |
| CSSアニメーション | **動く**（`style-src 'unsafe-inline'`） |

アニメーションは使えるとわかったので棒グラフを伸びる演出にしたのですが、ここで一度ハマりました。

最初、こう書いていました。

```xml
<!-- 高さ0で出力して、animateで伸ばす -->
<rect height="0"><animate attributeName="height" to="28"/></rect>
```

これをローカルで画像化すると、**棒が消えます**。SMILを解釈しないレンダラだと `height="0"` のままだからです。GitHub のOGP画像生成やソーシャル共有時のサムネイルでも同じことが起きます。

最終形を属性に書き、アニメーションは `from` だけを与える形に直しました。

```xml
<!-- 最終形を属性に持たせ、animateはfromだけ与える -->
<rect height="28"><animate attributeName="height" from="0" to="28"/></rect>
```

これならアニメーションが無視されても完成形が出ます。テストにも入れてあります。

```js
test('animated rendering still declares the finished geometry', () => {
  const svg = renderCard(sample, { animate: true });
  assert.doesNotMatch(svg, /height="0"/);
});
```

## 使い方

依存パッケージゼロなので、インストールせずそのまま試せます。

```console
$ GITHUB_TOKEN=ghp_... npx github:yosh1/cocommit --user your-login
```

`repo` スコープのPATが必要です。private を数えないと、多くの人は実態とかけ離れた数字になります。

プロフィールに出す場合は、ユーザー名と同名のリポジトリに[ワークフロー](https://github.com/yosh1/cocommit/blob/main/.github/workflows/cocommit.yml)を置いて、`COCOMMIT_TOKEN` シークレットにPATを入れるだけです。あとは日次で更新されます。

```markdown
![Commits co-authored with Claude](./cocommit.svg)
```

実際に自分のプロフィールに貼ってあるので、動いているものはこちらで見られます → [github.com/yosh1](https://github.com/yosh1)

テーマは3種類あります。

![ライトテーマ](/images/cocommit/card-light.png)

![Claudeテーマ](/images/cocommit/card-claude.png)

## 外に出る情報について

このツールが出すのは**数だけ**です。

リポジトリ名・コミットメッセージ・差分は一切読みません。API には `total_count` だけを聞いていて、コミットを1件も列挙しません（そもそも Search API は1000件で列挙が打ち切られるので、列挙する設計だと成立しなかった）。

private を数えても、カードに出るのは「10,223 / 26,198・39%」と月別の本数だけです。

## 数字が語ってくれたこと

12ヶ月のグラフを見ると、2025年10月は42件だったのが、2026年9月は2,022件。**48倍**です。

しかも直近月は共著率が80%を超えている。自分の開発のやり方が1年で別物になったことが、体感ではなく数字で出てきました。

作ってよかったのは、ツールそのものより**この事実が見えたこと**かもしれません。

---

使ってみて気に入ったら、リポジトリに ⭐️ をいただけると励みになります。

https://github.com/yosh1/cocommit

他のAIエージェントのトレーラ（Cursor、Devin など）も、`AGENTS` に数行足すだけで対応できる作りにしてあります。「このエージェントも数えたい」があれば Issue か PR をください。
