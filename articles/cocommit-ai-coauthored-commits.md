---

title: "GitHubプロフィールにAIとの開発量を貼れるツールを作ってみた"
emoji: "🤝"
type: "tech"
topics: ["ClaudeCode", "GitHub", "GitHubActions", "AI", "OSS"]
published: true

---

## 自分が実際どれだけAIと書いているか、知らなかった

Claude Code を毎日使っています。でも「どれくらい使っているか」と聞かれても、体感でしか答えられませんでした。

GitHub のプロフィールには、コミット数もスター数も草も出ます。でも「そのコードを誰と書いたか」は、どこにも出ていない。

数えてみたら、こうでした。

```console
$ git log --format='%b' | grep -c 'Co-Authored-By: Claude'
960
```

このリポジトリ、1,107コミット中960件。**87%がClaudeと一緒に書いたコミット**でした。

全リポジトリ横断で数えたら **26,198コミット中10,223件、39%**。この数字が思ったより大きくて、プロフィールに出したくなったので、ツールを作って公開しました。

https://github.com/yosh1/cocommit

![cocommitカード](/images/cocommit/card-dark.png)

## 既存ツールは「会話量」を数えていた

同じことを考えた人がいるはずだと思って探すと、AI利用状況をカードにするツールはいくつも見つかりました。よくできたものが多いです。

ただ、見つけた範囲ではどれも**同じデータソース**を読んでいました。ローカルのセッションログ（`~/.claude/**/*.jsonl`）です。

そこから出るのは「トークン数」「セッション数」「ツール呼び出し回数」。**AIとどれだけ会話したか**を測る数字です。

自分が知りたかったのはそこではなく、**結果としてリポジトリに何が残ったか**でした。`Co-authored-by` トレーラの付いたコミットを数えているものは見つからなかったので、作ることにしました。

![比較図](/images/cocommit/compare.png)

この違いは、数字の性質に効いてきます。

ローカルログから出る数字は、自分のマシンの中だけにあります。PCを変えれば消えるし、他人が確かめる方法もない（クロスデバイス同期を機能として持つツールがあるのは、この弱点への対処だと思います）。

一方 `Co-authored-by` は **git の履歴に刻まれた事実**です。誰でも `git log` で確認できるし、GitHub のサーバーに残るのでマシンを変えても消えません。

トークン数のほうが、数字としては派手に出ます。「N億トークン」は「1万コミット」より大きく見える。でも自分は、検証できるほうの数字をプロフィールに置きたかった。それだけの違いです。

## 実装中にぶつかった3つの壁

### 1. `co-authored-by:` は公式ドキュメントに載っていない

GitHub の Search Commits API で、こう叩けます。

```console
$ gh api '/search/commits?q=author:yosh1+co-authored-by:noreply@anthropic.com&per_page=1' --jq '.total_count'
10228
```

動きます。が、**公式ドキュメントのどこにも書かれていません**。

本当に機能しているのか確かめるため、対照実験をしました。

| クエリ | total_count |
|---|---|
| `co-authored-by:noreply@anthropic.com` | 10,228 |
| `co-authored-by:copilot@github.com` | **29** |
| `zzznotaqualifier:noreply@anthropic.com`（存在しない修飾子） | **0** |
| 修飾子なし | 26,206 |

存在しない修飾子は 0 を返す。値を変えれば結果も変わる。**パーサに認識されている**ことは確かです。

ただし未記載ということは、無予告で壊れうるということ。そこで、壊れ方を2通り想定して検知するようにしました。

```js
// 共著コミットは全コミットの部分集合。総数と一致したら
// 修飾子が効いていない（クエリ全体がただの全文検索になった）
const unusable = (n) => n === 0 || n >= total;
```

「0件になる」だけでなく「**総数と同じ値が返る**」ケースを潰すのが重要でした。後者を見逃すと、フィルタが効いていないのに「AI率100%！」と誇らしげに表示してしまいます。検知したら `"Co-Authored-By: Claude"` の全文検索にフォールバックします。

### 2. private が数えられないと、数字が意味を失う

当初は github-readme-stats のような「URLを叩いたらSVGが返るサービス」にするつもりでした。が、自分のデータを見て方針を変えました。

```console
$ gh api '/search/commits?q=author:yosh1' --jq '.total_count'
26206   # private含む

$ gh api '/search/commits?q=author:yosh1+is:public' --jq '.total_count'
5687    # public のみ
```

AIと書いたコミットで見ると、この差はもっと極端です。

| 範囲 | AIと書いた | 全体 | 比率 |
|---|---|---|---|
| private含む | 10,223 | 26,198 | **39%** |
| public のみ | 124 | 5,687 | **2%** |

**仕事のコードはたいてい private にあります。** public 限定で集計すると、自分の場合 2% という、実態とかけ離れた数字になってしまう。

ここで Search Commits API の性質が効いてきます。他人の public コミットは検索できますが、**他人の private は当然見えません**。

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

しかも直近月は80%を超えている。自分の開発のやり方が1年で別物になったことが、体感ではなく数字で出てきました。

作ってよかったのは、ツールそのものより**この事実が見えたこと**かもしれません。

---

使ってみて気に入ったら、リポジトリに ⭐️ をいただけると励みになります。

https://github.com/yosh1/cocommit

`--agent` で9種類に対応しています。Claude / Copilot / Cursor / Codex / Devin / Gemini / Jules / Aider / Amp。

```console
$ npx github:yosh1/cocommit --agent cursor --user your-login
```

追加するときに一点だけ気をつけたことがあります。**エージェントは名前ではなくメールアドレスで引く**ようにしました。名前で引くと同名の他人を巻き込むからです。実際 `co-authored-by:claude` で引くと、Claude とは無関係な人間の共著者が混ざったコミットが出てきました。

なので9件とも、実際の公開コミットからトレーラを読んで確認しています。

```
Co-authored-by: Cursor <cursoragent@cursor.com>
Co-authored-by: aider (anthropic/claude-sonnet-5) <aider@aider.chat>
```

Aider のように「使ったモデル名が表示名に入る」ものもあるので、アドレスで引く判断は結果的に正解でした。

まだ入っていないエージェントがあれば、こちらで自分のトレーラを調べて Issue をもらえれば追加します。

```console
$ git log --format='%b' | grep -i co-authored | sort -u
```
