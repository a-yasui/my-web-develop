# Local Development Traefik

このプロジェクトは、複数のwebサーバのコンテナを扱うためのリバースプロキシーサーバです。[Traefik](https://traefik.io/)を使用して、開発環境でのマイクロサービスへのルーティングを簡単に行えるようにします。

あたしの身近の web 開発は、ほとんどが docker compose を使ってローカル環境で開発をしています。 そのため複数の web 開発プロジェクトで、それぞれのサービスを同時に立ち上げる必要がある事があります。それぞれが 80 番ポートを使うため、衝突がおきてしまいます。それを回避するための方法として、それぞれ別々のポートにする方法がありますが、どれがどの番号なのか忘れた為に確認しないといけないといった問題があります。

このリポジトリは、個々のサービスを traefik に接続させる事で、 `hoge.localhost` といったドメインでアクセスできるようにして、ポート番号の衝突問題を解決する事を主にしています。

## 概要

このプロジェクトは以下の機能を提供します：

- Traefikを使用したリバースプロキシー
- ローカル開発環境でのHTTPルーティング
- Grafana、Lokiを使用したログ収集と可視化(option)
- Docker Composeによる簡単なセットアップ

## 構成

プロジェクトは以下のコンポーネントで構成されています：

### メインコンポーネント

- **Traefik**: リバースプロキシーサーバ（ポート80、8080）

#### Option

起動してもしなくても、どちらでもという感じ。500 エラー時のログ取得とか通知とかがない、等といった設定が不十分であり、作成中。

- **Grafana**: メトリクスとログの可視化（`grafana.localhost`でアクセス, user:admin, password: change-me で初期アクセスします）
- **Loki**: ログ集約システム
- **Grafana Agent**: コンテナログの収集

**memo**: Grafana/Loki/Agent の設定はまだ不十分です。

## セットアップ方法

### 前提条件

- Docker と Docker Compose がインストールされていること
- 外部ネットワーク `develop-net` が作成されていること

### インストール手順

1. リポジトリをクローンします：

```bash
git clone git@github.com:a-yasui/my-web-develop.git
cd local-devel-traefik
```

2. 外部ネットワークを作成します（まだ存在しない場合）：

```bash
docker network create develop-net
```

3. Traefikサーバーを起動します：

```bash
docker compose up -d
```

4. （オプション）ロギングシステムを起動します：

```bash
docker compose -f compose.yml -f compose.logging.yml up -d
```

## 使用方法

### Traefikダッシュボード

Traefikダッシュボードには以下のURLでアクセスできます：
- http://traefik.localhost

### 他のサービスを追加

他のDockerコンテナをTraefikに接続するには、既存の `docker-compose.yml` を上書き設定する `docker-compose.overrride.yml` を作成して、以下のようなラベルを設定します。保存の後、 `docker compsoe -f docker-compose.yml -f docker-compose.override.yml up --build -d` という形で起動して使用します。

```yaml
services:
  app:
    # ポートは公開せず、Traefikを通してアクセスする
    ports:
      !reset []
    # develop-netネットワークに接続する
    networks:
      - develop-net
    # Traefik設定
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=develop-net"
      - "traefik.http.routers.<サービス名>.rule=Host(`<ホスト名>.localhost`)"
      - "traefik.http.services.<サービス名>.loadbalancer.server.port=<コンテナ内ポート>"
      - "traefik.http.routers.<サービス名>.entrypoints=web"
networks:
  develop-net:
    external: true
```

実際の例（Grafanaサービスの場合）：

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.docker.network=develop-net"
  - "traefik.http.routers.grafana.rule=Host(`grafana.localhost`)"
  - "traefik.http.services.grafana.loadbalancer.server.port=3000"
  - "traefik.http.routers.grafana.entrypoints=web"
```

必要に応じて、セキュリティヘッダーなどのミドルウェアも設定できます：

```yaml
  - "traefik.http.middlewares.<ミドルウェア名>.headers.frameDeny=true"
  - "traefik.http.middlewares.<ミドルウェア名>.headers.browserXssFilter=true"
  - "traefik.http.routers.<サービス名>.middlewares=<ミドルウェア名>"
```

### ログの確認

docker compose で起動時に `compose.logging.yml` も追加している時、Grafanaダッシュボードには以下のURLでアクセスできます：
- http://grafana.localhost

初期ログイン情報：
- ユーザー名: admin
- パスワード: change-me (compose.logging.ymlで設定)

## 設定ファイル

- `compose.yml`: メインのTraefikサービス設定
- `compose.logging.yml`: ロギングシステム（Grafana、Loki、Grafana Agent）の設定
- `config/develop.traefik.yml`: Traefikの主要設定
- `traefik-dynamic/`: 動的設定ファイル
- `config/agent.river`: Grafana Agentの設定
- `config/loki-config.yml`: Lokiの設定

## ネットワーク

このプロジェクトは `develop-net` という外部ネットワークを使用して、他のプロジェクトのコンテナと通信します。これにより、異なるプロジェクト間でのサービスディスカバリーが可能になります。

## 注意事項

- このプロジェクトは開発環境用に設計されており、本番環境での使用は推奨されません。
- セキュリティ設定は開発用に最適化されています（例：APIのinsecureモードが有効）。
