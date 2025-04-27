---
title: "Google Groups 宛メールに Cloudflare Workers + SendGrid で自動返信する"
emoji: "📧"
type: "tech"
topics: ["Cloudflare", "SendGrid", "GoogleGroups", "自動化", "メール"]
published: true
---

## はじめに

この記事では、Google Groups に届くメールに対し、Cloudflare Workers と SendGrid を利用して自動で返信するシステムを構築する方法を解説します。

具体的には、Google Groups のメール転送機能を利用し、Cloudflare Email Routing 経由で Worker にメールを渡し、SendGrid API を使って自動返信メールを送信します。

## システム概要

1. **Google Groups:** 受信したメールを指定した転送用アドレス（Cloudflare Email Routing で作成）に転送します。
2. **Cloudflare Email Routing:** 転送されてきたメールを捕捉し、指定された Cloudflare Worker にルーティングします。
3. **Cloudflare Worker:**
   * `mailparser` ライブラリで転送されてきたメールを解析し、元の送信者情報（メールアドレス、名前）を取得します。
   * 取得した情報に基づき、SendGrid を利用して元の送信者へ自動返信メールを作成・送信します。
4. **SendGrid:** Worker からの指示に基づき、実際にメールを送信します。

![システム構成図のイメージ（テキストでの代替）](フロー図： Google Groups -> 転送 -> Cloudflare Email Routing (カスタムアドレス) -> Cloudflare Worker (メール解析 -> SendGridで自動返信) -> 元の送信者 )

## 前提条件

* Cloudflare アカウント（独自ドメイン設定済み）
* SendGrid アカウントと API キー
* Node.js と npm (または pnpm, yarn) がインストールされた開発環境
* 自動返信を設定したい Google Groups の管理権限

## プロジェクトのセットアップ

### 1. プロジェクトの初期化と依存パッケージのインストール

```bash
# プロジェクトディレクトリを作成し移動
mkdir gg-auto-reply-worker
cd gg-auto-reply-worker

# npm プロジェクトを初期化
npm init -y

# Wrangler (Cloudflare Workers CLI) をインストール
npm install --save-dev wrangler

# 必要なライブラリをインストール
npm install @sendgrid/mail mailparser
```

### 2. 設定ファイルの作成

#### `package.json` の確認

以下のような構成になっていることを確認します。

```json
{
  "name": "gg-auto-reply-worker",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "deploy": "wrangler deploy",
    "dev": "wrangler dev",
    "start": "wrangler dev"
  },
  "devDependencies": {
    "wrangler": "^3.0.0" # 最新バージョンを確認・指定推奨
  },
  "dependencies": {
    "@sendgrid/mail": "^8.0.0", # 最新バージョンを確認・指定推奨
    "mailparser": "^3.0.0"   # 最新バージョンを確認・指定推奨
  }
}
```

#### `wrangler.toml` の作成

プロジェクトルートに `wrangler.toml` を作成します。

```toml
name = "gg-auto-reply-worker" # Worker の名前
main = "src/worker.js"       # エントリーポイントのファイルパス
compatibility_date = "YYYY-MM-DD" # デプロイ日の日付などを指定 (例: "2025-04-27")
compatibility_flags = ["nodejs_compat"] # Node.js 互換モードを有効化

# [vars] セクションは使用せず、Secrets を使用します

# (オプション) ローカル開発時の設定
[dev]
ip = "localhost"
port = 8787
```

* **`compatibility_date`:** デプロイする日の日付を指定してください。

### 3. 機密情報の設定 (Secrets)

SendGrid API キーと、自動返信メールの送信元として表示されるメールアドレスを、`wrangler secret put` コマンドで Cloudflare に安全に登録します。

```bash
# SendGrid API キーを登録 (YOUR_SENDGRID_API_KEY を実際のキーに置き換える)
npx wrangler secret put SENDGRID_API_KEY

# 自動返信メールの送信元アドレスを登録 (例: no-reply@yourdomain.com など)
# このアドレスは SendGrid で認証済みである必要があります
npx wrangler secret put FROM_EMAIL
```

コマンド実行後、プロンプトに従って実際の値を入力します。

## Cloudflare Email Routing と Google Groups の設定

### 1. Cloudflare Email Routing で転送用アドレスを作成

まず、Google Groups からのメールを受け取るための専用メールアドレスを Cloudflare で作成します。

1. Cloudflare ダッシュボードにログインし、対象のドメインを選択します。
2. 左側のメニューから「メール」>「Email Routing」を選択します。
3. 「ルート」タブに移動し、「カスタムアドレス」セクションで「カスタムアドレスを作成」をクリックします。
4. **カスタムアドレス:** Google Groups の転送先に設定するメールアドレスを入力します。例えば、`group-reply@yourdomain.com` のように、この目的専用のアドレスを作成するのがおすすめです。
5. **アクション:** 「Worker に送信」を選択し、`wrangler.toml` で設定した Worker 名（例: `gg-auto-reply-worker`）を指定します。
6. 「保存」をクリックしてアドレスを有効化します。

### 2. Google Groups の転送設定

次に、自動返信を設定したい Google Groups の設定画面で、上記で作成した Cloudflare のカスタムアドレスにメールを転送するように設定します。

1. Google Groups (groups.google.com) にアクセスし、対象のグループを選択します。
2. 左メニューから「グループ設定」>「メール配信オプション」に進みます。
3. （詳細設定が表示されていない場合）設定項目を探し、「すべてのメールを転送するアドレス」のような項目を見つけます。（※Google Groups の UI は変更される可能性があるため、正確な項目名は適宜確認してください）
4. Cloudflare Email Routing で作成したカスタムアドレス（例: `group-reply@yourdomain.com`）を入力し、設定を保存します。

**注意:** Google Groups の転送設定では、多くの場合、転送先アドレスの所有権確認が必要です。Cloudflare で作成したアドレス宛に確認メールが送信されるため、一時的にそのアドレス宛のメールを別の受信可能なメールアドレスに転送するルートを設定するか、Worker で確認メールの内容をログ出力するなどして、確認コードを取得・入力する必要があります。所有権確認が完了したら、Worker へのルーティング設定に戻してください。

これで、Google Groups 宛のメールが Cloudflare Worker に転送されるようになりました。

## Worker スクリプト (`src/worker.js`)

`src` ディレクトリを作成し、その中に `worker.js` ファイルを作成して以下のコードを記述します。

```javascript
// src/worker.js
import { simpleParser } from 'mailparser';
import sgMail from '@sendgrid/mail';

export default {
  async email(message, env, ctx) {
    // --- 設定値のチェック ---
    const sendGridApiKey = env.SENDGRID_API_KEY;
    const fromEmail = env.FROM_EMAIL;

    if (!sendGridApiKey) {
      console.error('SendGrid APIキー (SENDGRID_API_KEY) が設定されていません。');
      message.reject('Configuration error: SendGrid API key missing.');
      return;
    }
    sgMail.setApiKey(sendGridApiKey); // APIキーを設定

    if (!fromEmail) {
      console.error('送信元メールアドレス (FROM_EMAIL) が設定されていません。');
      message.reject('Configuration error: Sender email address missing.');
      return;
    }

    try {
      // --- メール解析 ---
      // message.raw (ReadableStream) を mailparser に渡す
      const parsed = await simpleParser(message.raw);

      // 元の送信者情報を取得 (Google Groups転送メールの 'From' ヘッダから)
      let originalSenderEmail = null;
      let originalSenderName = null;
      if (parsed.from?.value?.length > 0) {
        originalSenderEmail = parsed.from.value[0].address;
        originalSenderName = parsed.from.value[0].name || originalSenderEmail.split('@')[0];
      }

      // 元の件名
      const originalSubject = parsed.subject || '(件名なし)';

      // --- 自動返信ループ防止 ---
      // 1. 元の送信者が自分自身(FROM_EMAIL) でないかチェック
      if (originalSenderEmail && originalSenderEmail.toLowerCase() === fromEmail.toLowerCase()) {
        console.log(`送信者がFROM_EMAIL (${fromEmail}) と一致。自動返信をスキップ。`);
        return; // ループ防止のため終了
      }
      // 2. Precedence ヘッダーをチェック (bulk, list, junk は返信しない)
      const precedence = message.headers?.get('precedence')?.toLowerCase();
      if (precedence === 'bulk' || precedence === 'list' || precedence === 'junk') {
         console.log(`Precedenceヘッダー (${precedence}) により自動返信をスキップ。`);
         return;
      }
      // 3. Auto-Submitted ヘッダーをチェック (自動生成メールには返信しない)
      const autoSubmitted = message.headers?.get('auto-submitted')?.toLowerCase();
      if (autoSubmitted && autoSubmitted !== 'no') {
          console.log(`Auto-Submittedヘッダー (${autoSubmitted}) により自動返信をスキップ。`);
          return;
      }

      // --- 自動返信メールの送信 ---
      if (originalSenderEmail) {
        const replySubject = `Re: ${originalSubject}`;
        const replyBody = `${originalSenderName} 様\n\nお問い合わせありがとうございます。\n内容を確認の上、担当者より改めてご連絡いたします。\n\n(受信日時: ${new Date().toLocaleString('ja-JP')})\n\n--\n[あなたの組織名やサービス名]\n\n※このメールは自動送信されました。`;

        const msg = {
          to: originalSenderEmail, // 元の送信者に返信
          from: {
             email: fromEmail, // 設定した送信元アドレス
             name: "サポート窓口" // 送信者名 (適宜変更)
          },
          subject: replySubject,
          text: replyBody,
          // 自動返信を示すヘッダーを付与 (他のシステムでのループ防止)
          headers: {
            'Precedence': 'bulk',
            'Auto-Submitted': 'auto-generated',
          },
        };

        try {
          const response = await sgMail.send(msg);
          console.log(`自動返信メール送信成功: To=${originalSenderEmail}, Status=${response[0].statusCode}`);
        } catch (sendError) {
          console.error(`SendGrid送信エラー: To=${originalSenderEmail}`, sendError.response?.body || sendError.message);
          // SendGridのエラー詳細 (必要に応じてログを強化)
          // if (sendError.response) { console.error(JSON.stringify(sendError.response.body)); }
          // 送信試行はしたので、ここでは message.reject() しない
        }
      } else {
        console.warn('元の送信者メールアドレスが見つからず、自動返信できませんでした。', `Subject: ${originalSubject}`);
      }

      // 処理完了
      return;

    } catch (err) {
      console.error('メール処理中に予期せぬエラー:', err);
      // エラー発生時はメールを Reject する (送信者に配信エラーが返る可能性)
      message.reject(`Internal processing error: ${err.message}`);
    }
  },
};
```

**コードのポイント:**

* **シンプル化:** Slack通知やCCメール関連のコードを削除しました。
* **変数名の明確化:** Google Groups から転送されてきたメールの元の送信者を `originalSenderEmail`, `originalSenderName` としました。
* **ループ防止強化:** `Auto-Submitted` ヘッダーのチェックも追加しました。
* **送信者名:** `msg.from.name` で自動返信メールの送信者名を指定できます（例: "サポート窓口"）。
* **エラーハンドリング:** SendGrid 送信エラー時のログを少し具体的にしました。致命的な設定不備や予期せぬエラーの場合は `message.reject()` を呼び出します。

## デプロイとテスト

### 1. ローカルでのテスト (限定的)

`wrangler dev` で Worker の起動と構文チェックが可能です。

```bash
npx wrangler dev
```

### 2. Cloudflare へのデプロイ

以下のコマンドで Worker を Cloudflare にデプロイします。

```bash
npx wrangler deploy
```

デプロイ後、設定した Google Groups のアドレス宛に、**グループメンバー以外** の外部メールアドレスからテストメールを送信します。元の送信者に自動返信メールが届けば成功です。

**テスト時の注意点:**

* **送信元:** Google Groups のメンバーがグループ宛に送信した場合、通常そのメールは転送されません（グループの設定による）。必ずグループ外のアドレスからテストしてください。
* **ループ防止:** 自分自身のアドレス（`FROM_EMAIL` に設定したもの）からテストメールを送ると、ループ防止機能により自動返信はされません。
* **確認:** 自動返信が届かない場合は、まず Cloudflare ダッシュボードの「メール」>「ログ」と、Worker のログ (`wrangler tail` またはダッシュボード）を確認してください。

## トラブルシューティング

* **自動返信が届かない:**
    * Cloudflare Email Routing のログを確認し、メールが Worker に転送されているかチェック。
    * Worker のログを確認し、API キー設定エラー、SendGrid 送信エラー、ループ防止によるスキップ、その他のコードエラーがないかチェック。
    * SendGrid の Activity Feed を確認し、メールが送信されているか、ブロックやバウンスが発生していないかチェック。送信元アドレスの認証（Domain Authentication）が必須です。
    * Google Groups の転送設定と Cloudflare のカスタムアドレスが正しく連携されているか（所有権確認含む）再確認。
* **文字化け:** `simpleParser` が対応していない稀なエンコーディングの可能性があります。受信メールのヘッダーを確認してください。
* **意図しないメールにも返信する:** ループ防止の条件（`FROM_EMAIL` チェック、`Precedence`/`Auto-Submitted` ヘッダーチェック）を見直してください。

## まとめ

Google Groups の転送機能と Cloudflare Workers / Email Routing / SendGrid を組み合わせることで、既存システムを保ったままサーバーレスで効率的なメール自動返信システムを構築できます。`mailparser` による簡単な解析、Secrets による安全なキー管理、そして重要な自動返信ループ防止策を実装することがポイントです。

## 参考リソース

* **Cloudflare Workers:** [https://developers.cloudflare.com/workers/](https://developers.cloudflare.com/workers/)
* **Cloudflare Email Routing:** [https://developers.cloudflare.com/email-routing/](https://developers.cloudflare.com/email-routing/)
* **Wrangler CLI:** [https://developers.cloudflare.com/workers/wrangler/](https://developers.cloudflare.com/workers/wrangler/)
* **SendGrid API:** [https://docs.sendgrid.com/api-reference/](https://docs.sendgrid.com/api-reference/)
* **mailparser:** [https://nodemailer.com/extras/mailparser/](https://nodemailer.com/extras/mailparser/)
