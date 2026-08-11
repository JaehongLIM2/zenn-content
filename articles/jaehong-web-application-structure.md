---
title: "React・Spring Boot・Docker・AWSの役割をWebアプリ全体像から整理する"
emoji: "🌐"
type: "tech"
topics: ["react", "springboot", "aws", "docker", "web"]
published: false
---

以前のチーム開発では、React、Spring Boot、MariaDB、Docker、AWSを組み合わせてWebアプリケーションを構築しました。

それぞれの使い方を個別に覚えていても、「どの技術がどこまで担当していたのか」は曖昧になりやすいものです。そこで今回は、ユーザーの操作がデータベースに届くまでの流れを追いながら、各技術の役割を整理します。

## Webアプリケーションの基本構成

一般的なWebアプリケーションは、主に次の3つの領域に分けられます。

```text
ユーザー
  ↓
フロントエンド
  ↓
バックエンド
  ↓
データベース
```

今回扱う技術を当てはめると、次のようになります。

```text
ブラウザ
  ↓
React
  ↓ REST API
Spring Boot
  ↓ SQL
MariaDB
```

ここにDockerとAWSを加えると、アプリケーションを実行・公開するための構成まで含めた全体像になります。

| 領域 | 技術例 | 主な役割 |
|---|---|---|
| フロントエンド | React | 画面表示とユーザー操作 |
| バックエンド | Spring Boot | API、認証、業務ロジック |
| データベース | MariaDB | データの永続化 |
| コンテナ | Docker | 実行環境のパッケージ化 |
| インフラ | AWS | インターネット上で実行する環境 |

DockerとAWSは、画面やAPIの機能を直接作るものではありません。作成したアプリケーションを「どの環境で、どのように動かすか」を担当します。

## Reactが担当すること

Reactはブラウザ上のUIを構築します。入力フォーム、一覧、ボタンなどをコンポーネントに分け、状態に応じて画面を更新します。

```text
投稿一覧画面
├─ Header
├─ SearchForm
├─ PostList
│  └─ PostItem
└─ Pagination
```

一方で、ユーザーの権限確認や注文金額の確定といった重要な判断をブラウザだけに任せることはできません。ブラウザに配信されたコードはユーザー側から確認・変更できるためです。

そのため、信頼できる環境で行うべき処理はバックエンドに送ります。

## Spring Bootが担当すること

Spring Bootで構築したバックエンドは、フロントエンドから受け取ったリクエストを検証し、必要なロジックを実行します。

- ログイン状態と権限の確認
- 入力値の検証
- 業務ルールに基づく計算
- データベースの検索・更新
- JSON形式での結果返却

たとえば、ユーザーが投稿を作成する場合は次のように処理が流れます。

```text
1. Reactがタイトルと本文をAPIへ送信する
2. Spring Bootが認証状態と入力値を確認する
3. 問題がなければMariaDBへ保存する
4. 保存結果をReactへ返す
5. Reactが完了画面を表示する
```

この境界を明確にすると、「画面で行う処理」と「サーバーで保証する処理」を分けて考えやすくなります。

## MariaDBが担当すること

MariaDBは、ユーザー、投稿、注文など、サービスが継続して保持するデータを保存します。

```text
users
├─ id
├─ email
└─ name

posts
├─ id
├─ user_id → users.id
├─ title
└─ content
```

バックエンドはSQLやORMを通じてデータベースへアクセスします。一般的な構成では、ブラウザから本番データベースへ直接接続させず、バックエンドで権限と業務ルールを管理します。

SupabaseのData APIのようにブラウザからデータを扱える仕組みもありますが、その場合もRow Level Securityなどのアクセス制御が必要です。

## Dockerが担当すること

Spring Bootのソースコードだけではアプリケーションを実行できません。Javaのバージョン、依存ファイル、環境変数、起動コマンドなどが必要です。

Dockerはこれらの実行条件をイメージとしてまとめます。

```text
Spring Bootアプリケーション
+ Java実行環境
+ 必要なファイル
+ 起動コマンド
        ↓
Dockerイメージ
        ↓ 実行
Dockerコンテナ
```

DockerがSpring Bootの代わりになるわけではありません。Spring Bootを異なる環境でも同じ条件で起動しやすくする役割です。

## AWSが担当すること

ローカルPCで動いているアプリケーションは、そのままでは外部のユーザーが利用できません。公開するには、サーバー、ネットワーク、HTTPS、ドメインなどの環境が必要です。

AWSでは、たとえば次のような構成を選択できます。

```text
Reactの静的ファイル → S3 + CloudFront
Spring Boot         → EC2またはECS
MariaDB             → RDS
```

AWSは単一のデプロイツールではなく、コンピューティング、ストレージ、データベース、ネットワークなどを組み合わせられるクラウドプラットフォームです。

## ログイン処理で全体を確認する

ログインを例にすると、各技術の関係が分かりやすくなります。

```text
ユーザー
  ↓ メールアドレスとパスワードを入力
React
  ↓ POST /api/login
Spring Boot
  ↓ ユーザーを検索
MariaDB
  ↑ 検索結果
Spring Boot
  ↓ 認証結果を返す
React
  ↓ ログイン後の画面を表示
```

Spring BootがDockerコンテナとしてAWS上で実行されている場合、DockerとAWSはこの処理の外側で実行環境と稼働場所を支えます。

## まとめ

今回の構成は、次の一文で整理できます。

> Reactで画面を作り、Spring BootでAPIと業務ロジックを処理し、MariaDBにデータを保存する。Dockerで実行環境をそろえ、AWS上でサービスを稼働させる。

新しい技術を調べるときも、UI、サーバーロジック、データ保存、実行環境、インフラのどこを担当するものか確認すると、既存技術との関係を理解しやすくなります。

## 参考資料

- [React公式ドキュメント](https://react.dev/)
- [Spring Boot公式ドキュメント](https://docs.spring.io/spring-boot/)
- [AWSの製品一覧](https://aws.amazon.com/jp/products/)
- [Docker Get Started](https://docs.docker.com/get-started/)
- [Supabase: Securing your data](https://supabase.com/docs/guides/database/secure-data)
