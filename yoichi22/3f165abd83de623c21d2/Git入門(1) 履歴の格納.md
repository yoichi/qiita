# この話のゴール

* Git を多少使ったことがある人に向けた解説
* git add, commit の裏側で、シンプルなデータ構造で履歴が格納されていることを知る
* コミットログに何を書くべきか、がんばって考えようという気持ちになる

# はじめに

Gitはバージョン管理システムの一つです。バージョン管理システムとは、

* ファイルの集合を格納する
* ある時点のファイルの集合を素早く取り出せる
* 変更の履歴を参照できる

などの機能を持つシステムです。ここで「変更の履歴」とは、

* 誰が、いつ、何をいじったか
* コミットログメッセージ
 * 変更内容の要約
 * 変更を選択した背景
* 実際の変更内容

を、時系列に沿って記録した一連の情報を指します。

バージョン管理システムとしては Git の他に CVS や Subversion などがありますが、Gitの特徴としては次のようなものがあります

* ファイルの内容そのものを保持する
 * CVS,Subverionは差分を保持する
* 履歴をSHA1ハッシュ値で管理する
 * CVS, Subversionは連番で管理する
* ファイルやディレクトリの移動、コピーを管理しない
 * Subversion は移動、コピーを管理する
 * Subversion ではブランチ、タグをディレクトリの空間的コピーで表現します
 * CVS はファイル単位の履歴のみを管理するので、移動、コピーの管理は不可能
* 履歴の改ざんができない
 * CVS, Subversionはやろうと思えば改ざんできる
 * 履歴の作り直しなら容易にできる
* 複数のリポジトリ間での履歴の統合ができる
 * CVS, Subversion は一つのリポジトリ内で閉じているので不可能
 * 複数人でのコラボレーションを容易にする

# コンテンツの格納

はじめに、コンテンツ格納庫としての git の動作を見てみましょう。
まず、空の git リポジトリを作成します

```
% mkdir repo1
% cd repo1
% git init
Initialized empty Git repository in repo1/.git/
```

readme.txt というファイルを作って、その内容をリポジトリに格納し、さらに別の内容でファイルを上書きして、それをリポジトリに格納します。

```
% echo aaa > readme.txt
% git hash-object -w readme.txt
72943a16fb2c8f38f9dde202b7a70ccc19c52f34
% echo bbb > readme.txt
% git hash-object -w readme.txt
f761ec192d9f0dca3329044b96ebdb12839dbff6
% rm -f readme.txt
```

hash-object コマンド実行時に表示された文字列をキーとして、格納されたファイルの内容を取り出せます。

```
% git cat-file -p 72943a16fb2c8f38f9dde202b7a70ccc19c52f34 > readme.txt
% cat readme.txt
aaa
% git cat-file -p f761ec192d9f0dca3329044b96ebdb12839dbff6 > readme.txt
% cat readme.txt
bbb
```

注目すべき点は

* ファイルの内容を(何やらよくわからない文字列をキーとして)格納し、取り出せた
* ファイル名(パス)はまだ格納できていない
* ファイルの変更(ある時点の内容から別のある時点の内容への差分)は格納できていない

ことです。できていない部分ができるようになると、変更の履歴の保持ができるようになるのですが、それを見る前に、「何やらよくわからない文字列」がファイルの内容(+α)の SHA1 ハッシュ値であることを確認しておきましょう。


```py:hash-object.py
import hashlib
import sys

if len(sys.argv) != 2:
    print("usage: %s file" % sys.argv[0])
    sys.exit(-1)

try:
    f = open(sys.argv[1])
except Exception:
    print("open %s failed" % sys.argv[1])
    sys.exit(-1)

data = f.read()
sha1 = hashlib.sha1("blob %d" % len(data) + "\0" + data).hexdigest()
print(sha1)
```

この Python のプログラムは

* "blob " という文字列
* コンテンツ(ファイルの内容)のサイズ
* ヌル文字
* コンテンツ

を連結したものの SHA1 ハッシュ値を計算して、その16進数での表現を出力します。
実際に先程の2つのコンテンツに対して使ってみると、

```
% echo aaa | python hash-object.py /dev/stdin
72943a16fb2c8f38f9dde202b7a70ccc19c52f34
% echo bbb | python hash-object.py /dev/stdin
f761ec192d9f0dca3329044b96ebdb12839dbff6
```

と、計算結果が先程コンテンツの格納に用いられていたキーと一致する（ファイルの内容+αの SHA1 ハッシュ値をキーとしてコンテンツが格納されている）ことがわかります。
さて、コンテンツのリポジトリ内での格納先は 

```
% find .git/objects -type f
.git/objects/72/943a16fb2c8f38f9dde202b7a70ccc19c52f34
.git/objects/f7/61ec192d9f0dca3329044b96ebdb12839dbff6
```

であり、ディレクトリ名とファイル名を連結したものがSHA1ハッシュ値になっていますが、
これらのファイル自体のSHA1値をとってみると

```
% sha1sum `find .git/objects -type f`
cf6e4f80cfae36e20ae7eb1a90919ca48f59514b  .git/objects/72/943a16fb2c8f38f9dde202b7a70ccc19c52f34
cdb05607e2e073287a81a908564d9d901ccdd687  .git/objects/f7/61ec192d9f0dca3329044b96ebdb12839dbff6
```

と値が異なっています。これは内容を圧縮して格納しているためであり、例えば

```py:decompress_sha1.py
import hashlib
import sys
import zlib

if len(sys.argv) != 2:
    print("usage: %s git_object_file" % sys.argv[0])
    sys.exit(-1)

path = sys.argv[1]
try:
    f = open(path)
except Exception:
    print("open %s failed" % path)
    sys.exit(-1)

data = zlib.decompress(f.read())
sha1 = hashlib.sha1(data).hexdigest()
print("%s: %s" % (path, sha1))
```

というプログラムを使って解凍した上でのハッシュ値を計算すると

```
% for i in `find .git/objects -type f`; do python ../decompress_sha1.py $i; done
.git/objects/72/943a16fb2c8f38f9dde202b7a70ccc19c52f34: 72943a16fb2c8f38f9dde202b7a70ccc19c52f34
.git/objects/f7/61ec192d9f0dca3329044b96ebdb12839dbff6: f761ec192d9f0dca3329044b96ebdb12839dbff6
```

のようにちゃんと一致している(ハッシュ値が一致しているので、内容も一致していることが期待される)ことが見れます。

# ディレクトリツリーの格納

ファイルの内容を .git/objects/ 以下に格納する方法を見ました。Git は、それに加えてファイル名やコミットログメッセージの情報も Git オブジェクトと呼ばれる .git/objects/ 以下のファイルに格納します。

リポジトリを作成した時点では、オブジェクトは1つも格納されていない状態です。

```
% mkdir repo2
% cd repo2
% git init
Initialized empty Git repository in repo2/.git/
% ls .git
HEAD         config       hooks/       objects/
branches/    description  info/        refs/
% find .git/objects -type f
```

git add でステージングエリアにファイルを一個追加してみましょう。

```
% echo aaa > readme.txt
% git add readme.txt
% find .git/objects -type f
.git/objects/72/943a16fb2c8f38f9dde202b7a70ccc19c52f34
% ls .git
HEAD         config       hooks/       info/        refs/
branches/    description  index        objects/
```

オブジェクトが1つ追加され、index というファイルができています。
Git オブジェクトの内容は git cat-file で確認できます。

```
% git cat-file -t 729
fatal: Not a valid object name 729
% git cat-file -t 7294
blob
% git cat-file -s 7294
4
% wc -c readme.txt
4 readme.txt
% git cat-file -p 7294
aaa
% cat readme.txt
aaa
```

cat-file の使い方として、

* ハッシュ値を与えますが、先頭の4文字以上を与えればそれにマッチするものを拾い上げてくれます
 * 3文字以下の場合、マッチするものが複数ある場合は弾かれる 
* `-t` で種類を確認すると、前の節で見たようにファイルの内容を格納する blob オブジェクトです
* `-s` でサイズ、`-p` で内容を表示すると、実際のファイルの内容と合致していました

次に index に入っている情報をオブジェクトとして書き出してみましょう。

```
% git write-tree
580c73c39691399d09ad01152ad0a691ce80bccf
% find .git/objects -type f
.git/objects/58/0c73c39691399d09ad01152ad0a691ce80bccf
.git/objects/72/943a16fb2c8f38f9dde202b7a70ccc19c52f34
% git cat-file -t 580c
tree
% git cat-file -p 580c
100644 blob 72943a16fb2c8f38f9dde202b7a70ccc19c52f34    readme.txt
```

このとき、

* 新たに tree という種類のオブジェクト `580c` が格納された
* ファイル名と、blob オブジェクトへのポインタが入っている。

ことがわかります。

![tree-1.png](https://qiita-image-store.s3.amazonaws.com/0/64323/cad89c23-8b54-1148-7b91-8bf3d95666cb.png)

次にディレクトリとその下にファイルを作成して git add してみます。

```
% mkdir tmp
% echo bbb > tmp/bbb.txt
% git add tmp/bbb.txt
% find .git/objects -type f
.git/objects/58/0c73c39691399d09ad01152ad0a691ce80bccf
.git/objects/72/943a16fb2c8f38f9dde202b7a70ccc19c52f34
.git/objects/f7/61ec192d9f0dca3329044b96ebdb12839dbff6
% git cat-file -t f761
blob
% git cat-file -p f761
bbb
```

新たに追加されたオブジェクトは bbb.txt の内容を格納した blob オブジェクトです。
この状態で再度 index をオブジェクトとして書き出すと

```
% git write-tree
6434b2415497a42647800c7e828038a2fb6fbbaf
% find .git/objects -type f
.git/objects/58/0c73c39691399d09ad01152ad0a691ce80bccf
.git/objects/5c/40d98927de9cdb27df5b3a7bd4f7ee95dbfc85
.git/objects/64/34b2415497a42647800c7e828038a2fb6fbbaf
.git/objects/72/943a16fb2c8f38f9dde202b7a70ccc19c52f34
.git/objects/f7/61ec192d9f0dca3329044b96ebdb12839dbff6
% git cat-file -t 6434
tree
% git cat-file -p 6434
100644 blob 72943a16fb2c8f38f9dde202b7a70ccc19c52f34    readme.txt
040000 tree 5c40d98927de9cdb27df5b3a7bd4f7ee95dbfc85    tmp
% git cat-file -t 5c40
tree
% git cat-file -p 5c40
100644 blob f761ec192d9f0dca3329044b96ebdb12839dbff6    bbb.txt
```

ここで、

* 先程見た tree オブジェクト `580c` はそのまま残っている。
 * Git のリポジトリは基本的には追記のみされる。
* 新たに2つの tree オブジェクトが追加された。
 * それぞれの tree オブジェクトはディレクトリを表現している。
 * 根本の tree オブジェクト (`580c` or `6434`) から、ある時点のファイルの集合を特定できる

ということが見てとれます。

![tree-2.png](https://qiita-image-store.s3.amazonaws.com/0/64323/cdf84ea8-9a29-58f6-7588-8b1613a767e1.png)

せっかくなので tree のパーサを書いてみました。

```py:parse_tree.py
import hashlib
import sys
import zlib

if len(sys.argv) != 2:
    print("usage: %s git_object_file" % sys.argv[0])
    sys.exit(-1)

try:
    f = open(sys.argv[1])
except Exception:
    print("open %s failed" % sys.argv[1])
    sys.exit(-1)

data = zlib.decompress(f.read())
sha1 = hashlib.sha1(data).hexdigest()

eoh = data.find("\0")
if eoh < 0:
    print("no end of header")
    sys.exit(-1)

header = data[:eoh]
t, n = header.split(" ")

if len(data) - eoh - 1 != int(n):
    print("size mismatch %d,%d" % (len(data) - eoh - 1, int(n)))
    sys.exit(-1)
if t != "tree":
    print("not tree: %s" % t)
    sys.exit(-1)

dsize = hashlib.sha1().digest_size
ptr = eoh + 1
while ptr < len(data):
    eorh = data.find("\0", ptr)
    if eorh < 0:
        print("no end of reference header")
        sys.exit(-1)
    mode, name = data[ptr:eorh].split(" ")
    sha1_ = "".join(map(lambda x: "%02x" % ord(x), data[eorh+1:eorh+1+dsize]))
    print("%s (%6s) %s" % (sha1_, mode, name))
    ptr = eorh + 1 + dsize
```

```
% python parse_tree.py .git/objects/64/34b2415497a42647800c7e828038a2fb6fbbaf
72943a16fb2c8f38f9dde202b7a70ccc19c52f34 (100644) readme.txt
5c40d98927de9cdb27df5b3a7bd4f7ee95dbfc85 ( 40000) tmp
```

tree オブジェクトのデータ構造としては、(blob と同様に) zlib で圧縮されたデータの中に

* "tree " という文字列
* コンテンツのサイズ
* ヌル文字
* コンテンツ

コンテンツの部分は

* ポイントするオブジェクトの種類とファイルモード
* ファイルまたはディレクトリの名前
* ヌル文字
* ポイントするオブジェクトのSHA1ハッシュ値

の繰り返しになっています。

# コミット履歴の格納

さて、blob, tree という二種類のオブジェクトを見たので、最後に commit オブジェクトを見ましょう。
tree オブジェクト `580c` を参照するコミットを作成してみます。

```
% git commit-tree -m "initial commit" 580c
7a5c786478f17fd96b385c725c95d10fa74e4576
% ls .git/objects/7a/5c786478f17fd96b385c725c95d10fa74e4576
.git/objects/7a/5c786478f17fd96b385c725c95d10fa74e4576
% git cat-file -t 7a5c
commit
% git cat-file -p 7a5c
tree 580c73c39691399d09ad01152ad0a691ce80bccf
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1447772602 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1447772602 +0900

initial commit
```

次にその commit オブジェクト `7a5c` を親として、tree オブジェクト `6434` を参照するコミットを作成しましょう。

```
% git commit-tree -p 7a5c -m "second commit" 6434
88470d975c1875e2e03a46877c13dde9ed2fd1ea
% ls .git/objects/88/470d975c1875e2e03a46877c13dde9ed2fd1ea
.git/objects/88/470d975c1875e2e03a46877c13dde9ed2fd1ea
% git cat-file -t 8847
commit
% git cat-file -p 8847
tree 6434b2415497a42647800c7e828038a2fb6fbbaf
parent 7a5c786478f17fd96b385c725c95d10fa74e4576
author Yoichi Nakayama <yoichi.nakayama@gmail.com> 1447772754 +0900
committer Yoichi Nakayama <yoichi.nakayama@gmail.com> 1447772754 +0900

second commit
```

![commits.png](https://qiita-image-store.s3.amazonaws.com/0/64323/b241b729-5cab-5e1b-3ba2-16bcd6d7ec11.png)

HEAD の参照先の master にこの commit オブジェクトのハッシュ値を記入すると、
git log で履歴を参照することができます。

```
% cat .git/HEAD
ref: refs/heads/master
% echo 88470d975c1875e2e03a46877c13dde9ed2fd1ea > .git/refs/heads/master
% git log
commit 88470d975c1875e2e03a46877c13dde9ed2fd1ea
Author: Yoichi Nakayama <yoichi.nakayama@gmail.com>
Date:   Wed Nov 18 00:05:54 2015 +0900

    second commit

commit 7a5c786478f17fd96b385c725c95d10fa74e4576
Author: Yoichi Nakayama <yoichi.nakayama@gmail.com>
Date:   Wed Nov 18 00:03:22 2015 +0900

    initial commit
```

普段 git commit 後に見れている履歴が見れるようになりました。
git diff に対象の commit オブジェクトのハッシュ値を与えて差分を見ることもできます。

```
% git diff 7a5c 8847
diff --git a/tmp/bbb.txt b/tmp/bbb.txt
new file mode 100644
index 0000000..f761ec1
--- /dev/null
+++ b/tmp/bbb.txt
@@ -0,0 +1 @@
+bbb
```

commit オブジェクトのデータ構造は、先頭が "commit " である他は blob オブジェクトと同じで、コンテンツとして

* ファイルの集合の根本の tree オブジェクト
 * これにより対象のファイル集合全体が特定される
* 親の commit オブジェクト
* author, committer (詳細は次回見る予定)
* コミットログメッセージ

を含みます。

# まとめ

* Git では commit, tree, blob を用いて履歴が管理されている
* オブジェクトはSHA1ハッシュで識別される
 * 基本的にオブジェクトは追加されるのみ。勝手に消えない
 * ハッシュ衝突したらどうなる？→参考文献あげときます
* commit オブジェクトには次の情報が格納されている
 * 誰が、いつコミットしたか
 * コミットログメッセージ
 * 親の commit オブジェクト
 * ファイル集合
* 実際の変更内容(差分)は明示的に格納されていない
 * commit オブジェクトが指すファイル集合同士から算出される
 * ファイルの移動やコピーも算出される→間違うこともあるよ
* コミットログメッセージの内容は利用者に任せられている
 * がんばって有用な内容を書こう！
 * 有用な内容 = Git が勝手に格納してくれない情報
 * 変更内容の要約 → どんな変更が入ったかを一望できる
 * 変更を選択した背景 → 将来の改修時に制約を把握できる
 * 補足情報が課題管理システムにある場合、チケット番号を書いておくといいことあるかも
 * コミット→改修チケット→元の実装チケット→...

## リファレンス

* コマンドリファレンス
 * man git-{コマンド名}
 * https://git-scm.com/docs
* [Gitの内側 - Gitオブジェクト](https://git-scm.com/book/ja/v1/Gitの内側-Gitオブジェクト)
* [Git index format](https://www.kernel.org/pub/software/scm/git/docs/technical/index-format.txt)
* [stackoverflow - How would git handle a SHA-1 collision on a blob?](http://stackoverflow.com/questions/9392365/how-would-git-handle-a-sha-1-collision-on-a-blob)
* [永遠に未完成 - Gitはファイルの移動を追跡できない](http://d.hatena.ne.jp/thinca/20090217/1234799036)

## 続き

* [Git入門(2) 怖くないリセット](http://qiita.com/yoichi22/items/b58ea6d4058d9425c04b)
