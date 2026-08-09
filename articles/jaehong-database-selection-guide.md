---
title: "PostgreSQL・MariaDB・Redis・Supabase・Neonの使い分けを整理する"
emoji: "🗄️"
type: "tech"
topics: ["postgresql", "mariadb", "redis", "supabase", "neon"]
published: false
---

Webアプリケーションでデータベースを選ぶとき、「最近よく使われているもの」だけを基準にすると判断が難しくなります。

データ構造、Transaction、Teamの経験、運用方法によって適切な選択肢は変わります。今回はPostgreSQL、MySQL・MariaDB、Redisの役割と、Supabase、Neonの違いを整理します。

## 最初にリレーショナルデータベースを確認する

ユーザー、投稿、注文などのデータは、互いに関係を持ちます。

```text
users
└─ id

posts
├─ id
└─ user_id → users.id
```

PostgreSQL、MySQL、MariaDB、Oracle、SQL Serverは代表的なリレーショナルデータベースです。データをTable、Row、Columnで構造化し、SQLを使って操作します。

リレーショナルデータベースには、データの整合性を守るための仕組みがあります。Foreign Keyを使えば存在しないユーザーと投稿が関連付けられることを防げます。また、Transactionによって複数の更新を1つの処理として扱えます。

```text
注文を作成
+ 在庫を減らす
+ 決済記録を保存
= 1つのTransaction
```

途中でErrorが発生した場合は全体をRollbackし、不整合な状態が残らないように設計できます。

## PostgreSQL：新規プロジェクトの有力な選択肢

PostgreSQLはOpen SourceのObject-Relational Databaseです。一般的なリレーショナルデータに加えて、JSON/JSONB、Array、全文検索、複数のIndex、Extensionを利用できます。

次のような要件で候補になります。

- 新しいWebサービスで汎用的なDatabaseが必要
- 複雑なQueryとTransactionがある
- JSONとリレーショナルデータを一緒に扱う
- 地理情報やVectorなどのExtensionを利用する
- SupabaseやNeonを検討している

機能が多いからといって、最初からすべて使う必要はありません。一般的なCRUDアプリケーションでも標準的なリレーショナルデータベースとして利用できます。

## MySQL・MariaDB：既存の知識と資産を生かす

MySQLとMariaDBは、Webサービスで長く利用されてきました。Hosting、Library、運用Toolが多く、既存Systemで扱う機会もあります。

以前のチーム開発で利用した構成は次のようなものでした。

```text
React
  ↓
Spring Boot
  ↓
MariaDB
```

TeamにMariaDBの運用経験があり、既存SchemaもMySQL系であれば、流行だけを理由にPostgreSQLへ変更する必要はありません。

Databaseの移行では、SQL Syntaxだけでなく、Data Migration、Backup、Monitoring、障害対応も変わります。新しいProjectで比較するときは、次の項目を確認します。

- 必要なData TypeとIndex
- ORMとDriverの対応状況
- Teamの運用経験
- 利用するCloudのManaged Service
- BackupとRestoreの要件

## Oracle・SQL Server：組織の環境から選ばれるDatabase

OracleとSQL Serverは、企業の業務Systemで利用されることが多いCommercial Databaseです。

個人開発で新しく選ぶというより、既存Infrastructure、Support Contract、製品連携、組織のSecurity Policyなどが選択理由になる場合があります。

個人学習ではPostgreSQLやMySQL系でSQLとTransactionの基本を学び、担当するSystemに応じてCommercial Database固有の機能を追加で学ぶ進め方もできます。

## RedisはPostgreSQLの代わりではない

Redisは高速なIn-Memory Data Storeです。PostgreSQLやMariaDBと同じ目的で置き換えるのではなく、次のような用途で併用します。

- 繰り返し参照する結果のCache
- Sessionと有効期限の短い認証Data
- Rate LimitのCounter
- MessageやQueueに近い処理
- 頻繁に変化する一時状態

```text
PostgreSQL → 元Dataを永続化する
Redis      → よく使うDataを高速に返す
```

Cacheは削除されたり古くなったりすることを前提に設計します。注文の原本など、失ってはいけないデータをRedisだけに保存する構成は避けます。

## Managed Databaseを使う理由

Databaseを自分でInstallして運用する場合、次の作業も自分たちで担当します。

- Version Update
- BackupとRestore
- Disk容量とPerformanceの監視
- Access ControlとNetwork Security
- 障害対応とHigh Availability

AWS RDS、Railway PostgreSQL、Neon、SupabaseなどのManaged Serviceは、この運用作業の一部をPlatformへ任せます。

ただし、Schema設計、Query Performance、Access権限、Cost管理、Restore手順の確認まで自動で解決されるわけではありません。

## Supabase：PostgreSQLとBackend機能をまとめて使う

SupabaseのProjectにはPostgreSQL Databaseが用意され、その上で複数のBackend機能を利用できます。

```text
Supabase
├─ PostgreSQL
├─ Authentication
├─ Storage
├─ Realtime
└─ Edge Functions
```

会員登録とFile Uploadが必要な小規模Serviceでは、複数の製品を個別に組み合わせずに開始できます。

```text
Next.js
  ↓
Supabase
```

ブラウザからData APIを使う場合は、公開対象のTableにRow Level Securityを設定します。たとえば投稿Tableには、次のようなPolicyが必要です。

```text
公開された投稿は誰でも参照できる
ログインUserは自分の投稿を作成できる
作成者だけが自分の投稿を更新・削除できる
```

SupabaseはDatabase設計を代行するServiceではありません。PostgreSQLと、よく使うBackend機能をまとめて提供するBaaSです。

## Neon：Serverless PostgreSQLに集中する

NeonはServerless PostgreSQL Platformです。ComputeとStorageを分離し、Autoscaling、Database Branching、Instant Restoreなどの機能を提供します。

```text
Next.jsまたはBackend API
  ↓
Prisma / Drizzle / SQL Driver
  ↓
Neon PostgreSQL
```

Supabaseとの違いは、主に提供範囲です。

| 観点 | Supabase | Neon |
|---|---|---|
| 中心 | PostgreSQLを基盤にしたBaaS | Serverless PostgreSQL |
| Auth | 統合機能を提供 | 必要に応じて別Serviceを選択 |
| File Storage | 統合機能を提供 | 別Serviceを組み合わせる |
| Realtime | 統合機能を提供 | 要件に合わせて構成する |
| 選択例 | Backend機能をまとめたい | PostgreSQLを中心に自由に組みたい |

Auth、Storage、Databaseを1つのPlatformで使いたいならSupabase、PostgreSQLだけをManaged Serviceとして利用し、他の機能を個別に選びたいならNeonが候補になります。

## 構成ごとの選択例

### Backendの基本を学ぶ

```text
Spring Boot + PostgreSQL
```

REST API、SQL、ORM、Transaction、認証を自分で実装しながら確認できます。

### MySQL系の既存Projectを継続する

```text
Spring Boot + MariaDB
```

既存SchemaとTeamの経験を活用できます。

### 会員制Serviceを短期間で作る

```text
Next.js + Supabase
```

Database、Auth、Storageを同じPlatformで構成できます。

### PostgreSQL以外を個別に選ぶ

```text
Next.js + Auth Service + Object Storage + Neon
```

各要素を独立して選択したい場合の構成です。

### 参照負荷が大きいService

```text
Backend + PostgreSQL + Redis
```

PostgreSQLを元Dataの保存先とし、繰り返し参照するDataだけをRedisへCacheします。

## まとめ

新しい一般的なWebプロジェクトでは、PostgreSQLは有力な出発点です。ただし、MariaDBが古い選択肢になるわけではありません。

Databaseを選ぶときは、機能の多さだけでなく、Teamがデータの整合性、Backup、Restoreまで継続して扱えるかを確認します。また、SupabaseとNeonは「どちらのDatabaseが優れているか」ではなく、Database以外のBackend機能をどこまで同じPlatformへ任せるかで比較すると分かりやすくなります。

## 参考資料

- [PostgreSQL公式サイト](https://www.postgresql.org/about/)
- [Supabase Database](https://supabase.com/docs/guides/database/overview)
- [Supabase: Securing your data](https://supabase.com/docs/guides/database/secure-data)
- [Neon Documentation](https://neon.com/docs/introduction)
- [Redis Documentation](https://redis.io/docs/latest/)
