---
title: "LiteLLM × Langfuse を安く安定して動かす構成（VPS + Cloudflare）"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["litellm", "langfuse", "llm", "vultr", "cloudflare"]
published: true
---

## はじめに

LLM アプリを運用し始めると、プロンプトの管理やコスト計算、トレースの仕組みがすぐに欲しくなります。**LiteLLM** と **Langfuse** の組み合わせはそのあたりをうまくカバーしてくれるのですが、Langfuse v3 から ClickHouse が必須になったことで、セルフホストのハードルが一気に上がりました。メモリ不足（OOM）で落ちる、再起動を繰り返す——そんな運用地獄に片足を突っ込んだ末にたどり着いた構成をまとめます。

方針はシンプルで、**「まともに動いて、かつ安い」** ことだけを優先しています。

## 結論：推奨構成（月額 ¥3,000〜6,000）

| コンポーネント | サービス | プラン・スペック | 月額コスト |
|---|---|---|---|
| **Langfuse** | Langfuse Cloud | Hobby プラン（マネージド） | ¥0 |
| **LiteLLM** | 国内 VPS（東京リージョン） | 4 vCPU / 8 GB RAM | ¥3,000〜6,000 |
| **DNS / SSL** | Cloudflare | Free プラン | ¥0 |

**なぜこの構成にしたか**

最初は Langfuse も VPS にセルフホストしようとしました。が、Langfuse v3 は ClickHouse を内包していて、まともに動かすにはメモリが 8GB でもギリギリ。LiteLLM + PostgreSQL + Redis と同居させるとすぐ OOM で落ちます。ClickHouse だけ別サーバーに分ける手もありますが、月額が倍になる上に運用の手間も増える。

であれば、Langfuse は公式の Hobby プラン（無料）に任せて ClickHouse の運用をゼロにし、LiteLLM だけ東京リージョンの VPS に置くのが一番割りに合います。LLM API へのプロキシなのでレイテンシは気にしたい。国内サーバーは外せません。HTTPS は Cloudflare の DNS + Caddy の自動証明書で十分です。

### VPS はどこでもいい

この記事では Vultr を使っていますが、Docker が動く東京リージョンの VPS なら何でも大丈夫です。むしろコスト重視なら国産 VPS のほうが安いです。

| サービス | 8GB RAM プラン | 月額目安 |
|---|---|---|
| Xserver VPS | 4 vCPU / 8 GB | 約 ¥3,000〜 |
| ConoHa VPS | 4 vCPU / 8 GB | 約 ¥4,000〜 |
| さくらの VPS | 4 vCPU / 8 GB | 約 ¥5,000〜 |
| Vultr (東京) | 4 vCPU / 8 GB | 約 ¥6,000 |

自分が Vultr を使ったのは、たまたまアカウントを持っていたからです。今から始めるなら Xserver VPS あたりが半額で済むので、そっちのほうがいいと思います。手順は Ubuntu + Docker が入ればどれも同じです。

---

## 構築手順

### 1. VPS でサーバーを立ち上げる

以下は Vultr の例ですが、他の VPS でも Ubuntu + Docker が使えれば同じ手順で進められます。

- **Location**: Tokyo（国内リージョン）
- **Image**: Ubuntu 24.04 LTS
- **Plan**: 4 vCPU / 8 GB RAM 以上
- **Additional Features**: Auto Backups 等はすべて OFF（節約のため）

### 2. Cloudflare で DNS レコードを追加

取得した Vultr の IP アドレス（例: `167.179.115.77`）を Cloudflare に登録します。

- **Type**: A
- **Name**: `llmproxy` (任意のサブドメイン)
- **Content**: `167.179.115.77`
- **Proxy status**: **DNS only（グレー雲）**
  - ※ Let's Encrypt で SSL 証明書を取得するため、最初は必ずグレー雲にします。

### 3. サーバーの初期設定と Docker インストール

Vultr サーバーに SSH 接続し、Docker をインストールします。

```bash
# パッケージ更新
apt-get update -y

# Docker インストール
apt-get install -y docker.io docker-compose-v2

# 作業ディレクトリ作成
mkdir -p /opt/litellm
cd /opt/litellm
```

### 4. LiteLLM の設定ファイル作成

`/opt/litellm` ディレクトリに以下の3つのファイルを作成します。

#### `.env`
```env
# LiteLLM Master Key
LITELLM_MASTER_KEY="sk-litellm-your-secret-key"

# Langfuse 連携設定
LANGFUSE_PUBLIC_KEY="pk-lf-..."
LANGFUSE_SECRET_KEY="sk-lf-..."
LANGFUSE_HOST="https://us.cloud.langfuse.com" # USリージョンの場合

# PostgreSQL 設定
POSTGRES_USER="litellm"
POSTGRES_PASSWORD="your_db_password"
POSTGRES_DB="litellm"
DATABASE_URL="postgresql://litellm:your_db_password@db:5432/litellm"
```

#### `litellm_config.yaml`
```yaml
model_list:
  # ここに OpenAI などのモデル設定を追記していく

litellm_settings:
  success_callback: ["langfuse"]
  failure_callback: ["langfuse"]
  cache: true
  cache_params:
    type: "redis"
    host: "redis"
    port: 6379

general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
  database_url: "os.environ/DATABASE_URL"
```

#### `docker-compose.yml`
```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    container_name: litellm
    ports:
      - "4000:4000"
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
    env_file:
      - .env
    command: [ "--config", "/app/config.yaml", "--port", "4000", "--num_workers", "4" ]
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: always

  db:
    image: postgres:15-alpine
    container_name: litellm-db
    env_file:
      - .env
    volumes:
      - litellm_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U litellm -d litellm"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: always

  redis:
    image: redis:7-alpine
    container_name: litellm-redis
    volumes:
      - litellm_redis_data:/data
    command: redis-server --save 60 1 --loglevel warning
    restart: always

volumes:
  litellm_db_data:
  litellm_redis_data:
```

### 5. LiteLLM の起動

```bash
docker compose up -d
```

### 6. ファイアウォール（UFW）の設定

Vultr はデフォルトで UFW が有効になっていて、SSH（22）しか通していません。Caddy で HTTPS 化するには **ポート 80 と 443** を開ける必要があります。ここを忘れると Let's Encrypt の ACME チャレンジがタイムアウトして、証明書が一生取れません。

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw status
```

### 7. Caddy で HTTPS 化

Nginx でもいいですが、Caddy のほうが設定が圧倒的に短い。SSL 証明書の取得・更新も勝手にやってくれます。

```bash
# Caddy インストール
apt-get install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt-get update -y
apt-get install -y caddy

# Caddyfile の作成
cat << 'EOF' > /etc/caddy/Caddyfile
llmproxy.yourdomain.com {
    reverse_proxy localhost:4000
}
EOF

# Caddy 起動
systemctl enable caddy
systemctl restart caddy
```

数分待つと Let's Encrypt から証明書が取得され、`https://llmproxy.yourdomain.com` でアクセスできるようになります。

> **Note:** 証明書が取得できたら、Cloudflare の Proxy status を「Proxied（オレンジ雲）」に戻しても問題ありません。

## トラブルシューティング

### SSL 証明書が取得できない

Caddy のログに `Timeout during connect (likely firewall problem)` が出ている場合、UFW でポート 80/443 が開いていない可能性が高いです。

```bash
# ログ確認
journalctl -u caddy --no-pager -n 30

# ポート開放
ufw allow 80/tcp && ufw allow 443/tcp

# Caddy 再起動
systemctl restart caddy
```

なお、失敗が短時間に重なると Let's Encrypt 側でレート制限がかかります。その場合は `retry after ...` のログに表示される時刻まで待ってから再試行してください。

### Cloudflare でオレンジ雲にしたら証明書更新が失敗する

Cloudflare の Proxy を有効（オレンジ雲）にすると、Let's Encrypt の HTTP-01 チャレンジが Cloudflare のエッジで止まり、Caddy に到達しません。証明書の初回取得・更新時は一時的に「DNS only（グレー雲）」に切り替え、取得完了後にオレンジ雲に戻してください。

## まとめ

月額約6,000円で、東京リージョンの低遅延プロキシと安定した Langfuse の分析基盤が手に入ります。ClickHouse の面倒を見なくていいだけで、運用のストレスがだいぶ減りました。

リクエストが増えて Langfuse Cloud の無料枠を超えたら、有料プランに上げるか、別サーバーにセルフホスト版を立てて `.env` の `LANGFUSE_HOST` を書き換えるだけ。LiteLLM 側は何も変えなくていいので、移行もスムーズです。
