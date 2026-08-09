---
title: "複数のGitHubアカウントを分ける：SSHエイリアスと内蔵Gitの互換性で詰まった話"
emoji: "🔐"
type: "tech"
topics: ["git", "github", "ssh", "windows"]
published: true
---

1台のWindows PCで、個人用と業務用のGitHubアカウントを使い分ける必要がありました。

当初はどちらのリポジトリもHTTPSで利用していましたが、プライベートリポジトリへの`git pull`や`git push`で、次のエラーが発生するようになりました。

```text
remote: Repository not found.
fatal: repository 'https://github.com/example/example-repository.git/' not found
```

最終的には、個人用リポジトリをSSHに分離し、内蔵Git機能を使うリポジトリはHTTPSに戻しました。

この記事では、最初に試したSSHキーの分離がGit CLIでは成功した一方、別のツールに組み込まれたGit機能では失敗した理由と、最終的な切り分け方を整理します。

## 発生した問題

最初に確認できた状況は次のとおりです。

- ブラウザではプライベートリポジトリを開ける
- 個人用リポジトリへの`git push`が失敗する
- 別アカウントで利用するリポジトリの`git pull`も失敗する
- どちらも`Repository not found`と表示される

リポジトリがブラウザで開けたため、URLの入力ミスだけでなく、Gitが意図しないアカウントの認証情報を使っている可能性を考えました。

私の環境では、HTTPS認証に次の状態が関係していました。

- Windowsの資格情報マネージャー
- Git Credential Manager
- ブラウザのGitHubログイン状態
- 各アプリケーションが保持する認証状態

HTTPSそのものに複数アカウントを扱えない制限があるわけではありません。しかし、保存された認証情報の確認が不十分なままアカウントを切り替えていたため、どの資格情報が使われているのか分かりにくくなっていました。

## 最初に確認したこと

認証設定を変更する前に、まずリポジトリとリモートの状態を確認しました。

```powershell
git status
git remote -v
git rev-parse --show-toplevel
git config --get credential.helper
```

特に`git rev-parse --show-toplevel`は、現在地が本当に対象リポジトリの中かを確認するために有効でした。

`git status`が失敗したからといって、すぐに`git init`を実行すると、既存リポジトリの上位ディレクトリに別の`.git`を作り、入れ子のリポジトリにしてしまう可能性があります。

今回もフォルダーを整理する途中でリポジトリのルートが分かりにくくなったため、認証問題とディレクトリ構造の問題を分けて確認する必要がありました。

## 最初の対応：SSHキーをアカウントごとに分ける

HTTPSの認証状態に依存しないよう、個人用と業務用で別々のSSHキーを用意しました。

```powershell
ssh-keygen -t ed25519 -C "personal@example.com" -f "$HOME/.ssh/id_ed25519_personal"
ssh-keygen -t ed25519 -C "work@example.com" -f "$HOME/.ssh/id_ed25519_work"
```

生成されるファイルは、それぞれ秘密鍵と公開鍵のペアです。

```text
id_ed25519_personal
id_ed25519_personal.pub

id_ed25519_work
id_ed25519_work.pub
```

GitHubに登録するのは`.pub`が付いた公開鍵だけです。秘密鍵は共有したり、リポジトリにコミットしたりしてはいけません。

## SSH configで接続先を分ける

次に、`~/.ssh/config`へアカウントごとの接続設定を追加しました。

```sshconfig
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  IdentitiesOnly yes

Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
```

`github-personal`と`github-work`は実在するホスト名ではなく、ローカルで定義したSSHのエイリアスです。

どちらも実際には`github.com`へ接続しますが、`IdentityFile`によって使用する秘密鍵を切り替えます。`IdentitiesOnly yes`を指定すると、ssh-agentに複数の鍵が読み込まれていても、指定した鍵だけを認証に使用できます。

## リモートURLにもエイリアスを指定する

SSH configを書いただけでは、各リポジトリがどのエイリアスを使うか決まりません。リモートURLのホスト部分にも対応する名前を指定します。

```powershell
# 個人用リポジトリ
git remote set-url origin git@github-personal:personal-user/personal-repository.git

# 業務用アカウントでアクセスするリポジトリ
git remote set-url origin git@github-work:example-org/work-repository.git
```

これにより、リモートURLとSSHキーの対応は次のようになります。

```text
github-personal → id_ed25519_personal
github-work     → id_ed25519_work
```

設定後は、それぞれの接続を確認しました。

```powershell
ssh -T git@github-personal
ssh -T git@github-work
```

認証に成功すると、GitHubのシェルは利用できないものの、どのユーザーとして認証されたかが表示されます。

```text
Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

ここまではGit CLIで問題なく動作しました。

## 新しい問題：内蔵GitではURLが拒否された

ところが、業務で利用していたツールの内蔵Git機能からpullすると、次のエラーが発生しました。

```text
Unsupported URL protocol
```

同じリポジトリでも、PowerShellからGit CLIを実行するとSSHエイリアスを使えました。失敗するのは内蔵Git機能から操作した場合だけでした。

Gitは次のようなscp形式のSSH URLをサポートしています。

```text
git@host:owner/repository.git
```

しかし、すべてのGit連携ツールがGit CLIへURLをそのまま渡すとは限りません。今回の結果から、利用していたツールがリモートURLを独自に解析しており、SSHエイリアスを含むscp形式をサポート対象のプロトコルとして認識できなかった可能性が高いと判断しました。

これはツールの内部実装を確認した結果ではなく、次の差から導いた推測です。

- Git CLIでは同じリモートURLで成功する
- 内蔵Git機能では`Unsupported URL protocol`になる
- HTTPSへ戻すと内蔵Git機能でも成功する

## 最終対応：リポジトリごとに方式を分ける

すべてのリポジトリをSSHへ統一することにはこだわらず、実際に使うツールとの互換性を優先しました。

### 個人用リポジトリ

- SSHを使用する
- `github-personal`エイリアスを使用する
- HTTPSの認証状態から分離する

```text
git@github-personal:personal-user/personal-repository.git
```

### 内蔵Git機能を使うリポジトリ

- HTTPSへ戻す
- そのツールが対応する認証方法を使用する

```powershell
git remote set-url origin https://github.com/example-org/work-repository.git
git remote -v
git pull
```

この構成に変更した後、個人用リポジトリではSSHによるpush、内蔵Gitを使うリポジトリではHTTPSによるpullが成功することを確認しました。

## HTTPSで複数アカウントを使う選択肢

今回は個人用リポジトリをSSHへ移しましたが、複数アカウントをHTTPSだけで運用する方法もあります。

GitHubの公式ドキュメントでは、リモートURLのパスごとに資格情報を保持する設定として次のコマンドが案内されています。

```powershell
git config --global credential.https://github.com.useHttpPath true
```

ただし、アプリケーションの内蔵Git機能がOSのGitやGit Credential Managerと同じ設定を使うとは限りません。CLIだけで判断せず、実際に利用するツール側でもpullとpushを確認する必要があります。

## トラブルが再発したときの確認順序

今回の経験から、次の順番で確認すると問題を切り分けやすいと分かりました。

1. `git rev-parse --show-toplevel`でリポジトリのルートを確認する
2. `git remote -v`でリモートURLとプロトコルを確認する
3. SSHなら`ssh -T git@<alias>`で使用アカウントを確認する
4. HTTPSならCredential Helper（資格情報ヘルパー）と保存済み資格情報を確認する
5. Git CLIと内蔵Git機能の両方で同じ操作を試す
6. CLIだけ成功する場合は、URL形式や認証機能の互換性を疑う

## 今回分かったこと

- `Repository not found`はURLの不存在だけでなく、認証状態が原因の場合もある
- 複数アカウントでは、SSHキーとHostエイリアスで使用する鍵を明示できる
- SSH configとリモートURLのHost名は一致させる必要がある
- Git CLIで動くURLが、すべての内蔵Git機能で動くとは限らない
- SSHかHTTPSかを一律に決めず、認証の分離とツールの互換性を両方確認する
- `git init`の前に、現在のリポジトリルートを確認する

今回の一番大きな学びは、**Gitの認証問題とツールのURL互換性問題を分けて考えること**でした。

## 参考資料

- [GitHub Docs: Managing multiple accounts](https://docs.github.com/en/account-and-profile/how-tos/account-management/managing-multiple-accounts)
- [GitHub Docs: Testing your SSH connection](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)
- [GitHub Docs: Managing remote repositories](https://docs.github.com/en/get-started/git-basics/managing-remote-repositories)
- [Git Documentation: git-clone](https://git-scm.com/docs/git-clone)
