---
title: "Docker入門：イメージ・コンテナ・Composeから本番デプロイまで整理する"
emoji: "🐳"
type: "tech"
topics: ["docker", "dockercompose", "aws", "springboot"]
published: false
---

チーム開発でDockerを使っていても、`docker compose up`を実行するツールという理解だけでは、イメージとコンテナの違いやデプロイで使う理由が分かりにくいことがあります。

Dockerの役割は、アプリケーションと実行に必要なファイル・設定をイメージとしてまとめ、隔離されたコンテナプロセスとして実行できるようにすることです。

言い換えると、**実行環境をコードとして管理するための仕組み**です。

## Dockerが解決する環境差分

同じSpring Bootプロジェクトを複数の環境で実行する場合を考えます。

```text
開発者A  : Windows + Java 17 + MariaDB 10
開発者B  : macOS   + Java 21 + MariaDB 11
本番サーバー: Linux   + Javaなし + DBなし
```

ソースコードが同じでも、次の違いによってエラーが発生する可能性があります。

- Javaとデータベースのバージョン
- OSとシステムライブラリ
- 環境変数と設定ファイル
- 起動コマンドとポート
- インストールされている依存関係

Dockerでは必要な実行条件をファイルに定義し、同じイメージを開発者、CI、本番環境で利用することで差分を減らします。

## Dockerイメージとは

コンテナイメージは、コンテナの実行に必要なファイル、バイナリ、ライブラリ、設定を含む標準化されたパッケージです。

Spring Bootの場合、次のような流れでイメージを作成します。

```text
Springのソースコード
  ↓ ビルド
app.jar
  +
Dockerfile
  ↓ docker build
Dockerイメージ
```

イメージは実行中のプロセスではありません。実行環境を再現するための成果物です。作成済みのイメージを直接変更するのではなく、Dockerfileを修正して新しいイメージをビルドします。

```text
イメージ = 実行に必要な内容をまとめた変更不可能なパッケージ
```

## Dockerコンテナとは

イメージを実行するとコンテナになります。

```text
Dockerイメージ
  ↓ docker run
コンテナ
```

コンテナは、ホスト上で隔離されて実行されるプロセスです。同じイメージから複数のコンテナを起動できます。

```text
                       ┌─ Springコンテナ1
Springイメージ ────────├─ Springコンテナ2
                       └─ Springコンテナ3
```

各コンテナは同じ環境から起動しますが、実行中の状態は独立しています。

コンテナは交換できるものとして扱います。データベースファイルのように残す必要があるデータはコンテナ内だけに保存せず、ボリュームまたは外部のマネージドデータベースに置きます。

## Dockerfileを読む

ビルド済みの`app.jar`を実行する、簡単なDockerfileは次のようになります。

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY app.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

各命令の役割は次のとおりです。

| 命令 | 役割 |
|---|---|
| `FROM` | Java 17の実行環境をベースイメージにする |
| `WORKDIR` | コンテナ内の作業ディレクトリを指定する |
| `COPY` | `app.jar`をイメージへコピーする |
| `ENTRYPOINT` | コンテナ起動時のコマンドを指定する |

このDockerfileからイメージを作成し、コンテナとして起動します。

```bash
docker build -t my-spring-app .
docker run -p 8080:8080 my-spring-app
```

1つ目のコマンドは、現在のディレクトリにあるDockerfileから`my-spring-app`イメージを作成します。2つ目はホストの8080ポートをコンテナの8080ポートへ接続して実行します。

## Docker Composeが必要になる理由

Webアプリケーションは、1つのコンテナだけで完結しないことがあります。

```text
フロントエンド
バックエンド
データベース
Redis
```

個別の`docker run`で起動すると、ポート、環境変数、ネットワーク、ボリュームの指定が増えます。Docker Composeでは、複数コンテナの実行構成を1つのYAMLファイルへ定義できます。

以下はSpring BootとMariaDBを一緒に実行するための、単純化した例です。

```yaml
services:
  backend:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mariadb://db:3306/teamproject
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mariadb:11.4
    environment:
      MARIADB_DATABASE: teamproject
      MARIADB_USER: app
      MARIADB_PASSWORD: ${DB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  db-data:
```

パスワードを`.env`などに設定し、次のコマンドで起動します。

```bash
docker compose up -d --build
```

この構成では、次の点を確認できます。

- バックエンドは`localhost`ではなくサービス名の`db`でデータベースへ接続する
- `db-data`ボリュームにデータベースのデータを保存する
- `healthcheck`と`service_healthy`でデータベースの準備完了後にバックエンドを開始する

実際のパスワードを保存した`.env`はGitへコミットしません。本番環境では、利用するプラットフォームのシークレット管理機能を使います。

## DockerfileとComposeファイルの違い

2つのファイルは目的が異なります。

| ファイル | 役割 |
|---|---|
| `Dockerfile` | 1つのコンテナイメージを作る方法を定義する |
| `compose.yaml` | 複数コンテナをどの設定で実行するか定義する |

DockerfileでSpring Bootの実行環境を作り、ComposeファイルでSpring BootコンテナとMariaDB、Redisなどを接続します。

## Dockerと仮想マシンの違い

仮想マシンはゲストOS全体を実行します。コンテナは一般にホストのカーネルを共有しながらプロセスを隔離します。

```text
仮想マシン
ホストOS
└─ ハイパーバイザー
   └─ ゲストOS
      └─ アプリケーション
```

```text
コンテナ
ホストOS
└─ コンテナランタイム
   ├─ アプリケーションA
   └─ アプリケーションB
```

この構造により、コンテナは一般にVMより短時間で起動し、必要なリソースも少なくなります。

ただし、WindowsやmacOSのDocker DesktopでLinuxコンテナを実行する場合、内部ではLinux VMが使われます。「DockerではVMをまったく使わない」と理解すると正確ではありません。また、コンテナとVMでは隔離の仕組みが異なるため、セキュリティ設計で同じ境界として扱うこともできません。

## チーム開発でDockerを使う利点

### 実行環境をそろえる

同じイメージとComposeファイルを利用すると、チームメンバー間のJavaやデータベースバージョンの違いを減らせます。

```text
Gitリポジトリ
├─ ソースコード
├─ Dockerfile
└─ compose.yaml
       ↓
開発者A / 開発者B / CIが同じ構成を実行
```

### プロジェクトへの参加を簡単にする

READMEに複数のミドルウェアを手動インストールする手順を並べる代わりに、Dockerを準備した後、Composeで必要なサービスを起動できます。

### CIと本番環境の差を減らす

テストを通過したイメージをレジストリへプッシュし、本番環境でも同じイメージIDを実行すると、デプロイする成果物を明確にできます。

## コンテナを使ったデプロイ

コンテナを使うデプロイは、次のような流れになります。

```text
ソースコード
  ↓ ビルドとテスト
Dockerイメージ
  ↓ プッシュ
コンテナレジストリ
  ↓ デプロイ
AWS ECS / Kubernetes / Cloud Run / Container Apps
```

サーバーごとにSSH接続してJavaをインストールし、JARファイルを配置する代わりに、デプロイプラットフォームへ実行するイメージを指定します。

トラフィックが増えた場合は、同じイメージからコンテナ数を増やし、ロードバランサーでリクエストを分散できます。

```text
                ┌─ コンテナ1
ユーザー → LB ──├─ コンテナ2
                └─ コンテナ3
```

コンテナ数を増やすだけで自動的にスケーリングできるわけではありません。セッションを外部へ保存するステートレス設計、データベースの接続数、同時実行、モニタリング、コストも合わせて検討します。

## 開発用データベースと本番データベースを分けて考える

ローカル開発では、MariaDBやPostgreSQLをComposeで起動すると、インストールと初期化を簡単にできます。

本番環境では、別の判断が必要です。

```text
開発環境 → Docker Composeのデータベースコンテナ
本番環境 → RDSなどのマネージドデータベース
```

本番でデータベースコンテナを運用することもできますが、バックアップ、リストア、ディスク障害、バージョン更新、高可用性を自分たちで担当します。運用範囲を小さくしたい場合はマネージドデータベースが候補になります。

## まとめ

Dockerの基本を5つにまとめると、次のようになります。

1. Dockerfileはイメージの作り方を定義する
2. イメージは実行に必要な内容をまとめたイミュータブルなパッケージ
3. コンテナはイメージから起動する隔離されたプロセス
4. Composeは複数コンテナの実行構成を管理する
5. 永続データとシークレットをコンテナ内だけに置かない

Dockerの価値は、コマンドを短くすることだけではありません。実行環境を再現可能なコードとして扱える点にあります。この考え方を理解すると、ローカル開発の環境統一とコンテナを使ったクラウドデプロイを1つの流れとして捉えられます。

## 参考資料

- [Docker: What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Docker: What is an image?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
- [What is Docker Compose?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-docker-compose/)
- [Control startup and shutdown order in Compose](https://docs.docker.com/compose/how-tos/startup-order/)
- [MariaDB: Using healthcheck.sh](https://mariadb.com/docs/server/server-management/automated-mariadb-deployment-and-administration/docker-and-mariadb/using-healthcheck-sh)
