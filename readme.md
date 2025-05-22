# Local Development Traefik

このプロジェクトは、複数のwebサーバのコンテナを扱うためのリバースプロキシーサーバです。[Traefik](https://traefik.io/)を使用して、開発環境でのマイクロサービスへのルーティングを簡単に行えるようにします。

## 概要

このプロジェクトは以下の機能を提供します：

- Traefikを使用したリバースプロキシー
- ローカル開発環境でのHTTPルーティング
- Grafana、Lokiを使用したログ収集と可視化
- Docker Composeによる簡単なセットアップ

## 構成

プロジェクトは以下のコンポーネントで構成されています：

### メインコンポーネント

- **Traefik**: リバースプロキシーサーバ（ポート80、8080）
- **Grafana**: メトリクスとログの可視化（`grafana.degg`または`grafana.degg.biz`でアクセス）
- **Loki**: ログ集約システム
- **Grafana Agent**: コンテナログの収集

## セットアップ方法

### 前提条件

- Docker と Docker Compose がインストールされていること
- 外部ネットワーク `develop-net` が作成されていること

### インストール手順

1. リポジトリをクローンします：

```bash
git clone <repository-url>
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
- http://traefik.local

※ローカルホストファイルに `127.0.0.1 traefik.local` を追加してください。

### 他のサービスの追加

他のDockerコンテナをTraefikに接続するには、以下のようにラベルを設定します。

そして override する形で保存し、 `docker compsoe -f docker-compose.yml -f docker-compose.override.yml up --build -d` という形で使用します。

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
      - "traefik.http.routers.<サービス名>.rule=Host(`<ホスト名>.local`)"
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
  - "traefik.http.routers.grafana.rule=Host(`grafana.local`)"
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

Grafanaダッシュボードには以下のURLでアクセスできます：
- http://grafana.degg または http://grafana.degg.biz

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
