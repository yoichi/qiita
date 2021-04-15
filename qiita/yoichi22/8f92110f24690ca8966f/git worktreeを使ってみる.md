`git worktree` を使うと、一つのローカルリポジトリで作業ツリーを複数同時に持てる。
このコマンドは Git 2.5.0 から導入されている。
[Git - git-worktree Documentation](https://git-scm.com/docs/git-worktree)

既存のブランチを新たな作業ツリーとしてチェックアウトしたい場合、

```sh
git worktree add 生成する作業ツリーのパス ブランチ名
```

とする。例えば、feat-a ブランチで作業中に develop ブランチをチェックアウトして同時に利用したい場合は以下のようにすればよい

```sh
% pwd
/Users/yoichi/testrepo
% git branch
  develop
* feat-a
  master
% git worktree add ~/testrepo_develop develop
Preparing /Users/yoichi/testrepo_develop (identifier testrepo_develop)
HEAD is now at b5fb78b commit in develop
% cd ~/testrepo_develop
% git branch
* develop
  feat-a
  master
```

引数で指定したパス ~/testrepo_develop に、もう一つの引数で指定した develop ブランチがチェックアウトされた。

worktree で作成した作業ツリーは通常の作業ツリーと同様に利用することができる。例えばそこで更に作業ツリーを作成することも可能である。

```sh
% pwd
/Users/yoichi/testrepo_develop
% git worktree add ~/testrepo_master master
Preparing /Users/yoichi/testrepo_master (identifier testrepo_master)
HEAD is now at fcd0b60 commit in master
```

`git worktree` で生成された作業ツリーには、.git/ ディレクトリの変わりに .git ファイルが存在し、チェックアウト元リポジトリ内のディレクトリのパスが記載されている。

```sh
% cat .git
gitdir: /Users/yoichi/testrepo/.git/worktrees/testrepo_master
```

そのディレクトリの内容は以下のようになっている。

```sh
% cd /Users/yoichi/testrepo/.git/worktrees/testrepo_master
% ls
COMMIT_EDITMSG  ORIG_HEAD       gitdir          logs/
HEAD            commondir       index
% cat commondir
../..
% cat gitdir
/Users/yoichi/testrepo_master/.git
% ls logs
HEAD
```

* commondir には通常のリポジトリへの相対パスが記載されている。
* gitdir は作業ツリー側の .git ファイルのパスが記載されている。
* logs/ には HEAD の reflog が記録されている。
 * ブランチの reflog は通常のリポジトリ側 (上の例だと ../../logs/refs/heads/ 以下) に記録されている。

`git worktree list` で作業ツリーの一覧を取れる (Git 2.7.0 で対応されたとのこと)

```sh
% git worktree list
/Users/yoichi/testrepo          b5fb78b [feat-a]
/Users/yoichi/testrepo_develop  5b74214 [develop]
/Users/yoichi/testrepo_master   fcd0b60 [hoge]
```

既に作業ツリーが存在しているブランチをチェックアウトしようとするとエラーになる

```sh
% pwd
/Users/yoichi/testrepo
% git checkout develop
fatal: 'develop' is already checked out at '/Users/yoichi/testrepo_develop'
```

同じブランチをチェックアウトしている作業ツリー上で `git checkout` で別のブランチをチェックアウトして同じブランチがチェックアウトされていない状態にするか、もしくは、作業ツリーを消した上で `git worktree prune` すると、そのブランチをチェックアウトできるようになる。後者をやってみる:

```sh
% pwd
/Users/yoichi/testrepo
% rm -rf ~/testrepo_develop
% git checkout develop
fatal: 'develop' is already checked out at '/Users/yoichi/testrepo_develop'
% git worktree prune
% git worktree list
/Users/yoichi/testrepo         b5fb78b [feat-a]
/Users/yoichi/testrepo_master  fcd0b60 [master]
% git checkout develop
Switched to branch 'develop'
```

## まとめ

`git worktree` を知るまでは複数のブランチのファイルを同時に参照したい場合に

1. ローカルでブランチ切り替えて、同時に参照したいファイルを個別にコピー
2. リモートから複数クローンしてくる
3. ローカルでクローン

とかしていたが、1. は事故の元だし 2.,3. はローカルに複数のリポジトリのコピーができてややこしかった。
`git worktree` を知ったのでこれからは使ってみようと思っているが、要らなくなった時に `rm -rf` するところで間違ってローカルリポジトリ本体を消しちゃうと怖いなと思ってる。

(追記) 要らなくなったワークツリーに対して `git worktree remove ワークツリーのパス` でワークツリーのファイル削除と、ローカルリポジトリ上の管理情報の削除を一緒にやれる。ワークツリーじゃないパスを指定すると弾いてくれるし、こちらを使ったほうが事故りにくいですね。
