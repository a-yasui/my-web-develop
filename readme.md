# Local Development Traefik

このプロジェクトは、複数のWebサーバーコンテナを扱うためのリバースプロキシサーバーです。[Traefik](https://traefik.io/)を使用して、開発環境でのマイクロサービスへのルーティングを簡単に行えるようにします。

私の身近なWeb開発では、ほとんどがDocker Composeを使ってローカル環境で開発をしています。そのため複数のWeb開発プロジェクトで、それぞれのサービスを同時に立ち上げる必要があることがあります。それぞれが80番ポートを使うため、衝突が起きてしまいます。それを回避する方法として、それぞれ別々のポートにする方法がありますが、どれがどの番号なのか忘れてしまい、確認が必要になるといった問題があります。

このリポジトリは、個々のサービスをTraefikに接続させることで、`hoge.localhost`といったドメインでアクセスできるようにして、ポート番号の衝突問題を解決することを主な目的としています。

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

#### Optional

起動してもしなくても、どちらでも構いません。500エラー時のログ取得や通知などの設定が不十分で、現在作成中です。

- **Grafana**: メトリクスとログの可視化（`grafana.localhost`でアクセス、初期ログイン：user=admin, password=change-me）
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

他のDockerコンテナをTraefikに接続するには、既存の`docker-compose.yml`を上書き設定する`docker-compose.override.yml`を作成して、以下のようなラベルを設定します。保存後、`docker compose -f docker-compose.yml -f docker-compose.override.yml up --build -d`の形で起動して使用します。

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

Docker Compose起動時に`compose.logging.yml`も追加している場合、Grafanaダッシュボードには以下のURLでアクセスできます：
- http://grafana.localhost

初期ログイン情報：
- ユーザー名: admin
- パスワード: change-me（compose.logging.ymlで設定）

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
