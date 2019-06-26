# この話のゴール

* [前回](http://qiita.com/yoichi22/items/3f165abd83de623c21d2) と同様に、Gitを多少使ったことがある人向け
* `git reset` が何をリセットするのかを理解する
* `git reset` を安心して使えるようになる
* reflog と stash の関係に感銘を受ける

# ステージングエリア、作業ツリー

Gitのcommit済みの履歴は.git/objectsに格納された commit オブジェクトで管理されていました。また、ステージングエリア(index)には `git add` で登録されたblobたちを参照するtreeが格納されているのを見ました。

commit直後の index には何が格納されているでしょうか？

```
% git init
Initialized empty Git repository in .git/
% echo aaa > aaa.txt
% git add aaa.txt
% git commit -m "initial commit"
[master (root-commit) c2e81a7] initial commit
 1 file changed, 1 insertion(+)
 create mode 100644 aaa.txt
% git cat-file -p c2e8
tree a760aa994bbdd615f7ecb442be6bab636e11eba6
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448066205 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448066205 +0900

initial commit
% git write-tree
a760aa994bbdd615f7ecb442be6bab636e11eba6
```

commit直後の `git write-tree` では、最新のcommitの根本のtreeと同じハッシュ値が表示されました。つまり、commit 直後の index には最新のcommitと同じツリーが格納されています。別の見方をすると、 `git commit` は index 上のツリーを commit オブジェクトにぶら下げて .git/objects に格納するという処理をしていると言えます。

Git を用いた開発では、以下の手順を繰り返します

1. ソースコードを作成、編集する
2. `git add` で index 上のツリーを更新する
3. `git commit` で index 上のツリーをcommit オブジェクトにぶら下げる

このとき、次の3種類のツリー（根本の tree オブジェクトから辿れる tree, blob の集合をツリーと呼ぶことにします）が順に更新されていきます。

![trees.png](https://qiita-image-store.s3.amazonaws.com/0/64323/a5fcc5b5-4064-b9e9-af6b-5eacc04f780f.png)

* 作業ツリー ... tree,blob オブジェクトとも、必ずしも .git/objects/ に格納されていない。
* index 上の(キャッシュされた)ツリー ... 参照されるオブジェクトのうち、 tree オブジェクトたちは必ずしも .git/objects/ に格納されていない。
* commit済みのツリーたち ... commit オブジェクトから参照されるツリー。参照するオブジェクトは全て .git/objects/ に格納されている。

上２つでは（同じハッシュを持つtreeやblobが既に登録済みの場合があり得るものの、）一般にツリーの一部または全体は .git/objects/ に未格納です。それに対し commit 済みのツリーでは参照している全てのオブジェクトが .git/objects/ 格納済みであり、ファイルの集合をリポジトリから再構築できることが保証されます。

# ツリーの差分

`git diff` はツリーとツリーの差分を計算して出力します。

* `git diff` ... index から作業ツリーへの差分を出力する。(ただし一度もaddされていないファイルは含まれない)
* `git diff --cached` ... HEADが指す commit 済みツリーから index への差分を出力する
* `git diff HEAD` ... HEADが指すcommit済みのツリーから作業ツリーへの差分を出力する

![diff.png](https://qiita-image-store.s3.amazonaws.com/0/64323/c32ecc94-0bf6-a605-cefa-933482b6b04f.png)

管理対象のファイルが1つだけの単純な場合に限定して、いくつかの状態でそれぞれの差分がどう見えるか確認しましょう。

## コミット直後のdiff

```
% git init
Initialized empty Git repository in .git/
% echo aaa > aaa.txt
% git add aaa.txt
% git commit -m "1st commit"
[master (root-commit) e65c8d9] 1st commit
 1 file changed, 1 insertion(+)
 create mode 100644 aaa.txt
```

コミット直後には、HEAD と index と作業ツリーは同じツリーを指しているので、

* index から作業ツリーへの差分はない
* HEAD から index への差分はない
* HEAD から作業ツリーへの差分はない

```
% git diff
% git diff --cached
% git diff HEAD
```

といずれも差分はありません。

## 変更してaddする前のdiff

次に aaa.txt の内容を変更してみます

```
% echo bbb > aaa.txt
```

このとき、HEAD と index は同期したままですが、作業ツリーには変更が加わっているので、

* index から作業ツリーへの差分はある
* HEAD から index への差分はない
* HEAD から作業ツリーへの差分は、index から作業ツリーへの差分と同じ

```
% git diff
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
% git diff --cached
% git diff HEAD
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
```

## add した後のdiff

`git add` すると作業ツリーの内容が index に反映されます。

```
% git add aaa.txt
```

HEAD と作業ツリーの状態はいずれも変化していないので、

* index から作業ツリーへの差分はない
* HEAD から index への差分はある
* HEAD から作業ツリーへの差分は、HEADから index への差分と同じ

```
% git diff
% git diff --cached
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
% git diff HEAD
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
```

となり、この状態で `git commit` すると HEAD と index と作業ツリーの三者が再び同期します。

## コミットハッシュを指定

コミットハッシュを指定して、該当する commit からの差分を計算することもできます。

* `git diff --cached commit_hash` ... commit 済みのツリーから index への差分を出力する
* `git diff commit_hash` ... commit済みのツリーから作業ツリーへの差分を出力する

先ほどまでの例で、`git --cached` としていたのは実は `git diff --cached HEAD` と同じ(省略した場合には HEADからの差分になっていた)で、HEAD と書いた時は HEAD が指すコミットハッシュを指定するのと同じ動作になります。

```
% git diff --cached HEAD
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
% cat .git/HEAD
ref: refs/heads/master
% cat .git/refs/heads/master
e65c8d9e99a99caedf4f12e02383374361106eb6
% git diff --cached e65c8
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
```

なお、以下のようにすれば .git/ 以下のファイルを見ずとも HEAD が指しているコミットハッシュを確認できます：

```
% git rev-parse HEAD
e65c8d9e99a99caedf4f12e02383374361106eb6
```

## commit済みのツリー同士の差分

コミットハッシュを２つ指定すると commit済みのツリー同士の差分を出力できます：

* `git diff commit_hash1 commit_hash2`

3つのコミットがある状態を作って試してみましょう。

```
% git init
Initialized empty Git repository in .git/
% echo aaa > aaa.txt
% git add aaa.txt
% git commit -m "1st commit"
[master (root-commit) b44614f] 1st commit
 1 file changed, 1 insertion(+)
 create mode 100644 aaa.txt
% echo bbb > aaa.txt
% git add aaa.txt
% git commit -m "2nd commit"
[master 4c390b3] 2nd commit
 1 file changed, 1 insertion(+), 1 deletion(-)
% echo ccc > aaa.txt
% git add aaa.txt
% git commit -m "3rd commit"
[master e46be24] 3rd commit
 1 file changed, 1 insertion(+), 1 deletion(-)
```

2つ目と3つ目のコミットの差分を計算してみます。

```
% git log --oneline
e46be24 3rd commit
4c390b3 2nd commit
b44614f 1st commit
% git diff 4c39 e46b
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
```

これは以下と同じでした。

```
% git diff 4c39 HEAD
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
```

親との差分をとることはよくあることなので、以下のように書くこともできます。

```
% git diff HEAD~1 HEAD
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
```

HEAD~1 は HEAD から親を1回辿ったという意味です。同様に HEAD~N で親をN回辿ったコミットを表せます。

```
% git log --oneline
e46be24 3rd commit
4c390b3 2nd commit
b44614f 1st commit
% git rev-parse HEAD
e46be24d6b72c82e758deddff6c3ed064ee4ac08
% git rev-parse HEAD~1
4c390b3c8af5bc1e1fcdb57637565e9674c7b04b
% git rev-parse HEAD~2
b44614f2ad50308650dc4df5f1586c4c306a213f
```

# git reset

続けて `git reset` について見ていきましょう。
`git reset` により、HEADが指すツリー、indexが指すツリー、作業ツリーのそれぞれを更新することができます。

* `git reset --soft` ... HEAD のみをリセットする
* `git reset [--mixed]` ... HEAD と index をリセットする
* `git reset --hard` ... HEAD と index と作業ツリーをリセットする

これらを用いて、commitを無かったことにしたり、addを取り消したり、作業ツリーの変更を破棄してcommit済みのツリーと一致させたりといったことができます。具体的な動作を見てみましょう。

## soft reset

`git reset --soft` は HEAD のみをリセットし、index と作業ツリーの内容は変更しません。
例えばcommit直後の状態(HEADとindexと作業ツリーが一致している状態)で実行すると、

```
% git log --oneline
e46be24 3rd commit
4c390b3 2nd commit
b44614f 1st commit
% git diff
% git diff --cached
% git diff HEAD
% git diff HEAD~1 HEAD
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
% git reset --soft HEAD~1
% git rev-parse HEAD
4c390b3c8af5bc1e1fcdb57637565e9674c7b04b
% git diff
% git diff --cached
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
% git diff HEAD
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
```

コミット `e46b` が取り消されて HEAD は一つ前のコミット `4c39` に戻り、一方で index と作業ツリーはそのまま維持されているので、aaa.txt の変更を git add した状態に戻っています。

このとき、取り消されたコミットの内容を確認すると

```
% git cat-file -p e46b
tree 44e83542115cedc5609ac091e454a80b81c65c8d
parent 4c390b3c8af5bc1e1fcdb57637565e9674c7b04b
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448785412 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448785412 +0900

3rd commit
```

と .git/objects/ に存在しています。Git は基本的に追記のみをするので、この操作によって失われる情報は何もありません。実際に reset 前のコミットを指定して再度 `git reset --soft` すると元の状態に戻すことができます。

```
% git log --oneline
4c390b3 2nd commit
b44614f 1st commit
% git reset --soft e46b
% git rev-parse HEAD
e46be24d6b72c82e758deddff6c3ed064ee4ac08
% git log --oneline
e46be24 3rd commit
4c390b3 2nd commit
b44614f 1st commit
```

commit 済みの HEAD が指す先だけを変更するので、たとえ index や作業ツリーが HEAD と一致していない状態であろうと、 `git reset --soft` で失われる情報はありません。

## mixed reset

`git reset --mixed` (reset のデフォルト動作なので `--mixed` は省略可能)では HEAD と index をリセットし、作業ツリーの内容は変更しません。`git add` されている状態

```
% git log --oneline
4c390b3 2nd commit
b44614f 1st commit
% git diff
% git diff --cached
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
```

で `git reset --mixed` すると

* index は HEAD と一致するようにリセットされる
* 作業ツリーはそのまま維持される

となります。

```
% git reset --mixed
Unstaged changes after reset:
M	aaa.txt
% git diff
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
% git diff --cached
```

index 上のツリーと作業ツリーが同じだった場合には、単に add 前の状態に戻るだけで失われる情報はありませんので、再度 add すれば元の状態を復元できます。

```
% git add aaa.txt
% git diff
% git diff --cached
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
```

一方、index 上のツリーと作業ツリーが異なっている状態で `git reset --mixed` した場合には、index 上のツリーの情報は失われます。例えば、

```
% git diff --cached
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
% git diff
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
```

と HEAD、index、作業ツリーが全て異なっている状態で `git reset --mixed` すると

```
% git reset --mixed
Unstaged changes after reset:
M	aaa.txt
% git diff --cached
% git diff
diff --git a/aaa.txt b/aaa.txt
index 72943a1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+ccc
```

のように index 上のツリーの情報が失われてしまうので注意が必要です。

## hard reset

`git reset --hard` では HEAD と index および作業ツリーをリセットします。
実行後は HEAD が指すツリーと index が指すツリー、作業ツリーは一致した状態になります。

HEAD と index と作業ツリーが一致している状態で `git reset --hard` した場合にはそれらを一致させたまま自由に HEAD をリセットすることができ、それにより失なわれる情報はありません。

```
% git log --oneline
b940586 3rd commit
d8b5159 2nd commit
ee1dca4 1st commit
% git diff --cached
% git diff
% git reset --hard HEAD~1
HEAD is now at d8b5159 2nd commit
% git reset --hard HEAD~1
HEAD is now at ee1dca4 1st commit
% git reset --hard b940
HEAD is now at b940586 3rd commit
% git log --oneline
b940586 3rd commit
d8b5159 2nd commit
ee1dca4 1st commit
% git diff --cached
% git diff
```

一方で、HEAD と index と作業ツリーが異なる状態で `git reset --hard` すると index と作業ツリーの情報は失われます。このことを意図的に利用するとindex と作業ツリーの内容を破棄してHEADの状態に戻せます。コードを少しいじったけど、やっぱり違うやり方がいいと思ってやり直したいときなどに使えます：

```
% git diff --cached
diff --git a/aaa.txt b/aaa.txt
index 72943a1..f761ec1 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-aaa
+bbb
% git diff
diff --git a/aaa.txt b/aaa.txt
index f761ec1..b2a7546 100644
--- a/aaa.txt
+++ b/aaa.txt
@@ -1 +1 @@
-bbb
+ccc
% git reset --hard
HEAD is now at ee1dca4 1st commit
% git diff --cached
% git diff
```

# reflog

さて、HEAD 以外を指定して git reset を実行した場合、実行前に HEAD が指していたツリーはそのまま .git/objects/ 以下に保持されているので、そのコミットハッシュを覚えていれば HEAD を元に戻すことができました。で、実際のところ過去のコミットハッシュを覚えているでしょうか？

普通は覚えていませんよね。でも安心して下さい。Git が代わりに覚えてくれています。

```
% git init
Initialized empty Git repository in .git/
% echo aaa > aaa.txt
% git add aaa.txt
% git commit -m "1st commit"
[master (root-commit) dafe8c9] 1st commit
 1 file changed, 1 insertion(+)
 create mode 100644 aaa.txt
% find .git/logs -type f | xargs wc -l
  1 .git/logs/HEAD
  1 .git/logs/refs/heads/master
  2 total
% echo bbb > aaa.txt
% git add aaa.txt
% git commit -m "2nd commit"
[master ffeaf86] 2nd commit
 1 file changed, 1 insertion(+), 1 deletion(-)
% find .git/logs -type f | xargs wc -l
  2 .git/logs/HEAD
  2 .git/logs/refs/heads/master
  4 total
% git add aaa.txt
% git commit -m "3rd commit"
[master 40fd03d] 3rd commit
 1 file changed, 1 insertion(+), 1 deletion(-)
% find .git/logs -type f | xargs wc -l
  3 .git/logs/HEAD
  3 .git/logs/refs/heads/master
  6 total
% git reset --hard ffea
HEAD is now at ffeaf86 2nd commit
% find .git/logs -type f | xargs wc -l
   4 .git/logs/HEAD
   4 .git/logs/refs/heads/master
   8 total
% git reset --hard dafe
HEAD is now at dafe8c9 1st commit
% find .git/logs -type f | xargs wc -l
   5 .git/logs/HEAD
   5 .git/logs/refs/heads/master
  10 total
```

commit あるいは reset を行う度に .git/logs/HEAD と .git/logs/refs/heads/master が一行ずつ増えていることが見れました。ファイルの内容を見ると共に同じ中身で、

```
% diff -u .git/logs/HEAD .git/logs/refs/heads/master
% cat .git/logs/HEAD
0000000000000000000000000000000000000000 dafe8c9a992f3ab31d9008927550e913a29789be Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448633506 +0900	commit (initial): 1st commit
dafe8c9a992f3ab31d9008927550e913a29789be ffeaf86a68e62be970edc5bc83a0e018d2cb2e25 Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448633636 +0900	commit: 2nd commit
ffeaf86a68e62be970edc5bc83a0e018d2cb2e25 40fd03dfc5190e08c93d471d162f1c0048d1f9e5 Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448633690 +0900	commit: 3rd commit
40fd03dfc5190e08c93d471d162f1c0048d1f9e5 ffeaf86a68e62be970edc5bc83a0e018d2cb2e25 Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448633741 +0900	reset: moving to ffea
ffeaf86a68e62be970edc5bc83a0e018d2cb2e25 dafe8c9a992f3ab31d9008927550e913a29789be Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448633767 +0900	reset: moving to dafe
```

行なった操作と見比べると、

1. 操作前のコミットハッシュ(親がなければ0)
2. 操作後のコミットハッシュ
3. 操作を行なったユーザ
4. 操作を行なった日時とタイムゾーン
5. 操作の内容を表わす文字列

が各行に記録されています。`git reflog` によりこの要約を表示できます。

```
% git reflog
dafe8c9 HEAD@{0}: reset: moving to dafe
ffeaf86 HEAD@{1}: reset: moving to ffea
40fd03d HEAD@{2}: commit: 3rd commit
ffeaf86 HEAD@{3}: commit: 2nd commit
dafe8c9 HEAD@{4}: commit (initial): 1st commit
% git reflog master
dafe8c9 master@{0}: reset: moving to dafe
ffeaf86 master@{1}: reset: moving to ffea
40fd03d master@{2}: commit: 3rd commit
ffeaf86 master@{3}: commit: 2nd commit
dafe8c9 master@{4}: commit (initial): 1st commit
```

.git/log/ 以下のファイルと逆順(上の方が新しい)になっていることに注意してください。
過去に HEAD が指していたコミットハッシュを覚えていなくても、この情報を元に `git reset` することができます。なお、コミットハッシュの右にある名前を指定して `git reset` することもできます。

```
% git reset --hard HEAD@{2}
HEAD is now at 40fd03d 3rd commit
% git reflog
40fd03d HEAD@{0}: reset: moving to HEAD@{2}
dafe8c9 HEAD@{1}: reset: moving to dafe
ffeaf86 HEAD@{2}: reset: moving to ffea
40fd03d HEAD@{3}: commit: 3rd commit
ffeaf86 HEAD@{4}: commit: 2nd commit
dafe8c9 HEAD@{5}: commit (initial): 1st commit
% git reset --hard HEAD@{4}
HEAD is now at ffeaf86 2nd commit
% git reflog
ffeaf86 HEAD@{0}: reset: moving to HEAD@{4}
40fd03d HEAD@{1}: reset: moving to HEAD@{2}
dafe8c9 HEAD@{2}: reset: moving to dafe
ffeaf86 HEAD@{3}: reset: moving to ffea
40fd03d HEAD@{4}: commit: 3rd commit
ffeaf86 HEAD@{5}: commit: 2nd commit
dafe8c9 HEAD@{6}: commit (initial): 1st commit
```

# stash

`git stash` は reflog の仕組みを利用して差分たちを保持します。stash 自体の説明は他の文献にまかせて、ここではそのデータ保持方法のみ眺めておきましょう。

```
% git init
Initialized empty Git repository in .git/
% echo aaa > aaa.txt
% git add aaa.txt
% git commit -m "1st commit"
[master (root-commit) a26f0ff] 1st commit
 1 file changed, 1 insertion(+)
 create mode 100644 aaa.txt
% find .git/refs -type f
.git/refs/heads/master
% find .git/logs -type f
.git/logs/HEAD
.git/logs/refs/heads/master
```

aaa.txt を書き換えて `git stash save` してみます

```
% echo bbb > aaa.txt
% git stash save
Saved working directory and index state WIP on master: a26f0ff 1st commit
HEAD is now at a26f0ff 1st commit
% find .git/refs -type f
.git/refs/heads/master
.git/refs/stash
% cat .git/refs/stash
85cef9f5fdfcd420025071157b8d42f44ed09504
% git cat-file -p 85ce
tree ab0bfd782b3b2e0f9c565f42c8ffc7156139949d
parent a26f0ffd2214c4e45aeb2ba0dcca4d0ca76a7e56
parent 3a059345891f58303ed653b5e69ffc73474e2734
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448802800 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448802800 +0900

WIP on master: a26f0ff 1st commit
% git cat-file -p 3a05
tree a760aa994bbdd615f7ecb442be6bab636e11eba6
parent a26f0ffd2214c4e45aeb2ba0dcca4d0ca76a7e56
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448802800 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1448802800 +0900

index on master: a26f0ff 1st commit
% find .git/logs -type f
.git/logs/HEAD
.git/logs/refs/heads/master
.git/logs/refs/stash
% git reflog stash
85cef9f stash@{0}: WIP on master: a26f0ff 1st commit
```

リポジトリの状態から、

* index を元に commit オブジェクト `3a05` を生成
* HEAD (`a26f`) と `3a05` を親とする commit オブジェクト `85ce` を生成
* `85ce` を refs/stash として保持
* refs/stash の reflog が logs/refs/stash に記録される

となっています。さらに stash save すると refs/stash が書き換わってその reflog が logs/refs/stash に記録されますが、それがまさに `git stash list` で表示されるリストになっていることが見れます。

```
% echo ccc > aaa.txt
% git stash save
Saved working directory and index state WIP on master: a26f0ff 1st commit
HEAD is now at a26f0ff 1st commit
% cat .git/refs/stash
5d7dc0d6fc348d7aa2add40d37533768ee7876dd
% git reflog stash
5d7dc0d stash@{0}: WIP on master: a26f0ff 1st commit
85cef9f stash@{1}: WIP on master: a26f0ff 1st commit
% git stash list
stash@{0}: WIP on master: a26f0ff 1st commit
stash@{1}: WIP on master: a26f0ff 1st commit
```

stash は使っていたものの、それ自体の仕組みを知らずに使っていたので、これを見た時は本当にうまいことやってるなと感銘を受けました。

# まとめ

* Gitでは3種類のツリーが現れることを見た。
* git reset の soft, mixed, hard の動作を見た。
 * 失われる情報がない(安全に実行できる)状況があることを見た。
 * 状況に応じて何の情報が失われるかを見た。
* reflog と stash の関係を見た。

## リファレンス

* [git-diff](https://git-scm.com/docs/git-diff)
* [git-reset](https://git-scm.com/docs/git-reset)
* [git-stash](https://git-scm.com/docs/git-stash)
* [Git from the Bottom Up by John Wiegley](https://jwiegley.github.io/git-from-the-bottom-up/)

## to be continued

次回は今回辿りつけなかったブランチとマージの話を書く予定です。
