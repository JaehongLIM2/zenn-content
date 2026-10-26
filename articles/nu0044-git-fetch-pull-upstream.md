---
title: "git fetchとgit pullの違いをorigin/main・upstreamから理解する"
emoji: "🔄"
type: "tech"
topics: ["git", "github", "初心者", "バージョン管理"]
published: true
---

GitHubの変更をローカルへ反映するとき、`git pull`だけを覚えていると、内部で何が変わったのか分かりにくくなることがあります。

```text
main
origin
origin/main
upstream
fetch
pull
```

これらはすべてリモート操作に関係しますが、役割は異なります。

この記事では、ローカルブランチ、リモート追跡ブランチ（remote-tracking branch）、upstreamの関係から`fetch`と`pull`の違いを整理します。

## 最初に4つの名前を分ける

```text
main          ローカルブランチ
origin        リモートURLに付けたローカル上の名前
origin/main   最後に取得したリモートのmainを示すローカル参照
upstream      ローカルブランチが既定の相手とするリモートブランチ
```

特に重要なのは、`origin/main`もローカルリポジトリ内にある参照だという点です。

GitHub上の`main`をリアルタイムで直接見ているわけではありません。`git fetch`した時点の状態をローカルに記録しています。

## originはリモートリポジトリの別名

リモートを登録する例です。

```bash
git remote add origin <repository-url>
git remote -v
```

`origin`は慣例的な名前であり、GitHubの固定キーワードではありません。1つのローカルリポジトリに複数のリモートを登録することもできます。

```text
origin   → 自分が通常pushするリポジトリ
upstream → フォーク元のリポジトリ
```

ここで使うリモート名の`upstream`と、ブランチの追跡関係を表すupstreamブランチは文脈が異なるため注意が必要です。

## git fetchが変更するもの

```bash
git fetch origin
```

`fetch`はリモートからコミットや参照を取得し、設定されたリモート追跡ブランチを更新します。

```text
GitHub上のmain
    ↓ git fetch origin
origin/main を更新

ローカル main とワーキングツリーはそのまま
```

取得後は、統合前に差分を確認できます。

```bash
git log --all --graph --decorate --oneline
git diff main..origin/main
```

この「取得」と「統合」を分けられることがfetchの利点です。

## git pullはfetch後に統合する

```bash
git pull
```

`pull`はリモートから取得した後、現在のブランチへ変更を統合します。

```text
git pull = git fetch + 統合
```

ただし統合方法は必ずmergeとは限りません。設定やオプションによってmerge、rebase、fast-forward onlyを選べます。

```bash
git pull --rebase
git pull --ff-only
```

そのため、`git pull`を常に「fetch + merge」とだけ説明すると、リポジトリごとの設定差を見落とします。

## 慣れないうちはfetchと統合を分ける

リモートの状態を理解しながら進めたい場合は、次の順序が分かりやすくなります。

```bash
git status -sb
git fetch origin
git log --all --graph --decorate --oneline -10
git merge --ff-only origin/main
```

`--ff-only`で失敗した場合は、ローカルとリモートが分岐しています。すぐに別のコマンドを試すのではなく、グラフを確認してmergeまたはrebaseを選びます。

### 検証：分岐したブランチではff-onlyが停止する

動作を確認するため、1つの検証用リモートリポジトリを2つの作業ディレクトリへcloneしました。一方で「リモート側の変更」をpushし、もう一方には「ローカル側の変更」をcommitしてから`fetch`しています。検証環境はGit 2.49.0 for Windowsです。

```console
$ git status -sb
## main...origin/main [ahead 1, behind 1]
```

ローカルとリモートに1件ずつ固有のコミットがあることを、次のコマンドでも確認できました。

```console
$ git log --left-right --format="%m %s" 'HEAD...@{upstream}'
< ローカル側の変更
> リモート側の変更
```

この状態ではfast-forwardできないため、`--ff-only`は履歴を自動で統合せず停止します。

```console
$ git merge --ff-only origin/main
fatal: Not possible to fast-forward, aborting.
```

この失敗は異常ではなく、意図しないマージコミットやrebaseを避けるための確認結果です。今回は検証だけなので統合しませんでした。実際の作業では、共有ブランチの運用方針とコミット内容を確認してからmergeまたはrebaseを選びます。

## upstreamブランチとは

ローカルブランチには、追跡先となるupstreamブランチを設定できます。これは引数を省略した`git pull`や、ahead/behindの比較に使われます。upstreamは別のローカルブランチにも設定できますが、ここではリモート追跡ブランチを使う一般的な構成を説明します。

新しい作業ブランチを初めてpushする例です。

```bash
git push -u origin feat-docs
```

`-u`は`--set-upstream`の短縮形です。ローカル`feat-docs`と`origin/feat-docs`の関係を設定します。

同名ブランチを追跡し、既定の`push.default=simple`を使う一般的な構成では、以降はリモート名とブランチ名を省略できます。

```bash
git push
git pull
```

ただし、push先はupstreamだけで決まるわけではありません。`push.default`、`remote.pushDefault`、`branch.<name>.pushRemote`などの設定によって異なります。pull先とpush先を分ける運用では、それぞれの設定を確認します。

確認には次のコマンドを使います。

```bash
git branch -vv
git status -sb
```

## ローカルブランチとorigin/mainが分岐した状態

```text
          C---D  main
         /
A---B---M
         \
          E---F  origin/main
```

ローカルにもリモートにも異なるコミットがある状態です。

upstreamとの差は次のように確認できます。

```bash
git log --left-right --graph --oneline 'HEAD...@{upstream}'
```

`<`と`>`でそれぞれの側にだけ存在するコミットを確認できます。

この状態でどちらの履歴を優先するかはチームの運用方針によります。`merge`、`rebase`、変更の退避などを判断する前に、未コミットの作業とコミットグラフを確認します。

## clone時には何が設定されるのか

```bash
git clone <repository-url>
```

一般的なcloneでは次の処理が行われます。

- リポジトリのコミットとファイルを取得
- `.git`ディレクトリを作成
- リモートを`origin`として登録
- デフォルトブランチをチェックアウト
- ローカルデフォルトブランチのupstreamを設定

そのためclone直後のデフォルトブランチでは、`git pull`や`git push`の相手がすでに設定されていることが多くなります。

```bash
git remote -v
git branch -vv
```

## Pull Requestのbaseとcompare

作業ブランチをGitHubへpushした後、Pull Requestではブランチの方向を選びます。

```text
base    : 変更を受け取るブランチ（例: main）
compare : 変更を提案するブランチ（例: feat-docs）
```

```bash
git switch -c feat-docs

# 変更とコミット
git push -u origin feat-docs
```

GitHubでは`base: main`、`compare: feat-docs`を選びます。

Pull Requestは変更を統合するだけでなく、変更理由、差分、レビュー、テスト結果を共有する単位です。

## 問題が起きたときの確認順序

### 1. 現在のブランチと変更状態

```bash
git status -sb
```

### 2. upstream設定

```bash
git branch -vv
```

### 3. リモートURL

```bash
git remote -v
```

### 4. 最新のリモート状態を取得

```bash
git fetch origin
```

### 5. コミットグラフ

```bash
git log --all --graph --decorate --oneline -10
```

この順序なら、認証の問題、upstreamの設定漏れ、単純なリモート更新、ブランチの分岐を分けて考えられます。

## まとめ

`fetch`と`pull`の違いは「取得するかどうか」ではありません。どちらもリモートから情報を取得しますが、`pull`はその後の統合まで進めます。

```text
fetch : origin/mainなどを更新し、確認する余地を残す
pull  : fetch後に現在のブランチへ統合する
```

`origin/main`がローカルにあるリモート追跡ブランチだと理解すると、fetch後に差分を確認できる理由と、upstreamがpush・pullの省略形を可能にする理由がつながります。

## 参考資料

- [Git: git-fetch](https://git-scm.com/docs/git-fetch)
- [Git: git-pull](https://git-scm.com/docs/git-pull)
- [Git: git-push](https://git-scm.com/docs/git-push)
- [Git: git-branch](https://git-scm.com/docs/git-branch)
- [Git: git-clone](https://git-scm.com/docs/git-clone)
- [GitHub Docs: Quickstart for pull requests](https://docs.github.com/en/pull-requests/get-started/pull-request-quickstart)
- [GitHub Docs: About comparing branches in pull requests](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-comparing-branches-in-pull-requests)
