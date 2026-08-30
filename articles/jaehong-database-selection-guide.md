---
title: "PostgreSQL・MariaDB・Redis・Supabase・Neonの使い分けを整理する"
emoji: "🗄️"
type: "tech"
topics: ["postgresql", "mariadb", "redis", "supabase", "neon"]
published: true
---

Webアプリケーションでデータベースを選ぶとき、「最近よく使われているもの」だけを基準にすると判断が難しくなります。

データ構造、トランザクション、チームの経験、運用方法によって適切な選択肢は変わります。今回はPostgreSQL、MySQL・MariaDB、Redisの役割と、Supabase、Neonの違いを整理します。

> サービスの提供範囲は変化が速いため、SupabaseとNeonに関する記述は2026年8月30日時点の公式情報を基準にしています。

## 最初にリレーショナルデータベースを確認する

ユーザー、投稿、注文などのデータは、互いに関係を持ちます。

```text
users
└─ id

posts
├─ id
└─ user_id → users.id
```

PostgreSQL、MySQL、MariaDB、Oracle、SQL Serverは代表的なリレーショナルデータベースです。データをテーブル、行、列で構造化し、SQLを使って操作します。

リレーショナルデータベースには、データの整合性を守るための仕組みがあります。外部キーを使えば存在しないユーザーと投稿が関連付けられることを防げます。また、トランザクションによって複数の更新を1つの処理として扱えます。

```text
注文を作成
+ 在庫を減らす
+ 決済記録を保存
= 1つのトランザクション
```

途中でエラーが発生した場合は全体をロールバックし、不整合な状態が残らないように設計できます。

## PostgreSQL：新規プロジェクトの有力な選択肢

PostgreSQLはオープンソースのオブジェクトリレーショナルデータベースです。一般的なリレーショナルデータに加えて、JSON/JSONB、配列、全文検索、複数のインデックス、拡張機能を利用できます。

次のような要件で候補になります。

- 新しいWebサービスで汎用的なデータベースが必要
- 複雑なクエリとトランザクションがある
- JSONとリレーショナルデータを一緒に扱う
- 地理情報やベクトルなどの拡張機能を利用する
- SupabaseやNeonを検討している

機能が多いからといって、最初からすべて使う必要はありません。一般的なCRUDアプリケーションでも標準的なリレーショナルデータベースとして利用できます。

## MySQL・MariaDB：既存の知識と資産を生かす

MySQLとMariaDBは、Webサービスで長く利用されてきました。ホスティング、ライブラリ、運用ツールが多く、既存システムで扱う機会もあります。

以前のチーム開発で利用した構成は次のようなものでした。

```text
React
  ↓
Spring Boot
  ↓
MariaDB
```

チームにMariaDBの運用経験があり、既存スキーマもMySQL系であれば、流行だけを理由にPostgreSQLへ変更する必要はありません。

データベースの移行では、SQL構文だけでなく、データ移行、バックアップ、モニタリング、障害対応も変わります。新しいプロジェクトで比較するときは、次の項目を確認します。

- 必要なデータ型とインデックス
- ORMとドライバーの対応状況
- チームの運用経験
- 利用するクラウドのマネージドサービス
- バックアップとリストアの要件

## Oracle・SQL Server：組織の環境から選ばれるデータベース

OracleとSQL Serverは、企業の業務システムで利用されることが多い商用データベースです。

個人開発で新しく選ぶというより、既存インフラ、サポート契約、製品連携、組織のセキュリティポリシーなどが選択理由になる場合があります。

個人学習ではPostgreSQLやMySQL系でSQLとトランザクションの基本を学び、担当するシステムに応じて商用データベース固有の機能を追加で学ぶ進め方もできます。

## Redisは目的を分けて考える

Redisは高速なインメモリデータストアです。RDBやAOFによる永続化もできますが、典型的なWebアプリケーションではPostgreSQLやMariaDBと役割を分け、次のような用途で併用します。

- 繰り返し参照する結果のキャッシュ
- セッションと有効期限の短い認証データ
- レート制限のカウンター
- メッセージやキューに近い処理
- 頻繁に変化する一時状態

```text
PostgreSQL → 元データを永続化する
Redis      → よく使うデータを高速に返す
```

キャッシュとして使う場合は、データが削除されたり古くなったりすることを前提に設計します。注文の原本などをRedisに保存するなら、単なるキャッシュ設定のまま使わず、永続化方式、レプリケーション、バックアップ、許容できるデータ損失の範囲を要件として決める必要があります。

## マネージドデータベースを使う理由

データベースを自分でインストールして運用する場合、次の作業も自分たちで担当します。

- バージョン更新
- バックアップとリストア
- ディスク容量とパフォーマンスの監視
- アクセス制御とネットワークセキュリティ
- 障害対応と高可用性

AWS RDS、Railway PostgreSQL、Neon、Supabaseなどのマネージドサービスは、この運用作業の一部をプラットフォームへ任せます。

ただし、スキーマ設計、クエリ性能、アクセス権限、コスト管理、リストア手順の確認まで自動で解決されるわけではありません。

## Supabase：PostgreSQLとバックエンド機能をまとめて使う

SupabaseのプロジェクトにはPostgreSQLデータベースが用意され、その上で複数のバックエンド機能を利用できます。

```text
Supabase
├─ PostgreSQL
├─ Authentication
├─ Storage
├─ Realtime
└─ Edge Functions
```

会員登録とファイルアップロードが必要な小規模サービスでは、複数の製品を個別に組み合わせずに開始できます。

```text
Next.js
  ↓
Supabase
```

ブラウザからData APIを使う場合は、公開対象のテーブルにRow Level Securityを設定します。たとえば投稿テーブルには、次のようなポリシーが必要です。

```text
公開された投稿は誰でも参照できる
ログインユーザーは自分の投稿を作成できる
作成者だけが自分の投稿を更新・削除できる
```

Supabaseはデータベース設計を代行するサービスではありません。PostgreSQLと、よく使うバックエンド機能をまとめて提供するBaaSです。

## Neon：PostgreSQLを中心にバックエンドを構成する

NeonはサーバーレスPostgreSQLプラットフォームです。コンピュートとストレージを分離し、オートスケーリング、データベースブランチ、ポイントインタイムリストアなどの機能を提供します。

```text
Next.jsまたはバックエンドAPI
  ↓
Prisma / Drizzle / SQLドライバー
  ↓
Neon PostgreSQL
```

NeonはPostgreSQLだけを提供するサービスとして始まりましたが、現在はManaged Better Authによる認証も提供しています。さらにObject Storage、Functions、AI Gatewayがベータとして追加され、PostgreSQLを中心としたバックエンドプラットフォームへ範囲を広げています。

Supabaseとの違いは、単純な「BaaSかデータベースか」ではなく、各機能の成熟度と設計の中心にあります。

| 観点 | Supabase | Neon |
|---|---|---|
| 中心 | PostgreSQLを基盤にしたBaaS | ブランチ可能なサーバーレスPostgreSQLを中心とするプラットフォーム |
| 認証 | Authを統合 | Managed Better Authを統合 |
| ファイルストレージ | Storageを統合 | S3互換Object Storageをベータ提供 |
| リアルタイム | Realtimeを統合 | PostgreSQLとアプリケーション側で要件に応じて構成 |
| サーバー側処理 | Edge Functionsを提供 | Functionsをベータ提供 |
| 選択例 | 成熟したバックエンド機能をまとめて使いたい | データと周辺機能をブランチ単位で分離したい |

認証、ストレージ、リアルタイムなどを実績のある統合機能としてまとめて使いたいならSupabaseが候補になります。PostgreSQLを中心に、プレビュー環境やテスト環境をデータと周辺機能ごとブランチ化したいならNeonが候補になります。

ただし、NeonのObject StorageとFunctionsなどは2026年8月30日時点でベータです。対応リージョンや運用上の制限もあるため、本番採用では最新の提供状況を公式ドキュメントで確認します。

## 構成ごとの選択例

### バックエンドの基本を学ぶ

```text
Spring Boot + PostgreSQL
```

REST API、SQL、ORM、トランザクション、認証を自分で実装しながら確認できます。

### MySQL系の既存プロジェクトを継続する

```text
Spring Boot + MariaDB
```

既存スキーマとチームの経験を活用できます。

### 会員制サービスを短期間で作る

```text
Next.js + Supabase
```

データベース、認証、ストレージを同じプラットフォームで構成できます。

### データベース以外の機能を個別に選ぶ

```text
Next.js + 認証サービス + オブジェクトストレージ + Neon
```

Neonをデータベースとして利用し、認証やファイル保存には別のサービスを選ぶ構成も引き続き可能です。各要素の成熟度、リージョン、料金を個別に判断したい場合に向いています。

### 参照負荷が大きいサービス

```text
バックエンド + PostgreSQL + Redis
```

PostgreSQLを元データの保存先とし、繰り返し参照するデータだけをRedisへキャッシュします。

## まとめ

新しい一般的なWebプロジェクトでは、PostgreSQLは有力な出発点です。ただし、MariaDBが古い選択肢になるわけではありません。

データベースを選ぶときは、機能の多さだけでなく、チームがデータの整合性、バックアップ、リストアまで継続して扱えるかを確認します。また、SupabaseとNeonは「どちらのデータベースが優れているか」ではなく、必要なバックエンド機能の成熟度、データやファイルのブランチ戦略、リージョン、運用方法を含めて比較すると分かりやすくなります。

## 参考資料

- [PostgreSQL公式サイト](https://www.postgresql.org/about/)
- [Supabase データベース](https://supabase.com/docs/guides/database/overview)
- [Supabase: Securing your data](https://supabase.com/docs/guides/database/secure-data)
- [Neon Documentation](https://neon.com/docs/introduction)
- [Neon Auth](https://neon.com/docs/neon-auth/overview)
- [Neon backend beta](https://neon.com/blog/neon-backend-is-beta)
- [Redis Documentation](https://redis.io/docs/latest/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
