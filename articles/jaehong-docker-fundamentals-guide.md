---
title: "Docker入門：Image・Container・Composeから本番デプロイまで整理する"
emoji: "🐳"
type: "tech"
topics: ["docker", "dockercompose", "aws", "springboot"]
published: false
---

チーム開発でDockerを使っていても、`docker compose up`を実行するToolという理解だけでは、ImageとContainerの違いやデプロイで使う理由が分かりにくいことがあります。

Dockerの役割は、アプリケーションと実行に必要なFile・設定をImageとしてまとめ、隔離されたContainer Processとして実行できるようにすることです。

言い換えると、**実行環境をCodeとして管理するための仕組み**です。

## Dockerが解決する環境差分

同じSpring Boot Projectを複数の環境で実行する場合を考えます。

```text
開発者A  : Windows + Java 17 + MariaDB 10
開発者B  : macOS   + Java 21 + MariaDB 11
本番Server: Linux   + Javaなし + DBなし
```

Source Codeが同じでも、次の違いによってErrorが発生する可能性があります。

- JavaとDatabaseのVersion
- OSとSystem Library
- Environment Variableと設定File
- 起動CommandとPort
- InstallされているDependency

Dockerでは必要な実行条件をFileに定義し、同じImageを開発者、CI、本番環境で利用することで差分を減らします。

## Docker Imageとは

Container Imageは、Containerの実行に必要なFile、Binary、Library、設定を含む標準化されたPackageです。

Spring Bootの場合、次のような流れでImageを作成します。

```text
Spring Source
  ↓ Build
app.jar
  +
Dockerfile
  ↓ docker build
Docker Image
```

Imageは実行中のProcessではありません。実行環境を再現するための成果物です。作成済みのImageを直接変更するのではなく、Dockerfileを修正して新しいImageをBuildします。

```text
Image = 実行に必要な内容をまとめたImmutableなPackage
```

## Docker Containerとは

Imageを実行するとContainerになります。

```text
Docker Image
  ↓ docker run
Container
```

Containerは、Host上で隔離されて実行されるProcessです。同じImageから複数のContainerを起動できます。

```text
                 ┌─ Spring Container 1
Spring Image ────├─ Spring Container 2
                 └─ Spring Container 3
```

各Containerは同じ環境から起動しますが、実行中の状態は独立しています。

Containerは交換できるものとして扱います。Database Fileのように残す必要があるDataはContainer内だけに保存せず、Volumeまたは外部のManaged Databaseに置きます。

## Dockerfileを読む

Build済みの`app.jar`を実行する、簡単なDockerfileは次のようになります。

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY app.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

各Instructionの役割は次のとおりです。

| Instruction | 役割 |
|---|---|
| `FROM` | Java 17の実行環境をBase Imageにする |
| `WORKDIR` | Container内の作業Directoryを指定する |
| `COPY` | `app.jar`をImageへコピーする |
| `ENTRYPOINT` | Container起動時のCommandを指定する |

このDockerfileからImageを作成し、Containerとして起動します。

```bash
docker build -t my-spring-app .
docker run -p 8080:8080 my-spring-app
```

1つ目のCommandは、現在のDirectoryにあるDockerfileから`my-spring-app` Imageを作成します。2つ目はHostの8080 PortをContainerの8080 Portへ接続して実行します。

## Docker Composeが必要になる理由

Webアプリケーションは、1つのContainerだけで完結しないことがあります。

```text
Frontend
Backend
Database
Redis
```

個別の`docker run`で起動すると、Port、Environment Variable、Network、Volumeの指定が増えます。Docker Composeでは、複数Containerの実行構成を1つのYAML Fileへ定義できます。

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

Passwordを`.env`などに設定し、次のCommandで起動します。

```bash
docker compose up -d --build
```

この構成では、次の点を確認できます。

- Backendは`localhost`ではなくService名の`db`でDatabaseへ接続する
- `db-data` VolumeにDatabaseのDataを保存する
- `healthcheck`と`service_healthy`でDatabaseの準備完了後にBackendを開始する

実際のPasswordを保存した`.env`はGitへCommitしません。本番環境では、利用するPlatformのSecret管理機能を使います。

## DockerfileとCompose Fileの違い

2つのFileは目的が異なります。

| File | 役割 |
|---|---|
| `Dockerfile` | 1つのContainer Imageを作る方法を定義する |
| `compose.yaml` | 複数Containerをどの設定で実行するか定義する |

DockerfileでSpring Bootの実行環境を作り、Compose FileでSpring Boot ContainerとMariaDB、Redisなどを接続します。

## DockerとVirtual Machineの違い

Virtual MachineはGuest OS全体を実行します。Containerは一般にHostのKernelを共有しながらProcessを隔離します。

```text
Virtual Machine
Host OS
└─ Hypervisor
   └─ Guest OS
      └─ Application
```

```text
Container
Host OS
└─ Container Runtime
   ├─ Application A
   └─ Application B
```

この構造により、Containerは一般にVMより短時間で起動し、必要なResourceも少なくなります。

ただし、WindowsやmacOSのDocker DesktopでLinux Containerを実行する場合、内部ではLinux VMが使われます。「DockerではVMをまったく使わない」と理解すると正確ではありません。また、ContainerとVMでは隔離の仕組みが異なるため、Security設計で同じ境界として扱うこともできません。

## チーム開発でDockerを使う利点

### 実行環境をそろえる

同じImageとCompose Fileを利用すると、Team Member間のJavaやDatabase Versionの違いを減らせます。

```text
Git Repository
├─ Source Code
├─ Dockerfile
└─ compose.yaml
       ↓
開発者A / 開発者B / CIが同じ構成を実行
```

### Projectへの参加を簡単にする

READMEに複数のMiddlewareを手動Installする手順を並べる代わりに、Dockerを準備した後、Composeで必要なServiceを起動できます。

### CIと本番環境の差を減らす

Testを通過したImageをRegistryへPushし、本番環境でも同じImage IDを実行すると、Deployする成果物を明確にできます。

## Containerを使ったデプロイ

Containerを使うデプロイは、次のような流れになります。

```text
Source Code
  ↓ Build & Test
Docker Image
  ↓ Push
Container Registry
  ↓ Deploy
AWS ECS / Kubernetes / Cloud Run / Container Apps
```

ServerごとにSSH接続してJavaをInstallし、JAR Fileを配置する代わりに、Deploy Platformへ実行するImageを指定します。

Trafficが増えた場合は、同じImageからContainer数を増やし、Load BalancerでRequestを分散できます。

```text
                ┌─ Container 1
ユーザー → LB ──├─ Container 2
                └─ Container 3
```

Container数を増やすだけで自動的にScalingできるわけではありません。Sessionを外部へ保存するStateless設計、Databaseの接続数、同時実行、Monitoring、Costも合わせて検討します。

## 開発用Databaseと本番Databaseを分けて考える

Local開発では、MariaDBやPostgreSQLをComposeで起動すると、Installと初期化を簡単にできます。

本番環境では、別の判断が必要です。

```text
開発環境 → Docker ComposeのDatabase Container
本番環境 → RDSなどのManaged Database
```

本番でDatabase Containerを運用することもできますが、Backup、Restore、Disk障害、Version Update、High Availabilityを自分たちで担当します。運用範囲を小さくしたい場合はManaged Databaseが候補になります。

## まとめ

Dockerの基本を5つにまとめると、次のようになります。

1. DockerfileはImageの作り方を定義する
2. Imageは実行に必要な内容をまとめたImmutableなPackage
3. ContainerはImageから起動する隔離されたProcess
4. Composeは複数Containerの実行構成を管理する
5. 永続DataとSecretをContainer内だけに置かない

Dockerの価値は、Commandを短くすることだけではありません。実行環境を再現可能なCodeとして扱える点にあります。この考え方を理解すると、Local開発の環境統一とContainerを使ったCloud Deployを1つの流れとして捉えられます。

## 参考資料

- [Docker: What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Docker: What is an image?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
- [What is Docker Compose?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-docker-compose/)
- [Control startup and shutdown order in Compose](https://docs.docker.com/compose/how-tos/startup-order/)
- [MariaDB: Using healthcheck.sh](https://mariadb.com/docs/server/server-management/automated-mariadb-deployment-and-administration/docker-and-mariadb/using-healthcheck-sh)
