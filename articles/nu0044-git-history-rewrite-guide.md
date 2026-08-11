---
title: "Gitで「元に戻す」を選び間違えない：restore・amend・reset・rebase・revert"
emoji: "↩️"
type: "tech"
topics: ["git", "github", "初心者", "バージョン管理"]
published: false
---

Gitには変更を「元に戻す」ためのコマンドが複数あります。

```text
restore
commit --amend
reset
rebase -i
revert
```

名前だけを覚えると、どのコマンドを使えばよいか迷います。特に`reset --hard`やrebaseは、対象を誤ると必要な変更を失ったり、共有済みの履歴を壊したりする可能性があります。

この記事では、**何を戻したいのか**と**その履歴を他の人と共有したか**という2つの基準から使い分けを整理します。

## 最初に確認する2つのこと

コマンドを実行する前に、次の質問に答えます。

1. 対象はワーキングツリー、ステージングエリア、コミットのどこにあるか
2. 対象のコミットはまだローカルだけか、すでにpushして共有したか

```text
ワーキングツリー  --git add-->  ステージングエリア  --git commit-->  コミット
```

この位置関係が分かると、候補をかなり絞れます。

| 状況 | 最初に検討するコマンド | 履歴を書き換えるか |
|---|---|---|
| コミット前のファイル変更を破棄したい | `git restore` | いいえ |
| ステージからだけ外したい | `git restore --staged` | いいえ |
| 直前のローカルコミットを修正したい | `git commit --amend` | はい |
| ローカルブランチを以前のコミットへ戻したい | `git reset` | はい |
| 複数のローカルコミットを整理したい | `git rebase -i` | はい |
| 共有済みコミットの効果を取り消したい | `git revert` | いいえ。新しいコミットを追加 |

## コミット前の変更はrestore

### ステージから外して編集内容は残す

```bash
git restore --staged memo.md
```

ステージングエリアの`memo.md`をHEADの内容へ戻します。ワーキングツリーの編集内容は残るため、次のコミット候補から外すだけです。

### ワーキングツリーの編集内容を破棄する

```bash
git restore memo.md
```

デフォルトではステージングエリアを元にワーキングツリーを復元します。未保存の変更が消えるため、先に差分を確認します。

```bash
git status
git diff -- memo.md
```

`restore --staged`とオプションなしの`restore`では、変更する場所が異なります。

## 直前のローカルコミットはamend

コミットメッセージを修正する場合は次のように実行します。

```bash
git commit --amend -m "正しいコミットメッセージ"
```

入れ忘れたファイルを追加する場合は、ステージしてからamendします。

```bash
git add missed-file.md
git commit --amend --no-edit
```

amendは既存コミットを直接編集するのではなく、内容を反映した別のコミットで置き換えます。そのためコミットハッシュが変わります。

すでに共有したコミットをamendすると、ローカルとリモートの履歴が一致しなくなります。基本的にはpush前のコミットだけに使用します。

## ローカルブランチの位置を戻すreset

```bash
git reset [--soft|--mixed|--hard] <commit>
```

コミットを指定するresetは、現在のブランチが指す位置を移動します。モードによってステージングエリアとワーキングツリーも変更されます。

| モード | ブランチ・HEAD | ステージングエリア | ワーキングツリー |
|---|---|---|---|
| `--soft` | 移動 | 維持 | 維持 |
| `--mixed` | 移動 | 対象コミットに合わせる | 維持 |
| `--hard` | 移動 | 対象コミットに合わせる | 対象コミットに合わせる |

### soft：コミットだけを作り直す

```bash
git reset --soft HEAD~1
```

直前のコミットは取り消されますが、変更はステージされたままです。複数の変更を確認してからコミットし直す場合に使えます。

### mixed：ステージも解除する

```bash
git reset HEAD~1
```

`--mixed`はデフォルトのモードです。変更はワーキングツリーに残りますが、再度`git add`する必要があります。

### hard：ワーキングツリーも対象コミットへ戻す

```bash
git reset --hard HEAD~1
```

追跡中のファイルにある未コミットの変更も失われます。また、対象コミットのファイル配置と衝突する未追跡ファイルやディレクトリも削除される場合があります。

実行前に最低限、次を確認します。

```bash
git status
git branch --show-current
git log --all --graph --decorate --oneline -10
```

## 複数のローカルコミットを整理するinteractive rebase

最近の3つのコミットを整理する例です。

```bash
git rebase -i HEAD~3
```

エディターで次のように指定すると、後ろの2つを最初のコミットへまとめられます。

```text
pick   <hash1> 機能を追加
squash <hash2> 説明を修正
squash <hash3> テストを修正
```

interactive rebase（対話的rebase）では、コミットの並び替え、メッセージ変更、結合、削除ができます。ただし対象コミットを作り直すためハッシュが変わります。

共有済みのブランチではなく、push前の個人作業ブランチをレビューしやすい単位へ整える用途に向いています。

途中で取り消す場合と、競合解決後に再開する場合は次のコマンドを使います。

```bash
git rebase --abort
git rebase --continue
```

## 共有済みの変更はrevertで取り消す

```bash
git revert <commit>
```

revertは過去のコミットを削除しません。対象コミットの変更を打ち消す新しいコミットを作ります。

```text
A---B---C        共有済みの履歴
A---B---C---R    Cの効果を取り消すコミット R
```

履歴が追加されるため、他の開発者が持っているコミットとの対応関係を維持できます。

「履歴を消す」のではなく、「変更を取り消した事実も履歴に残す」方法です。

## HEAD~とHEAD^の違い

直線の履歴では`HEAD~1`と`HEAD^1`は同じ親コミットを指します。

マージコミットでは意味の違いが見えます。

```text
HEAD~2  : 1番目の親を2回たどる
HEAD^2  : HEADの2番目の親を選ぶ
```

相対参照をresetやrebaseへ渡す前に、対象コミットを確認します。

```bash
git show --stat HEAD~2
```

## 間違えてresetした場合はreflogを確認する

GitはHEADの移動履歴をreflogへ一定期間記録します。

```bash
git reflog
```

必要なコミットを見つけたら、先に保護用ブランチを作ります。

```bash
git branch recovery <commit-hash>
```

ただしreflogがあっても、コミットしていなかったワーキングツリーの変更まで必ず復元できるわけではありません。

## 選択フロー

```text
コミット前の変更か
  ├─ ステージから外すだけ → restore --staged
  └─ ワーキングツリーの変更を破棄 → restore

まだローカルだけのコミットか
  ├─ 直前を修正 → commit --amend
  ├─ ブランチ位置を戻す → reset
  └─ 複数コミットを整理 → rebase -i

すでに共有したコミットか
  └─ 効果を取り消す新しいコミットを作る → revert
```

## まとめ

Gitで安全に元へ戻すには、コマンド名より対象の状態を先に確認します。

特に重要なのは、**push前のローカル履歴**と**他の人が利用している共有履歴**を分けることです。前者はamend、reset、rebaseで整理できますが、後者はrevertで履歴を維持しながら取り消す方が安全です。

## 参考資料

- [Git: git-restore](https://git-scm.com/docs/git-restore)
- [Git: git-reset](https://git-scm.com/docs/git-reset)
- [Git: git-revert](https://git-scm.com/docs/git-revert)
- [Git: git-commit](https://git-scm.com/docs/git-commit)
- [Git: git-rebase](https://git-scm.com/docs/git-rebase)
- [Git: git-reflog](https://git-scm.com/docs/git-reflog)
