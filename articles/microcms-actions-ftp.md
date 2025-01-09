---
title: "microCMSのWebhookでGitHub Actionsを起動、FTPアップロードまで" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["microCMS", "GitHub Actions", "FTP", "自動化"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true
---

## 背景
普段使用しているVercelが利用できなかったため、microCMSを活用したプロジェクトにおいて、レンタルサーバーへのFTPアップロードを自動化するワークフローを構築することに挑戦しました。  
今回は、GMOのレンタルサーバー「ヘテムル」を使用し、GitHub Actionsでビルドからアップロードまでを自動化しました。  
この設定では、microCMSのWebhookによってGitHub Actionsをトリガーし、コンテンツ更新→サイトのビルド→サーバーへのデプロイというシンプルで効率的な流れを実現します。

---

## 対象読者のイメージ
- microCMSを使ったプロジェクトを効率化したいと考えている方  
- レンタルサーバー（FTP）のデプロイを簡略化したい方  
- GitHub Actionsを活用してみたい方  
- 全体のリリースフローの自動化を考えている方  

---

## 実現できること
- microCMSの記事更新をトリガーに、GitHub Actionsでの自動ビルドとFTP（レンタルサーバー）へのアップロードを行えるようになります。
- GitHubリポジトリでのビルド環境を活用しつつ、Vercelなどを使わずにレンタルサーバーでのホスティング対応が可能になります。
- CIツール（GitHub Actions）を介することで、途中に任意のスクリプトやテストを挿入する柔軟な開発フローを構築できます。

---

## ワークフローの全体概要
以下の流れで自動化を行います。

1. microCMSで記事を更新する。
2. Webhookを利用してGitHub Actionsをトリガーする。
3. GitHub Actionsがビルド処理を行い、静的ファイルを生成。
4. GitHub Actionsが生成された静的ファイルをFTPでサーバーにアップロードし、サイトをリリース。

---

## 必要な準備

### 前提条件
以下が事前に準備・設定されていることを想定しています。  
- **microCMS**: プロジェクトが作成されており、Webhookが利用可能であること。（WebhookのURLを取得できる状態）
- **GitHubリポジトリ**: ビルド環境（例: Next.jsやNuxt.jsなどの静的サイトジェネレーター）がすでにセットアップ済みであること。
- **FTP情報**: レンタルサーバー（今回はヘテムル）のホスト・ユーザー名・パスワード情報。

---

## 実際に構築したGitHub Actionsの設定ファイル

GitHub Actionsのワークフローは`.github/workflows/deploy.yml`に記述します。

### deploy.yml

```yaml
name: Deploy to Heteml Server

on:
  workflow_dispatch: # 手動でもトリガー可能
  push:
    branches:
      - main
  repository_dispatch: # microCMSのWebhook用
    types:
      - rebuild_site

jobs:
  build:
    name: Build Project
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16' # プロジェクトで指定のNode.jsバージョン

      - name: Install Dependencies
        run: npm ci # 関連パッケージをインストール

      - name: Build Project
        run: npm run build # 静的ファイルを生成

      - name: Upload via FTP
        uses: SamKirkland/FTP-Deploy-Action@4.3.0
        with:
          server: ${{ secrets.FTP_HOST }}
          username: ${{ secrets.FTP_USER }}
          password: ${{ secrets.FTP_PASSWORD }}
          local-dir: ./out # 静的ファイルが生成されるディレクトリ
          server-dir: /public_html # アップロード先パス
```

### 解説

1. **onセクション**:  
   - `workflow_dispatch`：手動でトリガーを発行可能な設定です。
   - `push`：`main`ブランチへのプッシュでワークフローが動作します。
   - `repository_dispatch`：microCMSのWebhookがGitHubに通知を送る際、このイベントがトリガーされます（後述）。

2. **stepsセクション**:  
   - リポジトリをクローンし、Node.jsをセットアップして静的サイトをビルドします。  
   - サーバーへのFTPアップロードは、[FTP-Deploy-Action](https://github.com/SamKirkland/FTP-Deploy-Action)を利用しています。  

3. **Secrets設定**:  
   - `FTP_HOST`、`FTP_USER`、`FTP_PASSWORD`はGitHub Secretsに登録しておき、セキュアにFTP情報を管理します。

---

## microCMSのWebhook設定

1. microCMSの管理画面にログインし、対象プロジェクトの「Webhook」を開く。
2. Webhook URLに以下のようなエンドポイントをセットする：  
   ```
   https://api.github.com/repos/<username>/<repo>/dispatches
   ```
   （例: `https://api.github.com/repos/myuser/myrepo/dispatches`）

3. Webhookの認証トークン（GitHub Personal Access Token）が必要です。GitHubで生成したトークンを指定。

4. トリガータイプとして必要なイベント（作成/更新/削除）を選択。

---

## 感想
今回、GitHub Actionsを活用した自動化の設定を通じて、VercelやNetlifyが不要なケースでも柔軟にデプロイフローを設計できることを学びました。  
特に、FTPを使わざるを得ない環境であっても、このようにCI/CDツールを介在させることで開発効率を大幅に向上させることができました。  
個人的には、microCMSのWebhookと合わせることで、簡易的なヘッドレスCMSワークフローを構築できる点が非常に魅力的だと感じました。  
ぜひ、似たような課題を抱えている方は、GitHub Actionsやその他のCIツールを活用して自動化を試してみてください。
