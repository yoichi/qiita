# 概要

[前々回](http://qiita.com/yoichi22/items/3f165abd83de623c21d2) は write-tree の挙動から、commit 準備されたツリーを構成する tree オブジェクトに相当する情報が index に格納されていることを見ました。[前回](http://qiita.com/yoichi22/items/b58ea6d4058d9425c04b)は、diff により commit 済みツリーと index 上のツリーの差分、index 上のツリーと作業ツリーの差分を確認する方法を見ました。ここまでは index から取り出した情報を元に index について間接的に調べてきましたが、今回は index のデータ構造を直接的に見ていきます。
まず基本的な役割であるコミット対象となるツリーの情報の管理について見た後、ブランチのマージの競合解決において index が果たす役割について見ていきましょう。

# index ファイルの基本構造

index ファイルのフォーマットは [Git index format](https://www.kernel.org/pub/software/scm/git/docs/technical/index-format.txt) にあり、基本的な構造は

* header
* index entries
* extensions (cached tree, resolve undo)
* SHA-1 checksum

となっています。 

## SHA-1 checksum

ファイル末尾20バイトはそれを除いたデータの SHA-1 ハッシュ値が格納されています。
これにより index ファイルの破損を検出することができます。

## header

ヘッダには以下の情報が含まれています

* index ファイル形式のバージョン
* 格納されている index entries の個数

## index entries

index entries にはヘッダに記載された個数分の index entry が並びます。index entry は可変長なので、一つづつ順番にパースしていく必要があります。各々に含まれる主な要素は

* ファイルモード
* ファイルの内容を格納する blob オブジェクトのSHA-1ハッシュ値(オブジェクトの名前)
* ステージ (0,1,2,3)
* ファイルパス

です。ステージについては後で詳しく見ますが通常は 0 になっています。それ以外の3つで tree オブジェクトに相当する情報を表現します。

## extensions

extensions は状況に応じて必要な情報が格納されます。

* cached tree extension はオブジェクトとして実体化した tree オブジェクトのキャッシュです。
* resolve undo extension はマージ情報のバックアップで、コンフリクト解消した後にマージをやり直す時に使われます。

# index の中身

index の中身を見るために Python を使って簡単なパーサを書きました (https://github.com/yoichi/git-cat-index) ので、それを使って具体例で見てみましょう。まず新規にリポジトリを作って add してみます。

```
% git init
Initialized empty Git repository in .git/
% touch aaa.txt
% git add aaa.txt
% git_cat_index.py .git/index
DIRC (dircache), 1 entries
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 aaa.txt
```

DIRC 以下に表示されているのが index entry です。さらに add すると

```
% mkdir 000
% touch 000/bbb.txt
% git add 000/bbb.txt
% mkdir zzz
% touch zzz/ccc.txt
% git add zzz/ccc.txt
% git_cat_index.py .git/index
DIRC (dircache), 3 entries
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 000/bbb.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 aaa.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 zzz/ccc.txt
```

と index entries が増え、名前でソートされているのが見れます。
この状態で commit すると、cached tree extension が index に追加されます。

```
% git commit -m "1st commit"
[master (root-commit) d29c4dd] 1st commit
 3 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 000/bbb.txt
 create mode 100644 aaa.txt
 create mode 100644 zzz/ccc.txt
% git_cat_index.py .git/index
DIRC (dircache), 3 entries
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 000/bbb.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 aaa.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 zzz/ccc.txt
TREE
47d9b8bcb208f8e88fc46e119abf7ba64f789ab6 (2/3)
0da07b494317103a607f0e42bb787afe111b5113 (0/1) 000
26099338016104b1a90a2917c607b1593e776c76 (0/1) zzz
% git cat-file -p 47d9
040000 tree 0da07b494317103a607f0e42bb787afe111b5113	000
100644 blob e69de29bb2d1d6434b8b29ae775ad8c2e48c5391	aaa.txt
040000 tree 26099338016104b1a90a2917c607b1593e776c76	zzz
% git cat-file -p 0da0
100644 blob e69de29bb2d1d6434b8b29ae775ad8c2e48c5391	bbb.txt
% git cat-file -p 2609
100644 blob e69de29bb2d1d6434b8b29ae775ad8c2e48c5391	ccc.txt
```

cached tree extension には、Git オブジェクトとして実体化された tree オブジェクトの情報が、

* tree オブジェクトのSHA-1ハッシュ
* サブツリーの数/内包するエントリ数
* ツリー自体のパス

の組み合わせで保持されています。

ファイルを編集して `git add` すると関連する部分が更新されます。

```
% echo aaa >> aaa.txt
% git add aaa.txt
% git_cat_index.py .git/index
DIRC (dircache), 3 entries
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 000/bbb.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (stage:0) 100644 aaa.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 zzz/ccc.txt
TREE
invalidated (2/-1)
0da07b494317103a607f0e42bb787afe111b5113 (0/1) 000
26099338016104b1a90a2917c607b1593e776c76 (0/1) zzz
```

* 編集したファイル(aaa.txt)の index entry の blob のSHA-1ハッシュ値が更新されている
* cached tree extension で変更の影響を受ける部分が無効化されている
 * 上の例では TREE 以下の aaa.txt を含むルートディレクトリの部分

この状態でコミットすると cached tree extension に生成されたツリーの情報が反映され、DIRC と TREE が合致した状態になります。

```
% git commit -m "update aaa.txt"
[master b2f2c1b] update aaa.txt
 1 file changed, 1 insertion(+)
% git_cat_index.py .git/index
DIRC (dircache), 3 entries
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 000/bbb.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (stage:0) 100644 aaa.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 zzz/ccc.txt
TREE
0d5e7c8f86e7f6f502e4ba65bb9906888cbd5529 (2/3)
0da07b494317103a607f0e42bb787afe111b5113 (0/1) 000
26099338016104b1a90a2917c607b1593e776c76 (0/1) zzz
```

結局 index には

* add 済みファイルに対して、tree に相当する情報(パス + blob のSHA-1ハッシュ値)
* コミット後にはそれに加えて、tree のキャッシュ(パス + tree のSHA-1ハッシュ値)

が格納されていることがわかりました。コンフリクト時にはさらにマージに関する情報が格納されるのを以下で見ていきましょう。

# ブランチとマージ

コンフリクトの話をする前に、ブランチとマージについて簡単におさらいしておきます。

## ブランチ

新規にリポジトリを作成し、ブランチを作ってみましょう。

```
% git init
Initialized empty Git repository in .git/
% touch readme.txt
% git add readme.txt
% git commit -m "initial commit"
[master (root-commit) 36409bb] initial commit
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 readme.txt
% find .git/objects -type f
.git/objects/36/409bb723d38326b0372d6dbe2557e275302405
.git/objects/77/37016481fd9dbdf0ec0d9145d56358fd71feb2
.git/objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391
% find .git/refs -type f
.git/refs/heads/master
% git branch feat-a
% find .git/objects -type f
.git/objects/36/409bb723d38326b0372d6dbe2557e275302405
.git/objects/77/37016481fd9dbdf0ec0d9145d56358fd71feb2
.git/objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391
% find .git/refs -type f
.git/refs/heads/feat-a
.git/refs/heads/master
```

`git branch` でブランチが作成されますが、このときオブジェクトは増えていません。一方で、.git/refs/heads 以下にブランチ名と同じ名前のファイルができており、これがブランチの実体です。

```
% cat .git/refs/heads/feat-a
36409bb723d38326b0372d6dbe2557e275302405
% git rev-parse feat-a
36409bb723d38326b0372d6dbe2557e275302405
```

この内容は `git branch` を実行した時点で HEAD が指していたコミットハッシュと一致しています。つまり Git においてブランチを作成する処理は、既に存在していたコミットオブジェクトへの参照を追加することに過ぎません。

ブランチ名を指定して `git checkout` すると HEAD が指定したブランチを指すようになります。

```
% git branch
  feat-a
* master
% cat .git/HEAD
ref: refs/heads/master
% git checkout feat-a
Switched to branch 'feat-a'
% git branch
* feat-a
  master
% cat .git/HEAD
ref: refs/heads/feat-a
```

この状態で commit すると .git/refs/heads/feat-a が新しいコミットを指すように更新され、一方で
.git/refs/heads/master は維持されます。

```
% diff .git/refs/heads/master .git/refs/heads/feat-a
% echo aaa > readme.txt
% git commit -a -m "commit in feat-a"
[feat-a b03280e] commit in feat-a
 1 file changed, 1 insertion(+)
% ls -l `find .git/refs -type f`
-rw-r--r--  1 yoichi  staff  41 12 10 06:54 .git/refs/heads/feat-a
-rw-r--r--  1 yoichi  staff  41 12 10 06:50 .git/refs/heads/master
% diff -u .git/refs/heads/master .git/refs/heads/feat-a
--- .git/refs/heads/master	2015-12-10 06:50:40.000000000 +0900
+++ .git/refs/heads/feat-a	2015-12-10 06:54:20.000000000 +0900
@@ -1 +1 @@
-36409bb723d38326b0372d6dbe2557e275302405
+b03280e575721d77052b886052caa9124100c67b
```

このようにブランチを作成してコミットをしていくことで、特定の機能の開発を他の開発の影響を受けずに進めたり、リリース対象を隔離して必要な変更のみ適用したりといった運用が可能になります。
なお、ここでは `git branch` と `git checkout` を組み合わせましたが、`git checkout -b` を使うとブランチを作成してそれをチェックアウトするまでを一気にやることもできます。

## マージ

feat-a でコミットした変更を master にマージしてみましょう。

```
% git checkout master
Switched to branch 'master'
% git merge --no-ff feat-a
Merge made by the 'recursive' strategy.
 readme.txt | 1 +
 1 file changed, 1 insertion(+)
% git log --graph --oneline
*   bf19c29 Merge branch 'feat-a'
|\
| * b03280e commit in feat-a
|/
* 36409bb initial commit
```

ブランチを統合したときにできるコミット(上の場合 `bf19`)をマージコミットと呼びます。
コミットオブジェクトの中を見ると、parent が二つ記録されています。

```
% git cat-file -p bf19
tree 580c73c39691399d09ad01152ad0a691ce80bccf
parent 36409bb723d38326b0372d6dbe2557e275302405
parent b03280e575721d77052b886052caa9124100c67b
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1449698148 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1449698148 +0900

Merge branch 'feat-a'
```

マージする前の状態は HEAD~1 もしくは ORIG_HEAD で参照できます。
マージをなかったことにしてみましょう。

```
% git rev-parse HEAD~1
36409bb723d38326b0372d6dbe2557e275302405
% cat .git/ORIG_HEAD
36409bb723d38326b0372d6dbe2557e275302405
% git rev-parse ORIG_HEAD
36409bb723d38326b0372d6dbe2557e275302405
% git reset --hard ORIG_HEAD
HEAD is now at 36409bb initial commit
% git log --graph --oneline
* 36409bb initial commit
% cat .git/ORIG_HEAD
bf19c29c28f76249cde3c8b138f767261e9ec9cf
```

ORIG_HEAD は merge や reset をすると更新され、操作前のHEADが指していたコミットを指しますので、操作を即座に取り消したい場合などに使うとよいでしょう。

もう一つ見ておきたいことがあるので、再度 reset を行ない、マージ済みの状態に戻しましょう。

```
% git reset --hard HEAD@{1}
HEAD is now at bf19c29 Merge branch 'feat-a'
% git log --graph --oneline
*   bf19c29 Merge branch 'feat-a'
|\
| * b03280e commit in feat-a
|/
* 36409bb initial commit
% git log --graph --oneline feat-a
* b03280e commit in feat-a
* 36409bb initial commit
```

マージ済みのブランチは安全に削除することができます。

```
% find .git/refs -type f
.git/refs/heads/feat-a
.git/refs/heads/master
% find .git/logs -type f
.git/logs/HEAD
.git/logs/refs/heads/feat-a
.git/logs/refs/heads/master
% git branch -d feat-a
Deleted branch feat-a (was b03280e).
% find .git/refs -type f
.git/refs/heads/master
% find .git/logs -type f
.git/logs/HEAD
.git/logs/refs/heads/master
```

削除したときに、refs 以下のブランチの実体とともに、logs 以下のログも削除されていることに注意してください。「安全に」とは削除したブランチを復活できるという意味です。マージコミットの親コミットから、マージ前のコミットハッシュがわかるのでそこからブランチを作り直せます。

```
% git checkout b032
Note: checking out 'b032'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at b03280e... commit in feat-a
% git checkout -b feat-a
Switched to a new branch 'feat-a'
% git log --graph --oneline
* b03280e commit in feat-a
* 36409bb initial commit
```

なお、復活させるブランチ名は Git オブジェクトとして明示的に保持されていないので、

* マージコミットのメッセージから抽出する(コミットログにマージ元ブランチ名が入っていた場合)
* reflog HEAD で見れるログ文字列から抽出する(手元でブランチ操作していた場合のみ)

といった間接的な情報から抽出するしかありません。

次に master ブランチをマージ前に戻して、feat-a がまだマージされていない状態にします。そこでブランチ feat-a を削除しようとすると、マージ済みじゃないのでエラーになります。

```
% git reset --hard HEAD~1
HEAD is now at 36409bb initial commit
% git branch -d feat-a
error: The branch 'feat-a' is not fully merged.
If you are sure you want to delete it, run 'git branch -D feat-a'.
```

ここまで `git reset` を乱用しつつ進んできた我々にとっては、ブランチ削除したとしてもコミットオブジェクト自体が消えるわけではないことを知っているのでそれほどの危険性はないのですが、`git branch` は初心者も安心して使える方向性を目指しているため、refs/heads から参照がなくなって混乱しないようエラーにしてくれています。本当に削除したい場合にはエラーメッセージにあるように `git branch -D` とすれば削除できます。

# コンフリクト

さて、本題のコンフリクトを見てみます。master はマージ前の状態に戻っていますので、
feature-a の変更と衝突するような変更をコミットしてからマージしてみます。

```
% echo bbb > readme.txt
% git commit -a -m "commit in master"
[master 60c2d95] commit in master
 1 file changed, 1 insertion(+)
% git log --graph --oneline
* 60c2d95 commit in master
* 36409bb initial commit
% ls .git
COMMIT_EDITMSG  branches/       hooks/          logs/
HEAD            config          index           objects/
ORIG_HEAD       description     info/           refs/
% git merge --no-ff feat-a
Auto-merging readme.txt
CONFLICT (content): Merge conflict in readme.txt
Automatic merge failed; fix conflicts and then commit the result.
% cat readme.txt
<<<<<<< HEAD
bbb
=======
aaa
>>>>>>> feat-a
```

想定通りコンフリクトしました。このとき、.git ディレクトリの中を見ると

```
% ls .git
COMMIT_EDITMSG  MERGE_MODE      branches/       hooks/          logs/
HEAD            MERGE_MSG       config          index           objects/
MERGE_HEAD      ORIG_HEAD       description     info/           refs/
% cat .git/ORIG_HEAD
60c2d9507e55666a10f42bfe06c5ea216822484e
% git rev-parse master
60c2d9507e55666a10f42bfe06c5ea216822484e
% cat .git/MERGE_HEAD
b03280e575721d77052b886052caa9124100c67b
% git rev-parse feat-a
b03280e575721d77052b886052caa9124100c67b
```

と、マージ前のHEAD(ORIG_HEAD,まだコミットしてないのでこの時点のHEADとも一致します)とマージ元(MERGE_HEAD)が記録されているのがわかります。

このとき index に格納されているものを見てみましょう。

```
% git_cat_index.py .git/index
DIRC (dircache), 3 entries
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:1) 100644 readme.txt
f761ec192d9f0dca3329044b96ebdb12839dbff6 (stage:2) 100644 readme.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (stage:3) 100644 readme.txt
```

それぞれの blob の中身を見ると、stage:2 がマージ先(ORIG_HEAD)、stage:3 がマージ元(MERGE_HEAD)のものになっていて、 stage:1 はこの場合は空のファイルです。

```
% git cat-file -p e69d
% git cat-file -p f761
bbb
% git cat-file -p 7294
aaa
```

stage:1 は実はマージ先とマージ元の共通の祖先のコミットに含まれるものです。

```
% git log --graph --oneline ORIG_HEAD
* 60c2d95 commit in master
* 36409bb initial commit
% git log --graph --oneline MERGE_HEAD
* b03280e commit in feat-a
* 36409bb initial commit
% git cat-file -p 3640
tree 7737016481fd9dbdf0ec0d9145d56358fd71feb2
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1449697840 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1449697840 +0900

initial commit
% git cat-file -p 7737
100644 blob e69de29bb2d1d6434b8b29ae775ad8c2e48c5391	readme.txt
```

二つのコミットの共通の祖先のコミットを見付けるには

```
% git merge-base ORIG_HEAD MERGE_HEAD
36409bb723d38326b0372d6dbe2557e275302405
```

とします。Git はマージを行う際に、共通の祖先からの差分同士の情報を使って、マージ結果を計算しています(3-way merge)。 index の情報を使って、マージ元ブランチのファイルを取り出したり、マージ先ブランチのファイルを取り出したり、マージ結果のファイルを再生成することができます。

```
% git checkout --theirs readme.txt
% cat readme.txt
aaa
% git checkout --ours readme.txt
% cat readme.txt
bbb
% git checkout -m readme.txt
% cat readme.txt
<<<<<<< ours
bbb
=======
aaa
>>>>>>> theirs
```

マージする前のコミットと作業ツリーの差分、
マージ対象のコミットと作業ツリーの差分はそれぞれ次のようにして見れます。

```
% git diff ORIG_HEAD
diff --git a/readme.txt b/readme.txt
index f761ec1..656de96 100644
--- a/readme.txt
+++ b/readme.txt
@@ -1 +1,5 @@
+<<<<<<< HEAD
 bbb
+=======
+aaa
+>>>>>>> feat-a
% git diff MERGE_HEAD
diff --git a/readme.txt b/readme.txt
index 72943a1..656de96 100644
--- a/readme.txt
+++ b/readme.txt
@@ -1 +1,5 @@
+<<<<<<< HEAD
+bbb
+=======
 aaa
+>>>>>>> feat-a
```

競合したファイルを編集してよきに図らった上で、

* `git add` してから `git commit`
* `git commit -a`

のいずれかでコミットすればマージコミットが記録されます。前者の手順で進めてみます。

```
% vi readme.txt
% git add readme.txt
% git_cat_index.py .git/index
DIRC (dircache), 1 entries
b47d0c04c248e6569a37fa9b327ad681466794ee (stage:0) 100644 readme.txt
REUC
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:1) 100644 readme.txt
f761ec192d9f0dca3329044b96ebdb12839dbff6 (stage:2) 100644 readme.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (stage:3) 100644 readme.txt
```

`git add` すると index 内の DIRC の所にあった stage:1-3 のエントリが REUC に移動され、DIRC には stage:0 としてマージ済みのblobが登録されています。この状態で

* `git checkout -m readme.txt` とすれば REUC の情報を元にコンフリクトした状態に戻れます。
* `git commit` すれば、index の内容をコミットできます。

コミットしてみましょう。

```
% git commit
[master 47010e0] Merge branch 'feat-a'
% git cat-file -p 4701
tree 81c664bf46f44e570f0775879a8850d4adf4bbcb
parent 60c2d9507e55666a10f42bfe06c5ea216822484e
parent b03280e575721d77052b886052caa9124100c67b
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1449841468 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1449841468 +0900

Merge branch 'feat-a'
```

parent を2つ持つマージコミットが生成されました。2つ目のparentは .git/MERGE_HEAD を元に設定されています。マージコミットが記録されたので、feat-a と master の共通の祖先は feat-a の先端と一致しており、もう一度 merge しても何も起こりません。

```
% git merge-base master feat-a
b03280e575721d77052b886052caa9124100c67b
% git rev-parse feat-a
b03280e575721d77052b886052caa9124100c67b
% git merge --no-ff feat-a
Already up-to-date.
```

コンフクトの解消を誤ったり、 .git/MERGE_HEAD の存在を認識せずにコミットして意図しないマージコミットが生成されてしまうと、その後マージしても想定したものが取り込まれないといったことが起こり得るので注意して下さい。もし、コンフリクトした時点でマージ自体をやめたくなった場合には、`git reset --hard` すれば index と作業ツリーの内容を戻して、.git/MERGE_HEADも削除してくれるので安心です。先程のマージコミットの手前に戻した上で試してみましょう。

```
% git reset --hard HEAD~1
HEAD is now at 60c2d95 commit in master
% git rev-parse HEAD
60c2d9507e55666a10f42bfe06c5ea216822484e
% git merge feat-a
Auto-merging readme.txt
CONFLICT (content): Merge conflict in readme.txt
Automatic merge failed; fix conflicts and then commit the result.
% git rev-parse HEAD
60c2d9507e55666a10f42bfe06c5ea216822484e
% cat .git/MERGE_HEAD
b03280e575721d77052b886052caa9124100c67b
% git reset --hard
HEAD is now at 60c2d95 commit in master
% git rev-parse HEAD
60c2d9507e55666a10f42bfe06c5ea216822484e
% cat .git/MERGE_HEAD
cat: .git/MERGE_HEAD: No such file or directory
```

# ローカル変更がある状態でのマージ

`git merge` では HEAD と指定したコミットの共通の祖先を探して処理を行うことを見ました。
ではローカル変更がある状態で `git merge` するとどうなるでしょうか？

まず準備します。

```
% git init
Initialized empty Git repository in .git/
% touch aaa.txt bbb.txt
% git add aaa.txt bbb.txt
% git commit -m "initial commit in master"
[master (root-commit) afdc80e] initial commit in master
 2 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 aaa.txt
 create mode 100644 bbb.txt
% git checkout -b feat-a
Switched to a new branch 'feat-a'
% echo aaa >> aaa.txt
% git commit -a -m "aaa in feat-a"
[feat-a e275b45] aaa in feat-a
 1 file changed, 1 insertion(+)
% git checkout master
Switched to branch 'master'
```

ローカル変更を加えます。

```
% echo bbb >> bbb.txt
% git add bbb.txt
% echo BBB >> bbb.txt
% echo ccc > ccc.txt
% git add ccc.txt
% git diff --cached
diff --git a/bbb.txt b/bbb.txt
index e69de29..f761ec1 100644
--- a/bbb.txt
+++ b/bbb.txt
@@ -0,0 +1 @@
+bbb
diff --git a/ccc.txt b/ccc.txt
new file mode 100644
index 0000000..b2a7546
--- /dev/null
+++ b/ccc.txt
@@ -0,0 +1 @@
+ccc
% git diff
diff --git a/bbb.txt b/bbb.txt
index f761ec1..f8ef162 100644
--- a/bbb.txt
+++ b/bbb.txt
@@ -1 +1,2 @@
 bbb
+BBB
```

`--no-ff` でマージするとエラーになります。

```
% git merge --no-ff feat-a
error: Your local changes to the following files would be overwritten by merge:
	bbb.txt
Please, commit your changes or stash them before you can merge.
Aborting
```

この時点でマージによる変更は加わっていないので、ローカルの変更とマージによる変更が混じってややこしい状態になるのを防いでくれています。こうなった場合の対処方法はやりたいことによって変わってきます：

* ローカル変更を破棄する場合→ `git reset --hard` してからマージをやり直し
* ローカル変更を先にコミットする場合→ コミットしてからマージをやり直し
* ローカル変更を退避させておいて先にマージする場合→ `git stash save` してからマージして、`git stash pop`

3つ目が少し複雑なので具体例で詳しく見てみましょう。

`git stash save` するとローカルの変更が退避されます

```
% git stash save
Saved working directory and index state WIP on master: afdc80e initial commit in master
HEAD is now at afdc80e initial commit in master
% git diff HEAD
% git reflog stash
f0c17f5 stash@{0}: WIP on master: afdc80e initial commit in master
% git diff HEAD stash@{0}
diff --git a/bbb.txt b/bbb.txt
index e69de29..f8ef162 100644
--- a/bbb.txt
+++ b/bbb.txt
@@ -0,0 +1,2 @@
+bbb
+BBB
diff --git a/ccc.txt b/ccc.txt
new file mode 100644
index 0000000..b2a7546
--- /dev/null
+++ b/ccc.txt
@@ -0,0 +1 @@
+ccc
```

ここで、作業ツリーにあった内容とindexにあった内容が一緒になっている(その差の情報は失なわれている)ことに注意して下さい。さて、HEAD との差分が退避されたのでマージをして、`git stash pop` で退避させていた情報を戻します。

```
% git merge --no-ff feat-a
Merge made by the 'recursive' strategy.
 aaa.txt | 1 +
 1 file changed, 1 insertion(+)
% git stash pop
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	new file:   ccc.txt

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   bbb.txt

Dropped refs/stash@{0} (f0c17f515814b1e2020c98ff584e2e2e7de51425)
```

退避されていた変更が、新規ファイルは index に登録された状態、既存ファイルの変更は index に登録されていない状態で作業ツリーに復旧されています(新規ファイルと既存ファイルの扱いが非対称になっていることの背景を私は理解できていません。`git commit -a` の結果が同じになるように？)

```
% git diff --cached
diff --git a/ccc.txt b/ccc.txt
new file mode 100644
index 0000000..b2a7546
--- /dev/null
+++ b/ccc.txt
@@ -0,0 +1 @@
+ccc
% git diff
diff --git a/bbb.txt b/bbb.txt
index e69de29..f8ef162 100644
--- a/bbb.txt
+++ b/bbb.txt
@@ -0,0 +1,2 @@
+bbb
+BBB
```

今回のケースだと問題ありませんでしたが、 `git stash pop` は作業ツリーへのマージを行っているので、マージにより同じファイルが変更された場合にはコンフリクトが発生することがあります。
さきほどの例の続きから、マージの前の状態に戻した上で試してみましょう。

```
% git reset --hard HEAD~1
HEAD is now at afdc80e initial commit in master
% echo AAA >> aaa.txt
% git merge --no-ff feat-a
error: Your local changes to the following files would be overwritten by merge:
	aaa.txt
Please, commit your changes or stash them before you can merge.
Aborting
% git stash save
Saved working directory and index state WIP on master: afdc80e initial commit in master
HEAD is now at afdc80e initial commit in master
% git merge --no-ff feat-a
Merge made by the 'recursive' strategy.
 aaa.txt | 1 +
 1 file changed, 1 insertion(+)
% git stash pop
Auto-merging aaa.txt
CONFLICT (content): Merge conflict in aaa.txt
% cat aaa.txt
<<<<<<< Updated upstream
aaa
=======
AAA
>>>>>>> Stashed changes
```

このとき index には、コミット間のマージをしたときと同じくマージ先、マージ元、共通の祖先の blob へのポインタが格納されています

```
% git_cat_index.py .git/index
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:1) 100644 aaa.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (stage:2) 100644 aaa.txt
43d5a8ed6ef6c00ff775008633f95787d088285d (stage:3) 100644 aaa.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 bbb.txt
```

この情報を用いて、マージ先、マージ元のファイルを取り出したり、コンフリクトを再現したりできます。

```
% git checkout --theirs aaa.txt
% cat aaa.txt
AAA
% git checkout --ours aaa.txt
% cat aaa.txt
aaa
% git checkout -m aaa.txt
% cat aaa.txt
<<<<<<< ours
aaa
=======
AAA
>>>>>>> theirs
```

一方で、MERGE_HEAD は存在していないため、コンフリクト解消してコミットすると(マージコミットではなく)通常のコミットとして記録されます。

```
% cat .git/MERGE_HEAD
cat: .git/MERGE_HEAD: No such file or directory
% vi aaa.txt
% git add aaa.txt
% git_cat_index.py .git/index
DIRC (dircache), 2 entries
9b9822a031a519d48c82873494045ce291067c87 (stage:0) 100644 aaa.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 bbb.txt
REUC
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:1) 100644 aaa.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (stage:2) 100644 aaa.txt
43d5a8ed6ef6c00ff775008633f95787d088285d (stage:3) 100644 aaa.txt
% git commit -m "stash poped"
[master 31446f5] stash poped
 1 file changed, 1 insertion(+)
% git cat-file -p 3144
tree b6d20047e7a9ebb019c4e38065b94c3dcdb13f55
parent 06b8c9db97e899bae7b1a6ec77c25d7109805b0a
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1450195362 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1450195362 +0900

stash poped
% git_cat_index.py .git/index
DIRC (dircache), 2 entries
9b9822a031a519d48c82873494045ce291067c87 (stage:0) 100644 aaa.txt
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:0) 100644 bbb.txt
TREE
b6d20047e7a9ebb019c4e38065b94c3dcdb13f55 (0/2)
REUC
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 (stage:1) 100644 aaa.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (stage:2) 100644 aaa.txt
43d5a8ed6ef6c00ff775008633f95787d088285d (stage:3) 100644 aaa.txt
```

# まとめ

* Gitのブランチはコミットオブジェクトへの参照として表現されている
 * ブランチ自体はオブジェクトとして永続的に管理される対象ではない
 * ローカルで reflog により一部の操作履歴のみ見れる
 * ブランチは削除した後に同じ名前で作り直され得ることに注意
* ブランチのマージにより二つのコミットを親とするコミットを生成できる
 * 3-way マージにより、ある程度までは自動でマージされる
 * 自動でマージされたからといって、整合したソースになっているとは限らないので注意
 * マージ前の状態を取り出したり、ブランチをその状態にリセットできる
 * マージしたブランチは安全に削除することができる
* マージでコンフリクトした時には対象ファイルの情報が index に保持されている
 * ファイル編集してコンフリクト解消した上でマージコミットを生成できる
 * マージ前のファイルやマージ元のファイル、コンフリクトの再現ができる
* stash pop でもマージが発生する
 * コンフリクト時の index の状態はブランチのマージの場合と同様

リポジトリの内部構造に触れながら Git の基本的な操作について3回に渡って見てきました。Git を使っていてややこしい状況になった時に、リポジトリの構造を知っていれば落ち付いて何が起きているかを調べられるはず。この文書が Git を使っている誰かの助けになればうれしいです。
欲張って内容盛り込みすぎた気がするのでわかりにくい部分があると思いますが、 手順に沿って操作すれば手元で内容を再現できるように書いたつもりなので、実際に手を動かして試していただければと思います。それでもわからない部分はコメントいただけるとありがたいです。

## 参考文献

* [こわくないgit](http://www.slideshare.net/kotas/git-15276118)
* [もうGitは怖くない： 自信を持って使いたいあなたへ](http://d.hatena.ne.jp/m-hiyama/20150928/1443397382)
