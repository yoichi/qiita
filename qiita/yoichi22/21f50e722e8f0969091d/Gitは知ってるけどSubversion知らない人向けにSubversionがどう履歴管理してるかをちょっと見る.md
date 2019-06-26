# この文書について

Gitを知っている人に、Subversionを理解するとっかかりを与えることを目指します。

最近周りの若い人から、Gitは知ってるけどSubversionはよく知らない、難しくて使えないという話を何度か聞きました。片方を知っているのであれば、両者のdiffを知ればもう片方も理解できるはずです。この文書では具体例としてそれぞれにおけるファイルのリネームの扱いを取り上げ、両者の履歴管理の差異を見ます。

# GitとSubversion

GitとSubversionはいずれもバージョン管理システム(VCS, Version Control System)という種類のソフトウェアであり、ソースツリー（ソフトウェアを構成するファイル群）とその変更履歴を管理します。

* Git
  * 作者: Linus Torvalds
  * 初版: 2005年
* Subversion
  * 作者: Karl Fogel, Jim Blandy (CollabNet)
  * 初版: 2000年

ある時点のソースツリーに対してリビジョンを割り当てていること、ブランチやタグを扱えるなど、ともにバージョン管理システムの基本機能を有することは同じですが、両者の設計には大きな方向性の違いがあります。実際、Linusが [Google Tech Talk のセッション](https://www.youtube.com/watch?v=4XpnKHJAok8) で

```
I see Subversion as being the most pointless project ever started,
because the slogan for Subversion for a while was, CVS done right
or something like that. (中略) There is no way to do CVS right.
```

と述べているように、Subversionは、それより昔から存在し広く使われていたCVSというバージョン管理システムの持つ課題を、その設計の延長線上で解決することを目指していました。一方でGitはLinuxカーネルという一つの具体的なソフトウェアをターゲットとして、多数の開発者が同時並行で書いたコードをいかにうまく統合するかという課題を解決するために、新たに書き起こされたものです。この問題設定の違いから両者の設計の大きな違いが生じています。

では具体的な中身に踏み込んで理解を深めていきましょう。リポジトリで管理されているファイルのリネームの扱いについて、まずはGitの方から見ていきます。

# ファイルのリネーム:Gitの場合

## リネームのやり方:Gitの場合

事前条件として、 fileA.txt というファイルがGitリポジトリで管理されているとします[^1]

[^1]: 例えば mkdir gitrepo && cd gitrepo && git init && touch fileA.txt && git add fileA.txt && git commit -m "initialize" でセットアップできます。

```shell-session
$ ls -a
./         ../        .git/      fileA.txt
```

ファイルをリネームしてコミットするには

```shell-session
$ git mv fileA.txt fileB.txt
$ git commit
```

とします。では、同じ事前条件の元で、以下のようにしたらどうなるでしょうか？

```shell-session
$ mv fileA.txt fileB.txt
$ git add fileA.txt
$ git add fileB.txt
$ git commit
```

git mv を使った場合、使わなかった場合で違いはあるでしょうか？Gitを知ってる人ならもちろんわかりますよね？

## 履歴の構造:Gitの場合

Gitは次の形で履歴を管理しています。

* CommitはトップレベルのTreeを持ち、Treeは子要素としてTreeやBlobを持つ[^2]
* Commitは親となるCommitを持つ

[^2]: Tree=ディレクトリ、Blob=ファイル

![git_commits.png](https://qiita-image-store.s3.amazonaws.com/0/64323/038b2c23-5abe-1c93-1e63-c9de2bc770cd.png)

ここで、 Blob 同士、Tree 同士は直接の関連を持たないことが要点です。

* ソースツリー全体のスナップショットに対してコミットを付与している 
* 個々のBlobやTreeの履歴を保持しているわけじゃない
* 今回はたまたま `Blob_B == Blob_A` だが本質ではない

git mvするのと、mvしてからgit addするのとで、ソースツリーのスナップショットは同じなので、Gitで履歴として記録される内容はどちらも同じです。

# CVSとその課題

GitやSubversionが現れる前に広く使われていたCVSというバージョン管理システムは、RCS由来のファイルフォーマットを使ってファイル単位で履歴管理を行うものでした。ファイルのパスはRCSファイルのパスで表現されます。

![cvs_commits.png](https://qiita-image-store.s3.amazonaws.com/0/64323/8c7afea2-379b-694d-b67c-d2d7a471587f.png)


多くのソフトウェアは一個のソースファイルのみから構成されるわけではなく、ファイル一式（複数のソースファイル、ビルド構成ファイル、ドキュメントなど）がひとかたまりになったものが価値を提供する単位になりますが、CVSはファイル毎の履歴しか管理できないので

* ソースツリー全体でのリビジョン管理ができない 
* ファイルのコピー、移動を追跡できない

ということが課題になりました。新たなバージョン管理システムの登場までの間、CVSを利用するソフトウェア開発者達は

* コミットメッセージとタイムスタンプが同じならファイルをまたいで同一のコミットだと解釈する
* リポジトリのRCSファイルのcp, mvをすることでファイルの移動、コピー時に履歴を維持する

という形で運用回避して使っていました。それらの課題に対してSubversionは、

* リビジョンをファイル一式に対して付与する
* ファイルのコピーや移動を追跡できるようにする

という設計変更により解決を図りました。リポジトリファイル内のリネームはSubversionではどう扱われるようになったのでしょうか？実際に見てみましょう。

# ファイルのリネーム:Subversionの場合

## リネームのやり方:Subversionの場合

こちらも同様に fileA.txt がSubversionリポジトリで管理されている状況を前提とします[^3]。ファイルをリネームしてコミットする方法として

[^3]: 例えば svnadmin create svnrepo && svn checkout file:///$(pwd)/svnrepo svnwc && cd svnwc && touch fileA.txt && svn add fileA.txt && svn commit -m "initialize" でセットアップできます。

```shell-session
$ svn move fileA.txt fileB.txt
$ svn commit
```

と

```shell-session
$ mv fileA.txt fileB.txt
$ svn delete fileA.txt
$ svn add fileB.txt
$ svn commit
```

があります。結論を言ってしまうと、これら２つのやり方では違うものが出来上がります。前者ではコミット後の fileB.txt の履歴からリネーム前の fileA.txt が辿れますが、後者では fileB.txt はリネーム前の fileA.txt とは無関係な新たなファイルとして扱われ、履歴を遡ってもリネーム前の fileA.txt に辿り着くことはできません。

## 履歴の構造:Subversionの場合

Subversionは次の形で履歴を管理しています。

* リポジトリ全体に対して整数の連番でリビジョンが割り当てられる
* ファイルやディレクトリはそれぞれの一つ前のリビジョンへのリンクを持つ

![svn_commits_props.png](https://qiita-image-store.s3.amazonaws.com/0/64323/ebae28d7-d275-e40c-c402-48b48acbd003.png) ![svn_commits.png](https://qiita-image-store.s3.amazonaws.com/0/64323/e826e645-9c6c-234f-d8e4-abcd1a9635f1.png)


Subversionは個々のファイルやディレクトリの履歴を保持しているので、リネームの２つのやり方によって図中で★で示したリンクの有無が違いとして現れます。パスを変えずにファイルの内容を更新した場合は同じパスにあるファイルの前のリビジョンから更新されたとみなしてくれますが、今回の例のようにリネームでパスが変わった場合にはそのことをsvn moveで明示的に記録しない限り、履歴は引き継がれません。

# まとめ

GitとSubversionのそれぞれがファイルのリネームをどう扱っているかを見ました。Subversionでは個々のファイルやディレクトリの履歴を追跡して管理していましたが、Gitでは個々のファイルの履歴を管理しておらず、ファイルの履歴は必要に応じてコンテンツの類似性から算出するものという位置づけになっています。この違いを知っておくと、Subversionの仕様を理解する助けになるでしょう。

この文書では触れませんでしたが、GitとSubversionでは

* ブランチ、タグの表現方法
* マージの記録保持方法

についても大きな違いがあります。これらも見てみると面白いので興味のある方は是非。

求める機能を持つソフトウェアをどう設計するかという観点でSubversionとGitを見比べると、問題をどう捉えるかによって全く別のものが出来上がるという一つの例になっています。問題発見について興味のある方には書籍「[ライト、ついてますか - 問題発見の人間学](https://www.amazon.co.jp/dp/4320023684)」をおすすめします。

一方、求める機能に対して何を使うかという観点では、SubversionリポジトリはGitリポジトリに履歴も含めて移行できる[^4]ので、Subversion知らない人が多い状況では、Subversionを頑張って理解するのではなく、既存のリポジトリを移行した上で、よく知っているGitを使うようにするのも一つの方法かもしれません。これもある意味問題をどう捉えるかという話ですね。

[^4]: 詳しくは https://git-scm.com/book/en/v2/Git-and-Other-Systems-Migrating-to-Git を見てください。
