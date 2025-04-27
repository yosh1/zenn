---
title: "Google Groups 宛メールに Cloudflare Workers + SendGrid で自動返信する"
emoji: "📧"
type: "tech"
topics: ["Cloudflare", "SendGrid", "GoogleGroups", "自動化", "メール"]
published: true
---

## はじめに

先日、社内のGoogle Groupsに届く問い合わせメールに対して、手動での返信作業が増えてきたため、自動返信システムの導入を検討しました。本記事では、Cloudflare WorkersとSendGridを組み合わせて実装した自動返信システムの構築方法を解説します。

具体的な実装では、以下のような流れで処理を行います。

1. Google Groupsに届いたメールをCloudflare Email Routing経由でWorkerに転送します。
2. Workerでメールを解析し、元の送信者情報を取得します。
3. SendGrid APIを使用して自動返信メールを送信します。

このシステムにより、問い合わせメールへの即時返信が可能となり、対応の遅延を防ぐことができます。

## システム概要

システムの構成要素と役割について説明します。

1. **Google Groups**
   - 問い合わせメールの受信窓口として機能します。
   - 受信したメールをCloudflare Email Routingの転送用アドレスに自動転送します。

2. **Cloudflare Email Routing**
   - 転送されてきたメールを捕捉します。
   - 指定したCloudflare Workerにメールをルーティングします。
   - カスタムメールアドレスの管理を行います。

3. **Cloudflare Worker**
   - メールの解析処理を行います（mailparserライブラリを使用）。
   - 送信者情報を抽出します。
   - 自動返信メールを生成し、SendGrid APIを呼び出します。

4. **SendGrid**
   - 自動返信メールの実際の送信処理を行います。
   - メール配信の信頼性を確保します。
   - 配信状況を監視します。

![システム構成図のイメージ（テキストでの代替）](フロー図： Google Groups -> 転送 -> Cloudflare Email Routing (カスタムアドレス) -> Cloudflare Worker (メール解析 -> SendGridで自動返信) -> 元の送信者 )

## 前提条件

本システムを構築するために必要な環境とアカウントは以下の通りです：

* **Cloudflare アカウント**
  - 独自ドメインの設定が完了していること
  - Email Routing機能が有効化されていること
  - Workers機能が利用可能であること

* **SendGrid アカウント**
  - 無料プラン以上で利用可能
  - APIキーが発行済みであること
  - 送信元ドメインの認証が完了していること

* **開発環境**
  - Node.js v18以上
  - npm v8以上
  - コードエディタ（VSCode推奨）

* **Google Groups の権限**
  - 対象グループの管理者権限
  - メール転送設定の変更権限

## プロジェクトのセットアップ

### 1. プロジェクトの初期化

まず、プロジェクト用のディレクトリを作成し、必要なパッケージをインストールします：

```bash
# プロジェクトディレクトリを作成
mkdir gg-auto-reply-worker
cd gg-auto-reply-worker

# npmプロジェクトの初期化
npm init -y

# 必要なパッケージのインストール
npm install --save-dev wrangler@3.22.1
npm install @sendgrid/mail@7.7.0 mailparser@3.6.5
```

### 2. 設定ファイルの作成

#### `package.json` の設定

プロジェクトの設定ファイルを以下のように編集します：

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
    "wrangler": "^3.22.1"
  },
  "dependencies": {
    "@sendgrid/mail": "^7.7.0",
    "mailparser": "^3.6.5"
  }
}
```

#### `wrangler.toml` の設定

Cloudflare Workersの設定ファイルを作成します：

```toml
name = "gg-auto-reply-worker"
main = "src/worker.js"
compatibility_date = "2024-04-27"
compatibility_flags = ["nodejs_compat"]

[dev]
ip = "localhost"
port = 8787
```

### 3. 機密情報の設定

セキュリティを考慮し、機密情報はCloudflareのSecrets機能を使用して管理します：

```bash
# SendGrid APIキーの設定
npx wrangler secret put SENDGRID_API_KEY
# プロンプトが表示されたら、SendGridのAPIキーを入力

# 送信元メールアドレスの設定
npx wrangler secret put FROM_EMAIL
# プロンプトが表示されたら、SendGridで認証済みのメールアドレスを入力
```

## Cloudflare Email RoutingとGoogle Groupsの設定

### 1. Cloudflare Email Routingの設定

まず、Google Groupsからのメールを受け取るための専用アドレスを設定します：

1. **Cloudflareダッシュボードでの設定**
   - Cloudflareダッシュボードにログイン
   - 対象ドメインを選択
   - 左メニューから「メール」→「Email Routing」を選択
   - 「ルート」タブを選択

2. **カスタムアドレスの作成**
   - 「カスタムアドレスを作成」ボタンをクリック
   - 以下の情報を入力：
     - アドレス: `group-reply@yourdomain.com`（例）
     - アクション: 「Workerに送信」を選択
     - Worker名: `gg-auto-reply-worker`（wrangler.tomlで設定した名前）
   - 「保存」をクリック

3. **設定の確認**
   - 作成したカスタムアドレスが「アクティブ」状態になっていることを確認
   - テストメールを送信して、Workerに正しく転送されることを確認

### 2. Google Groupsの設定

次に、Google Groupsのメール転送設定を行います：

1. **Google Groupsの設定画面を開く**
   - groups.google.comにアクセス
   - 対象のグループを選択
   - 左メニューから「グループ設定」を選択

2. **メール転送の設定**
   - 「メール配信オプション」セクションを探す
   - 「すべてのメールを転送するアドレス」の設定を探す
   - Cloudflareで作成したカスタムアドレスを入力
     - 例: `group-reply@yourdomain.com`

3. **所有権確認の対応**
   - Google Groupsから確認メールが送信される
   - 確認メールを受け取るために、一時的に以下のいずれかの対応が必要：
     - Cloudflare Email Routingで確認メール用の別ルートを設定
     - Workerのログ出力機能を使用して確認コードを取得
   - 確認コードを入力して所有権を確認
   - 所有権確認後、必要に応じて一時的な設定を元に戻す

4. **動作確認**
   - グループ外のメールアドレスからテストメールを送信
   - 自動返信メールが正しく送信されることを確認
   - メールの転送と自動返信が正常に機能していることを確認

## Workerスクリプトの実装

### 1. プロジェクト構造の作成

まず、必要なディレクトリとファイルを作成します：

```bash
mkdir src
touch src/worker.js
```

### 2. Workerスクリプトの実装

`src/worker.js`に以下のコードを実装します：

```javascript
// src/worker.js
import { simpleParser } from 'mailparser';
import sgMail from '@sendgrid/mail';

export default {
  async email(message, env, ctx) {
    // 環境変数の取得と検証
    const sendGridApiKey = env.SENDGRID_API_KEY;
    const fromEmail = env.FROM_EMAIL;

    if (!sendGridApiKey) {
      console.error('SendGrid APIキーが設定されていません');
      message.reject('Configuration error: SendGrid API key missing');
      return;
    }
    sgMail.setApiKey(sendGridApiKey);

    if (!fromEmail) {
      console.error('送信元メールアドレスが設定されていません');
      message.reject('Configuration error: Sender email address missing');
      return;
    }

    try {
      // メールの解析
      const parsed = await simpleParser(message.raw);

      // 送信者情報の取得
      const originalSender = parsed.from?.value?.[0];
      if (!originalSender) {
        console.warn('送信者情報が見つかりません');
        return;
      }

      const originalSenderEmail = originalSender.address;
      const originalSenderName = originalSender.name || originalSenderEmail.split('@')[0];
      const originalSubject = parsed.subject || '(件名なし)';

      // 自動返信ループ防止
      if (originalSenderEmail.toLowerCase() === fromEmail.toLowerCase()) {
        console.log(`送信者がFROM_EMAILと一致するため、自動返信をスキップ: ${fromEmail}`);
        return;
      }

      const precedence = message.headers?.get('precedence')?.toLowerCase();
      if (precedence === 'bulk' || precedence === 'list' || precedence === 'junk') {
        console.log(`Precedenceヘッダーにより自動返信をスキップ: ${precedence}`);
        return;
      }

      const autoSubmitted = message.headers?.get('auto-submitted')?.toLowerCase();
      if (autoSubmitted && autoSubmitted !== 'no') {
        console.log(`Auto-Submittedヘッダーにより自動返信をスキップ: ${autoSubmitted}`);
        return;
      }

      // 自動返信メールの作成と送信
      const replySubject = `Re: ${originalSubject}`;
      const replyBody = `${originalSenderName} 様

お問い合わせありがとうございます。
内容を確認の上、担当者より改めてご連絡いたします。

受信日時: ${new Date().toLocaleString('ja-JP')}

--
株式会社サンプル
カスタマーサポート

※このメールは自動送信されました。`;

      const msg = {
        to: originalSenderEmail,
        from: {
          email: fromEmail,
          name: 'カスタマーサポート'
        },
        subject: replySubject,
        text: replyBody,
        headers: {
          'Precedence': 'bulk',
          'Auto-Submitted': 'auto-generated'
        }
      };

      try {
        const response = await sgMail.send(msg);
        console.log(`自動返信メール送信成功: ${originalSenderEmail}, ステータス: ${response[0].statusCode}`);
      } catch (sendError) {
        console.error(`SendGrid送信エラー: ${originalSenderEmail}`, sendError.response?.body || sendError.message);
      }

    } catch (err) {
      console.error('メール処理中にエラーが発生:', err);
      message.reject(`Internal processing error: ${err.message}`);
    }
  }
};
```

### 3. コードの説明

実装したWorkerスクリプトの主要な機能は以下の通りです：

1. **メール解析**
   - `mailparser`ライブラリを使用してメールを解析
   - 送信者情報（メールアドレス、名前）を抽出
   - 件名を取得（ない場合は「(件名なし)」を使用）

2. **自動返信ループ防止**
   - 送信者が自己参照（FROM_EMAIL）の場合にスキップ
   - メールの種類（bulk, list, junk）をチェック
   - 自動生成メールの判定

3. **自動返信メールの送信**
   - 送信者名を含めた丁寧な返信文を作成
   - 受信日時を日本時間で表示
   - 自動返信を示すヘッダーを設定

4. **エラーハンドリング**
   - 設定値の検証
   - メール解析エラーの処理
   - SendGrid送信エラーの処理

### 4. デプロイとテスト

1. **ローカルでのテスト**
```bash
npx wrangler dev
```

2. **本番環境へのデプロイ**
```bash
npx wrangler deploy
```

3. **動作確認**
   - グループ外のメールアドレスからテストメールを送信
   - 自動返信メールが正しく送信されることを確認
   - Cloudflareダッシュボードでログを確認

## トラブルシューティング

### 1. 自動返信が届かない場合

1. **Cloudflare Email Routingの確認**
   - ダッシュボードの「メール」→「ログ」で転送状況を確認してください。
   - カスタムアドレスが「アクティブ」状態になっているか確認してください。
   - 転送ルールが正しく設定されているか確認してください。

2. **Workerのログ確認**
   ```bash
   npx wrangler tail
   ```
   - エラーメッセージを確認してください。
   - メール解析の成功/失敗を確認してください。
   - SendGrid APIの応答状況を確認してください。

3. **SendGridの確認**
   - Activity Feedでメールの送信状況を確認してください。
   - 送信元ドメインの認証状態を確認してください。
   - APIキーの権限設定を確認してください。

4. **Google Groupsの設定確認**
   - 転送設定が正しく行われているか確認してください。
   - 所有権確認が完了しているか確認してください。
   - グループのメール配信設定を確認してください。

### 2. 文字化けが発生する場合

1. **メールヘッダーの確認**
   - Content-Typeヘッダーの文字コード設定を確認してください。
   - エンコーディング方式を確認してください。

2. **Worker側の対応**
   - メール解析時の文字コード処理を確認してください。
   - 必要に応じて文字コード変換処理を追加してください。

### 3. 意図しないメールにも返信する場合

1. **ループ防止の確認**
   - FROM_EMAILとの一致チェックを確認してください。
   - Precedenceヘッダーのチェックを確認してください。
   - Auto-Submittedヘッダーのチェックを確認してください。

2. **メールフィルタリングの追加**
   - 特定のドメインからのメールを除外してください。
   - 特定のパターンのメールを除外してください。
   - スパム判定を追加してください。

## まとめ

本記事では、Google Groupsに届くメールに対して自動返信するシステムを構築する方法を解説しました。主なポイントについて説明します。

1. **システム構成**
   - Google Groupsのメール転送機能を使用します。
   - Cloudflare Email Routingでメールを捕捉します。
   - Cloudflare Workersでメール処理を行います。
   - SendGridでメール送信を実行します。

2. **実装のポイント**
   - メール解析で送信者情報を取得します。
   - 自動返信ループを防止します。
   - エラーハンドリングとログ出力を実装します。
   - セキュアな設定管理を行います。

3. **運用上の注意点**
   - 定期的にログを確認します。
   - エラー発生時は迅速に対応します。
   - 設定変更時は動作確認を行います。

このシステムにより、問い合わせメールへの即時返信が可能となり、対応の遅延を防ぐことができます。また、Cloudflare WorkersとSendGridを組み合わせることで、サーバーレスで効率的な運用が可能です。

## 参考リソース

* [Cloudflare Workers公式ドキュメント](https://developers.cloudflare.com/workers/)
* [Cloudflare Email Routing公式ドキュメント](https://developers.cloudflare.com/email-routing/)
* [SendGrid APIリファレンス](https://docs.sendgrid.com/api-reference/)
* [mailparserドキュメント](https://nodemailer.com/extras/mailparser/)
* [Google Groups管理ガイド](https://support.google.com/groups/answer/2455694)
