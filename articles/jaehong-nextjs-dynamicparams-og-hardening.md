---
title: "Next.jsの多言語OGPで見直した、未対応パスとメタデータの継承"
emoji: "🛡️"
type: "tech"
topics: ["nextjs", "vercel", "typescript", "seo", "ogp"]
published: true
---

Next.jsのApp Routerで`generateStaticParams()`を使い、`/ko`、`/en`、`/ja`の3言語を静的生成していました。

そのため、一覧に含めていない`/xx`のようなパスは自動的に存在しなくなると思っていました。しかし、`generateStaticParams()`は「ビルド時に生成するパス」を指定する仕組みであり、それだけでは「それ以外のパスを拒否する許可リスト」にはなりません。

この違いに気付いたのは、[CC101日本語版](https://cc101jp.vercel.app/ja)のOpen Graph画像を言語別に生成するよう変更したときでした。

- [言語別OGPを追加したコミット](https://github.com/im-jaehong/cc101jp/commit/e5b32bddd16047d8b91bdde39708ec1e113eecd7)
- [`dynamicParams = false`を追加したコミット](https://github.com/im-jaehong/cc101jp/commit/76103e62eb529f159fed2fa6f4425440779573bc)

この記事では、原リポジトリの変更と公式仕様を基に、未対応パスを404にする設定と、言語別メタデータを確認する方法を整理します。2026年11月2日に公開サイトへGETリクエストを送り、現在の応答も再確認しました。当時のビルドログは今回再取得していないため、ビルドに関する説明は検証手順として紹介します。

## きっかけは言語別OGPの追加だった

このサイトでは、Markdownで管理するClaude Code入門ガイドを韓国語、英語、日本語で公開しています。

本文だけでなく、SNSやチャットへURLを共有したときの画像も各言語に合わせるため、次のルートを追加しました。

```text
/ko/opengraph-image
/en/opengraph-image
/ja/opengraph-image

/ko/twitter-image
/en/twitter-image
/ja/twitter-image
```

OG画像は`ImageResponse`で生成し、タイトル、説明、言語バッジ、サイトURLを言語ごとに切り替えています。対象言語は`generateStaticParams()`で指定しました。

```tsx
export function generateStaticParams() {
  return [
    { lang: 'ko' },
    { lang: 'en' },
    { lang: 'ja' },
  ]
}
```

このような構成では、ビルドログに次のような生成対象が表示されます。以下は読み方を示す模式図で、今回取得したビルドログではありません。

```text
● /[lang]
  ├ /ko
  ├ /en
  └ /ja

● /[lang]/opengraph-image
  ├ /ko/opengraph-image
  ├ /en/opengraph-image
  └ /ja/opengraph-image
```

ここだけを見ると、3言語以外のパスは存在しないように見えます。

## `generateStaticParams()`は許可リストではなかった

対象コミットの[`package.json`](https://github.com/im-jaehong/cc101jp/blob/76103e62eb529f159fed2fa6f4425440779573bc/package.json)ではNext.jsは**16.1.6**です。[`next.config.ts`](https://github.com/im-jaehong/cc101jp/blob/76103e62eb529f159fed2fa6f4425440779573bc/next.config.ts)には`cacheComponents`の指定がありません。この記事は、この構成を対象にしています。

Next.js 16でCache Componentsを有効にした構成では、以下の`dynamicParams`設定は利用できません。別のプロジェクトへ適用する際は、バージョンと`next.config`を確認してください。詳しくは[Route Segment ConfigのVersion History](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#version-history)を参照してください。

Next.jsの`dynamicParams`はデフォルトで`true`です。そのため、`generateStaticParams()`に含まれていない動的セグメントも、リクエスト時に処理できる状態が残ります。

Next.jsの[公式ドキュメント](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)にも、未指定のパスを実行時に静的生成させない場合は、次の設定を追加すると説明されています。

```tsx
export const dynamicParams = false
```

つまり、二つの設定は役割が異なります。

| 設定 | 役割 |
|---|---|
| `generateStaticParams()` | ビルド時に生成する動的パスを列挙する |
| `dynamicParams = false` | 列挙されていないパスを404にする |

私は前者だけで後者の動作も得られると誤解していました。

## 未対応パスで何が起き得たか

ページ側では対応言語を確認し、対象外なら最終的に`notFound()`を呼び出していました。しかし、ルート自体への到達は許されているため、404を返すまでにページの処理が実行されます。

OG画像側には、未知の値を日本語として扱うフォールバックもありました。

```tsx
const lang: Lang =
  langParam === 'ko' || langParam === 'en' ? langParam : 'ja'
```

対応していない`xx`を渡しても、ルートレベルで拒否されなければ日本語画像の生成処理へ進める実装です。

```text
/xx
/xx/opengraph-image
/xx/twitter-image
```

これは認証突破や情報漏えいのような脆弱性ではありません。しかし、公開する必要のない入力をサーバー側の処理対象にすると、次の余地が残ります。

- 任意の言語コードに対する不要な実行時処理
- 同じ内容を異なるURLで生成・キャッシュする可能性
- 想定外の入力を受け付けるルートの増加
- 問題調査時に、存在するURLと存在しないURLの境界が曖昧になること

そこで、対応する言語の範囲をルート設定にも明示する変更を確認しました。負荷削減や攻撃耐性を測定した結果としては扱いません。

## ページと画像ルートを明示的に閉じた

`[lang]`ページへ`dynamicParams = false`を追加しました。

```tsx
export const dynamicParams = false

export function generateStaticParams() {
  return [{ lang: 'ko' }, { lang: 'en' }, { lang: 'ja' }]
}
```

同じ設定を`opengraph-image.tsx`にも追加します。

```tsx
export const alt = 'CC101 — Claude Code Guide (cc101jp.vercel.app)'
export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'
export const dynamicParams = false
```

Twitter CardはOG画像の実装を再利用しています。

```tsx
export {
  default,
  alt,
  size,
  contentType,
  generateStaticParams,
} from './opengraph-image'

export const dynamicParams = false
```

`dynamicParams`はルートの設定として静的に解析される必要があるため、`twitter-image.tsx`でも値を直接宣言しました。

ページ内の`notFound()`による検証も削除していません。ルート設定で未定義パスを閉じることと、アプリケーション側で入力を検証することは役割が異なるため、二段階で残しています。

## ビルド成果物を確認する手順

コードを変更したら、対象プロジェクトで静的検査とプロダクションビルドを実行します。このリポジトリでは次のコマンドに対応するスクリプトがあります。

```bash
pnpm lint
pnpm exec tsc --noEmit
pnpm build
```

ビルドログに3言語だけが表示されても、未指定パスのフォールバック動作までは分かりません。補助的な確認には`.next/prerender-manifest.json`が使えます。これはNext.js内部の成果物であり、バージョンによって構造やルート名が変わり得ます。

PowerShellでは次のように確認できます。

```powershell
$manifest = Get-Content .next/prerender-manifest.json -Raw |
  ConvertFrom-Json

$manifest.dynamicRoutes.PSObject.Properties |
  ForEach-Object {
    [pscustomobject]@{
      Route    = $_.Name
      Fallback = $_.Value.fallback
    }
  }
```

対象の動的ルートについて`fallback`が`false`になっているかを調べ、実際のHTTP応答と照合します。manifestだけで公開環境の挙動を確定せず、対応パスと未対応パスの両方をGETで確認することが大切です。

## Vercelの実環境でも404を確認した

2026年11月2日に、公開サイトの次のパスへGETリクエストを送りました。PowerShellでは`curl`が別コマンドの別名になっている環境もあるため、以下は`curl.exe`を使う例です。macOSやLinuxでは`curl`へ読み替えます。

```powershell
curl.exe -sS -o NUL -w '%{http_code}' https://cc101jp.vercel.app/xx
curl.exe -sS -o NUL -w '%{http_code}' https://cc101jp.vercel.app/xx/opengraph-image
curl.exe -sS -o NUL -w '%{http_code}' https://cc101jp.vercel.app/xx/twitter-image
```

確認結果は次のとおりです。

| パス | GETのHTTPステータス |
|---|---:|
| `/ko` | 200 |
| `/en` | 200 |
| `/ja` | 200 |
| `/xx` | 404 |
| `/xx/opengraph-image` | 404 |
| `/xx/twitter-image` | 404 |

この結果は、確認したURLがその時点で返した応答を示します。画像生成処理の実行有無や、任意の未対応URLでの動作まで証明するものではありません。処理の実行有無を調べるにはサーバーログや計測が必要です。

## OGP画像だけでなくメタデータも再確認した

未対応パスを閉じたあと、生成された画像だけでなく、各言語ページのHTMLに出力されるOpen Graphメタデータも確認しました。そこで、ルートの`layout.tsx`に設定していた次の値が、最終HTMLから消えていることに気付きました。

```text
og:site_name
og:locale
og:type
```

原因はNext.jsのメタデータのマージ方法でした。複数セグメントのMetadataは**浅くマージ**され、`openGraph`のようなネストしたフィールドで同じキーが定義されると、後から評価されたオブジェクトへ置き換わります。

ルートには共通値がありました。

```tsx
export const metadata: Metadata = {
  openGraph: {
    siteName: 'CC101',
    locale: 'ko_KR',
    type: 'website',
  },
}
```

一方、`[lang]/page.tsx`の`generateMetadata()`は、言語別の`openGraph`オブジェクトを返していました。

```tsx
return {
  openGraph: {
    url: 'https://cc101jp.vercel.app/ja',
    title: 'CC101 — Claude Code 日本語入門ガイド',
    description: '...',
  },
}
```

この場合、`url`、`title`、`description`だけがルートの`openGraph`へ追加されるのではありません。ページ側の`openGraph`オブジェクトが優先されるため、そこに書かなかった共通値が失われます。

そこで、言語別メタデータへ必要な値を明示しました。

```tsx
openGraph: {
  url: 'https://cc101jp.vercel.app/ja',
  siteName: 'CC101',
  locale: 'ja_JP',
  type: 'website',
  title: 'CC101 — Claude Code 日本語入門ガイド',
  description: '...',
}
```

同じ修正を韓国語と英語にも適用しています。

| パス | `og:locale` |
|---|---|
| `/ko` | `ko_KR` |
| `/en` | `en_US` |
| `/ja` | `ja_JP` |

2026年11月2日に公開ページのHTMLを再確認したところ、3言語すべてで`og:site_name=CC101`、表のとおりの`og:locale`、`og:type=website`、各言語の`opengraph-image`を指す`og:image`が出力されていました。これはHTMLの確認結果であり、SNS側の画像キャッシュや表示まで検証したという意味ではありません。

- [Open Graphメタデータを修正したコミット](https://github.com/im-jaehong/cc101jp/commit/eff5daeeaaa994e715760ba8ec3d2c900e0e07b9)
- [Next.js: MetadataのMerging](https://nextjs.org/docs/app/api-reference/functions/generate-metadata#merging)

今回の見落としから、OGPは画像URLだけでなく、最終的に生成されたHTMLの`<meta>`タグを単位として確認する必要があると分かりました。

## この設定で解決しないこと

`dynamicParams = false`は、許可していない動的セグメントのレンダリングを防ぐ設定です。サイト全体のセキュリティ対策ではありません。

次の対策を置き換えるものではない点には注意が必要です。

- DDoS対策やレート制限
- 認証・認可
- APIへ渡す入力値の検証
- クエリパラメーターの検証
- 依存パッケージの脆弱性対応
- Vercel側の利用量・ログ監視

また、Vercelの請求や関数実行回数がどれだけ減るかは、デプロイ構成やキャッシュ状態にも左右されます。今回、コスト削減の効果は測定していません。

今回確認したのは、対象コミットの設定と、公開サイトで未対応の3パスが404を返すことです。設定がない場合との実行時間や関数呼び出し回数の比較は行っていません。

## 今回学んだこと

### 1. 生成対象と受付可能範囲は別に確認する

ビルドログに表示されるルートだけでなく、一覧にない値を渡したときの動作まで確認する必要があります。

### 2. 動的セグメントは入力値として扱う

`[lang]`はファイル名の一部に見えますが、外部から自由に渡せる入力値です。対応値が決まっているなら、アプリケーションの判定だけでなくルート設定でも範囲を限定した方が明確です。

### 3. 特殊なメタデータファイルもルートになる

`opengraph-image.tsx`と`twitter-image.tsx`は画像ファイルのように見えますが、コードで生成する場合はNext.jsの特殊なRoute Handlerとして動作します。[公式ドキュメント](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/opengraph-image)で、通常ページとは別に動作を確認する必要があります。

### 4. メタデータは最終HTMLで確認する

設定ファイルだけを見ると、ルートで指定した値が子ページにも残るように見えます。しかし、ネストしたMetadataは浅くマージされます。各セグメントの設定だけでなく、ビルド後のHTMLとデプロイ後の`<meta>`タグを確認する方が確実です。

### 5. セキュリティという言葉を広げ過ぎない

不要な実行経路を閉じることには意味があります。一方で、それを重大な脆弱性や攻撃の完全な防止と表現すると、実施した対策以上の効果を主張することになります。

今回は「未対応パスを404として扱う設定を明示した」と説明するのが適切だと判断しました。サーバー負荷や攻撃耐性への効果は、今回の確認だけでは定量評価できません。

## まとめ

`generateStaticParams()`へ3言語だけを指定しても、それだけでは他の言語パスを拒否したことにはなりませんでした。

対象コミットでは、ページ、Open Graph画像、Twitter画像の各ルートに`dynamicParams = false`が宣言されています。別のサイトへ適用する場合も、次の順で確認すると設定と実際の応答を結び付けられます。

1. ESLint、TypeScript、プロダクションビルド
2. `prerender-manifest`のfallback
3. 公開環境でのGET応答と最終HTMLのメタデータ

その後、言語別ページの`openGraph`がルート設定を置き換え、`siteName`、`locale`、`type`が失われていた問題も修正しました。ルーティングとメタデータは別の仕組みですが、どちらも「設定を書いたか」ではなく、ビルド成果物と実環境で意図した結果になったかを確認する必要があります。

小さな設定でも、フレームワークのデフォルト値と適用条件を確認する必要があります。2026年11月2日の再確認では、未対応パスの404と、対応する3言語のOpen Graphメタデータを確認できました。検証した範囲を明示することで、設定の効果を過大に説明せずに改善を共有できます。

## 参考資料

- [CC101日本語版](https://cc101jp.vercel.app/ja)
- [GitHub: im-jaehong/cc101jp](https://github.com/im-jaehong/cc101jp)
- [Next.js: generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)
- [Next.js: Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)
- [Next.js: opengraph-image and twitter-image](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/opengraph-image)
- [Next.js: generateMetadata](https://nextjs.org/docs/app/api-reference/functions/generate-metadata)
