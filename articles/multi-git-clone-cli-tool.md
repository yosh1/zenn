---

title: "同じリポジトリを複数フォルダに一発クローン。CLIツール「mgc」を作った"
emoji: ":file_folder:"
type: "tech"
topics: ["CLI", "Git", "TypeScript", "ClaudeCode", "npm"]
published: true

---

## きっかけ：Claude Codeで並列作業がしたい

Claude Codeを使っていると、同じリポジトリを複数フォルダにクローンして並列で作業したくなる場面が増えてきました。

たとえば、ある機能をブランチAで開発しながら、別のブランチBでバグ修正。あるいは、Claude Codeのセッションをフォルダごとに分けて同時に走らせる。git worktreeという手もありますが、完全に独立したクローンのほうが都合がいいことも多いです。

ただ、毎回 `git clone` を手打ちして、フォルダ名に番号を振って、既存のフォルダと被らないか確認して……という作業は地味に面倒です。

そこで **`mgc`（multi-git-clone）** というCLIツールを作りました。

## mgcでできること

```bash
mgc vercel/next.js 3
```

これだけで、以下の3フォルダに並列クローンされます。

```
~/git/github.com/vercel/next.js
~/git/github.com/vercel/next.js-2
~/git/github.com/vercel/next.js-3
```

既にフォルダが存在する場合は、次の番号から自動採番してくれます。

```bash
# next.js と next.js-2 がすでにある状態で：
mgc vercel/next.js 3
# → next.js-3, next.js-4, next.js-5 にクローン
```

## インストール

```bash
npm install -g multi-git-clone
```

npxでも使えます。

```bash
npx multi-git-clone vercel/next.js 3
```

## 使い方

基本の形はシンプルです。

```bash
mgc <リポジトリ> [コピー数]
```

リポジトリの指定は、いくつかの形式に対応しています。

```bash
mgc vercel/next.js                              # org/repo 形式
mgc https://github.com/vercel/next.js            # HTTPS URL
mgc git@github.com:vercel/next.js.git            # SSH
```

### オプション

| オプション | 内容 | デフォルト |
|-----------|------|-----------|
| `--base <path>` | クローン先のベースディレクトリ | `~/git/github.com` |
| `--sep <char>` | 番号の区切り文字（`-` `_` `.`） | `-` |
| `--flat` | org サブディレクトリを省略 | `false` |
| `--dry-run` | 実行せずパスだけ表示 | `false` |

### dry-runで事前確認

実行前にどこにクローンされるか確認したいときは `--dry-run` が便利です。

```bash
$ mgc vercel/next.js 3 --dry-run

Clone paths (dry run):

  1. ~/git/github.com/vercel/next.js
  2. ~/git/github.com/vercel/next.js-2
  3. ~/git/github.com/vercel/next.js-3
```

### 区切り文字のカスタマイズ

ハイフン以外がいい場合は `--sep` で変更できます。

```bash
mgc vercel/next.js 3 --sep _
# → next.js, next.js_2, next.js_3

mgc vercel/next.js 3 --sep .
# → next.js, next.js.2, next.js.3
```

### フラットな構造

org ディレクトリを作りたくない場合は `--flat` を指定します。

```bash
mgc vercel/next.js 3 --flat
# → ~/git/github.com/next.js
# → ~/git/github.com/next.js-2
# → ~/git/github.com/next.js-3
```

## 設定ファイル

毎回オプションを指定するのが面倒なら、`~/.mgcrc` にデフォルト値を書いておけます。

```json
{
  "basePath": "~/projects",
  "useOrgDirectory": true,
  "separator": "-"
}
```

CLI引数で渡した値が優先されるので、普段のデフォルトだけ決めておいて、必要なときだけ上書きする運用ができます。

## Claude Codeとの組み合わせ

自分がmgcを一番使うのは、Claude Codeとの組み合わせです。

大きめの機能開発を3つに分けて、それぞれ別フォルダでClaude Codeを走らせる、みたいな使い方をしています。

```bash
# 3つのワークスペースを一発で用意
mgc myorg/my-app 3

# それぞれのフォルダでClaude Codeを起動
cd ~/git/github.com/myorg/my-app && claude
cd ~/git/github.com/myorg/my-app-2 && claude
cd ~/git/github.com/myorg/my-app-3 && claude
```

フォルダが丸ごと別なので、ファイルの競合を気にしなくていい。終わったらブランチをpushしてマージ、フォルダは消すだけです。

## 技術的なポイント

### 外部依存ゼロ

`commander` や `yargs` のようなCLI引数パーサーは使っていません。Node.js 18.3以降で標準搭載された `util.parseArgs` だけで実装しています。依存パッケージは TypeScript のビルドに使う `typescript` と `@types/node` のみ。実行時の依存はゼロです。

### 並列クローン

複数のクローンは `Promise.all` で同時に走らせています。3つクローンしても、かかる時間はほぼ1つ分。一部が失敗しても他には影響しません。

### 自動採番の仕組み

`fs.readdir` でディレクトリを一度だけ走査して、使われている番号を `Set` に集めます。そこから空いているスロットを順に割り当てるだけ。同じバッチ内で番号が被ることもありません。

## まとめ

やっていることは単純で、`git clone` を複数回叩いて番号を振るだけ。ただ、これを毎回手でやるのは地味にだるかったので、CLIにして正解でした。

Claude Codeに限らず、並列で作業するときにサッと使えるので、よかったら試してみてください。

```bash
npm install -g multi-git-clone
```

GitHub: [multi-git-clone](https://github.com/yosh1/multi-git-clone)
