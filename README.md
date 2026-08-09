# zenn-content

Zennで公開する日本語の技術記事を管理するRepositoryです。

## 運用方針

- `articles/`では日本語のZenn記事だけを管理します。
- 新しい記事は執筆・確認中は`published: false`で管理します。
- 内容、技術的な正確性、公開可能な情報かどうかを人が確認します。
- 検証済みの記事をGitHubへPushするときは、原則として`published: true`にして公開します。
- 下書きのままPushする必要がある場合だけ`published: false`を維持します。
- 複数の新規記事を公開する場合は、Zennの投稿数制限を避けるため`published_at`で公開日時を分散します。

## Preview

```powershell
npx zenn preview
```

- [Zenn CLIの使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
