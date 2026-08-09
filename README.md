# zenn-content

Zennで公開する日本語の技術記事を管理するRepositoryです。

## 運用方針

- `articles/`では日本語のZenn記事だけを管理します。
- 新しい記事は原則として`published: false`で作成します。
- 内容、技術的な正確性、公開可能な情報かどうかを人が確認してから公開します。
- GitへのCommit・Pushと`published: true`への変更は、公開を決めた後に行います。

## Preview

```powershell
npx zenn preview
```

- [Zenn CLIの使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
