# git checkout -

`git checkout -` とすると一つ前に居たブランチ(もしくはリビジョン)をチェックアウトしてくれる。別のブランチをチェックアウトして作業した後に元のブランチに戻りたい場合などに使うと便利。

```shell-session
$ git rev-parse --abbrev-ref HEAD
master
$ git checkout maint
Switched to branch 'maint'
Your branch is up to date with 'origin/maint'.
$ git rev-parse --abbrev-ref HEAD
maint
$ git checkout -
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
```

さらに `git checkout -`するとブランチ間を行ったり来たりできる。

```shell-session
$ git checkout -
Switched to branch 'maint'
Your branch is up to date with 'origin/maint'.
$ git checkout -
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
```

# 仕組み

リポジトリのどこかに前に居たブランチの情報を持っているのかなと思って実装を確認した。
[builtin/checkout.c](https://github.com/git/git/blob/master/builtin/checkout.c) の `parse_branchname_arg()` を見ると

```c
	if (!strcmp(arg, "-"))
		arg = "@{-1}";
```

となっていて、これにより `-` は `@{-1}` に読み替えられている。`@` で始まる記法なので [reflog](https://git-scm.com/docs/git-reflog)を使って一つ前の状態を取得しているのかなと推測したが、reflogのドキュメントには関連する記述は見当たらなかった。`-` が導入されたコミット[^1]を見ると負の整数を指定する書き方は `-` が導入される以前からあったようなので、さらに実装を確認していくと [sha1-name.c](https://github.com/git/git/blob/master/sha1-name.c) の `interpret_nth_prior_checkout()` で reflog 中の "checkout: moving from " を見ていることがわかり[^2]、reflogを使ってるという推測が正しいと確認できた。

ドキュメントでの言及は reflog の方じゃなく [checkout](https://git-scm.com/docs/git-checkout) のマニュアルでは見つけることができた[^3]：

```
You can use the "@{-N}" syntax to refer to the N-th last branch/commit checked out
using "git checkout" operation. You may also specify - which is synonymous to "@{-1}.
```

[^1]: https://github.com/git/git/commit/696acf45f9638b014c7132508de26d1a571c8a33
[^2]: https://github.com/git/git/commit/ae5a6c3684c378bc32c1f6ecc0e6dc45300c14c1
[^3]: クォート閉じ忘れを見つけたのでパッチ送っておきました。

# 何で `-` なの？

同僚にこの機能を説明する時に `cd -` と同じだよと言ったところ、そもそもそちらを知らなくてピンときてなかった。割と知られていないことなのかもしれないが、 `cd -` とすると前居たディレクトリに移動できる：

```shell-session
$ cd /usr
$ pwd
/usr
$ cd /var
$ pwd
/var
$ cd -
/usr
$ pwd
/usr
$ cd -
/var
$ pwd
/var
$ 
```

別のディレクトリに移動して何かした後に、元のディレクトリに戻るときとか、間違って cd しちゃったときに元のディレクトリに戻るとかいうときに便利。`git checkout -` の `-` はこれを真似てるんじゃないかと勝手に思ってる。
