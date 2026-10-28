---
title: "Vercel・Railway・AWSの違いからWebアプリのデプロイ先を考える"
emoji: "🚀"
type: "tech"
topics: ["aws", "vercel", "railway", "deploy"]
published: true
---

Webアプリケーションのデプロイ先を調べると、Vercel、Railway、AWSといったサービスが候補に挙がります。

ただし、この3つは同じ範囲を担当するサービスではありません。単純に新旧や優劣で比較するより、**開発者がどこまでインフラを管理するか**という観点で整理すると選びやすくなります。

この記事では一般的な機能表だけで終わらせず、Reactの静的フロントエンド、Spring Boot API、PostgreSQLを使う個人開発を例に、要件からデプロイ先を選ぶ過程を整理します。比較は2026年10月29日に確認した公式ドキュメントに基づくもので、同一アプリケーションを3サービスへ配置した性能比較ではありません。

## デプロイで必要になるもの

Webサイトを公開するには、ソースコードをサーバーへ置くだけでなく、次のような要素が必要です。

- アプリケーションのビルド
- サーバー、または静的ファイルの保存先
- ドメインとDNS
- HTTPS証明書
- CDN
- CI/CD
- ログとモニタリング

Reactの静的サイトをAWSで構成する場合、デプロイとユーザーからのアクセスは次のように分けて考えられます。

```text
デプロイ: GitHub → CI/CD → Reactの静的ビルド → S3
配信:     ユーザー → CloudFront（HTTPS / ACM）→ S3
DNS:      独自ドメイン → Route 53でCloudFrontへ名前解決
```

Route 53はユーザーのリクエストが通過するサーバーではなく、ドメイン名からCloudFrontの配信先を解決するDNSです。また、CloudFrontはS3に配置したファイルを配信するため、更新時にはキャッシュの有効期限や、必要に応じたキャッシュ無効化（Invalidation）も考慮します。

構成を細かく選べる一方で、それぞれのサービスを接続して運用する必要があります。VercelやRailwayのような管理型プラットフォームは、この作業の一部を標準機能として引き受けます。

## Vercel：フロントエンドを素早く公開する

Vercelでは、Gitリポジトリを連携すると、プッシュを契機にビルドとデプロイが実行されます。

```text
GitHub
  ↓ git push
Vercel
  ├─ ビルド
  ├─ デプロイ
  ├─ HTTPS
  ├─ CDN
  └─ プレビューURL
```

Reactの静的サイトだけでなく、Next.jsアプリケーションやVercel Functionsもデプロイできます。Pull Requestごとにプレビュー環境を確認できるため、画面変更のレビューにも向いています。

候補になりやすいのは次のようなケースです。

- ポートフォリオや技術ブログ
- React、Next.jsを使ったWebアプリ
- 早い段階で確認したいMVP
- フロントエンドが中心のサービス
- 短時間で完了するサーバー処理

一方、長時間動くジョブ、特殊なネットワーク、OSレベルの制御が必要なバックエンドでは、別の実行基盤を検討した方が自然です。

### Hobby Planを利用するときの注意

2026年10月時点の公式ドキュメントでは、Hobby Planは個人・非商用のプロジェクト向けです。業務や商用サービスで利用する場合は、最新の利用条件とPro Planを確認する必要があります。

無料枠の有無だけでなく、Vercel Functionsの実行時間や転送量など、アプリケーションに関係する制限もデプロイ前に確認します。

## Railway：バックエンドとデータベースを簡単に配置する

Railwayは、Spring BootやNode.jsなどのバックエンドとPostgreSQLを、比較的少ない設定でデプロイできるプラットフォームです。

```text
フロントエンド → Vercel
バックエンド   → Railway / Spring Boot
データベース   → Railway / PostgreSQL
```

Spring Bootは、GitHubリポジトリ、Railway CLI、Dockerfileなどからデプロイできます。EC2へSSH接続してJavaやプロセス管理ツールを設定する作業を減らせるため、まずAPIを外部から確認したいときに便利です。

次のようなケースで検討できます。

- 独立したバックエンドAPIが必要
- Spring Bootを短時間で公開したい
- アプリケーションとデータベースを同じプロジェクト内で接続したい
- インフラの構築より機能検証を優先したい

ただし、実行方法や料金はRailwayの仕様に依存します。ネットワークやセキュリティを細かく設計する必要がある場合は、要件を満たせるか先に確認します。

## AWS：必要なインフラを組み合わせる

AWSは、VercelやRailwayと同じ範囲だけを扱うサービスではありません。コンピュート、ストレージ、データベース、ネットワーク、セキュリティなどを組み合わせるクラウドプラットフォームです。

ReactとSpring Bootを使う場合、次のような構成が考えられます。

```text
ユーザー
  ├─ CloudFront → S3（Reactの静的ビルド）
  └─ Application Load Balancer → ECS（Spring Bootコンテナ）
                                      ↓
                                  RDS PostgreSQL
```

学習用であれば、最初はEC2上でDockerコンテナを1つ動かす構成も選べます。要件の増加に合わせてロードバランサーやECSへ移行する方法もあります。

AWSが候補になりやすいのは次のような場合です。

- ネットワークやセキュリティを細かく制御したい
- 複数のバックエンドや非同期ジョブを運用する
- コンテナ、ロードバランサー、Auto Scalingが必要
- 組織の権限管理や監査要件がある
- クラウドインフラ自体を学習したい

自由度が高い分、コスト、セキュリティ、モニタリング、障害対応まで、設計する範囲も広くなります。

## 3つの選択肢を比較する

| 観点 | Vercel | Railway | AWS |
|---|---|---|---|
| 主な強み | フロントエンド・Next.js | バックエンド・DBの簡単なデプロイ | 幅広いインフラ |
| 導入の難易度 | 低い | 低い〜中程度 | 中程度〜高い |
| サーバー制御 | 限定的 | 限定的 | 広い |
| 運用負担 | 少ない | 少ない〜中程度 | 構成による |
| 向いている例 | ポートフォリオ、MVP | API、小規模サービス | 複雑なサービス、企業システム |

同じプロジェクトで複数のサービスを組み合わせることもできます。

この表は、上で示した構成例に対する定性的な比較です。AWSでもAmplify Hostingなどの管理型サービスを使えば運用範囲を減らせるため、「AWSは必ず難しい」という意味ではありません。また、RailwayのPostgreSQLは、RDSと同じ運用保証を持つサービスとは限りません。バックアップ、復旧、更新、可用性の責任範囲は、採用するサービスとプランごとに確認します。

## 要件から候補を絞った判断例

ここでは、次の要件を持つ個人開発を想定します。

- Reactは静的にビルドできるSPA
- Spring Boot APIは常時リクエストを受け付ける
- PostgreSQLが必要
- まずは機能を外部から確認できる状態を優先する
- 個人・非商用で始めるが、料金と利用条件は公開前に再確認する
- VPCや複数AZなどの詳細なインフラ設計は現段階の要件に含めない

この要件では、最初の候補を次のように絞れます。

| 対象 | 選択 | 判断理由 |
|---|---|---|
| React | Vercel | Gitリポジトリ連携とプレビュー環境を利用し、画面変更を確認しやすくする |
| Spring Boot | Railway | GitHubまたはDockerfileからAPIを配置し、サーバー構築より機能検証を優先する |
| PostgreSQL | Railway | APIと同じプロジェクト内で接続情報を管理する |
| AWS | 今回は保留 | 現段階では詳細なネットワーク制御より、少ない構成要素で公開することを優先する |

AWSを選ばない理由は、AWSが劣っているからではありません。今回の要件では、VPC、ロードバランサー、ECS、RDSなどを個別に設計・運用する利点をまだ使い切れないと判断したためです。反対に、ネットワーク分離、権限管理、可用性、監査要件が加われば、この判断は変わります。

### この構成で残る確認事項

管理をプラットフォームへ任せても、アプリケーション側の確認は必要です。

- ブラウザーからRailwayのAPIを直接呼ぶ場合のCORS設定。本番フロントエンドのOriginを明示し、プレビュー環境を許可する範囲も決める
- SPAの画面URLへ直接アクセスしたときのフォールバック設定。トップページだけでなく、画面の再読み込みでも表示できるか確認する
- フロントエンドへ公開してよい環境変数と、サーバーだけで保持する秘密情報の分離
- Railway内のAPIからPostgreSQLへ接続するプライベート接続と、バックアップの設定・復元手順。外部接続が不要ならDBの公開アクセスを追加しない
- デプロイ失敗時に確認するログの場所
- 実際のレスポンスタイムと月額利用量
- 無料枠やプラン条件を超えた場合の移行方針

この段階では性能や料金を実測していないため、「最適な構成」とは断定しません。まず候補を絞り、実際にデプロイした後でログ、応答時間、利用量、復元手順を確認して判断を更新する前提です。

## 構成例

### 静的なポートフォリオ

```text
React + Vercel
```

APIもデータベースも不要であれば、構成を増やさない方が管理しやすくなります。

### Spring Bootを学ぶ個人開発

```text
React       → Vercel
Spring Boot → Railway
PostgreSQL  → Railway
```

バックエンドは自分で実装しつつ、サーバー構築の負担を抑える構成です。

### インフラまで扱うチーム開発

```text
Reactの静的ビルド     → S3 / CloudFront
Spring Bootコンテナ   → ECS
PostgreSQL  → RDS
```

コンテナ、ネットワーク、CI/CD、モニタリングまで学習対象にできます。

## まとめ

最近の流れは、AWSからVercelへ単純に置き換わったということではありません。アプリケーションの要件に応じて、管理をプラットフォームへ任せる範囲を選べるようになったと考える方が適切です。

デプロイ先を決めるときは、次の順で確認します。

1. 静的なフロントエンドか、常時動くバックエンドか
2. データベースや長時間ジョブが必要か
3. ネットワークとサーバーをどこまで制御したいか
4. 想定トラフィックと障害対応の水準
5. 本番運用時の料金と制限

小さなアプリでは単純な構成が運用上のメリットになります。複雑な要件がある場合は、最初の手軽さだけでなく、必要な制御範囲を基準に選ぶことが大切です。

## 参考資料

- [VercelのPlan](https://vercel.com/docs/plans)
- [Vercel Hobby Plan](https://vercel.com/docs/plans/hobby)
- [Vercelの制限](https://vercel.com/docs/limits)
- [Vercel: Vite](https://vercel.com/docs/frameworks/frontend/vite)
- [Railway: Deploy a Spring Boot App](https://docs.railway.com/guides/spring-boot)
- [Railway PostgreSQL](https://docs.railway.com/databases/postgresql)
- [Railway Pricing FAQs](https://docs.railway.com/pricing/faqs)
- [AWSの製品一覧](https://aws.amazon.com/jp/products/)
- [AWS Amplify Hosting](https://docs.aws.amazon.com/amplify/latest/userguide/welcome.html)
- [AWS: CloudFrontを使用した安全な静的Webサイト](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/getting-started-secure-static-website-cloudformation-template.html)
- [AWS: Route 53からCloudFrontへトラフィックをルーティングする](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-cloudfront-distribution.html)
