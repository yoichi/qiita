少し前に SHA1 の衝突の話題がありました([Announcing the first SHA1 collision](https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html))。
Git はリポジトリ内のオブジェクトの識別にSHA-1ハッシュを使っており、衝突が起きたときにどういう動作になるかが気になったので調べてみました。

# blob のオブジェクトID

Git リポジトリにおいてファイルは blob という種類のオブジェクトで表現されます。まずはじめにSHA1が衝突している2つのファイルがどう扱われるか見てみましょう。

上述のリンク先からPDFファイルをダウンロードすると

* ファイルサイズが一致
* SHA1が一致
* ファイルの内容は異なる(SHA256は異なる)

という2つのファイルが入手できます。

```
~/Downloads$ ls -a
.  ..  shattered-1.pdf  shattered-2.pdf
~/Downloads$ wc -c *.pdf
422435 shattered-1.pdf
422435 shattered-2.pdf
844870 total
~/Downloads$ sha1sum *.pdf
38762cf7f55934b34d179ae6a4c80cadccbb7f0a  shattered-1.pdf
38762cf7f55934b34d179ae6a4c80cadccbb7f0a  shattered-2.pdf
~/Downloads$ sha256sum *.pdf
2bb787a73e37352f92383abe7e2902936d1059ad9f1ba6daaa9c1e58ee6970d0  shattered-1.pdf
d4488775d29bdef7993367d541064dbdda50d383f89f0aa13a6ff2e0894ba5ff  shattered-2.pdf
```

リポジトリを初期化してこれらを git add してみます。

```
~/Downloads$ git init
Initialized empty Git repository in /home/yoichi/Downloads/.git/
~/Downloads$ git add shattered-1.pdf 
~/Downloads$ find .git/objects/ -type　f
.git/objects/ba/9aaa145ccd24ef760cf31c74d8f7ca1a2e47b0
~/Downloads$ git cat-file -t ba9aaa145ccd24ef760cf31c74d8f7ca1a2e47b0
blob
~/Downloads$ git add shattered-2.pdf ~/Downloads$ find .git/objects/ -type f
.git/objects/ba/9aaa145ccd24ef760cf31c74d8f7ca1a2e47b0
.git/objects/b6/21eeccd5c7edac9b7dcba35a8d5afd075e24f2
~/Downloads$ git cat-file -t b621eeccd5c7edac9b7dcba35a8d5afd075e24f2
blob
```

それぞれが異なる blob オブジェクトとして格納されています。SHA1で管理しているのに異なるオブジェクトIDになっているのは、ファイルのコンテンツのSHA1がオブジェクトIDになっているのではなく、

* オブジェクトの種類 (今の場合 "blob ")
* コンテンツのサイズ
* ヌル文字
* コンテンツ

を合わせた blob オブジェクトのハッシュ値を取っているためです。
したがってSHA1が一致する２つのファイルがあっても、GitにおけるオブジェクトID衝突時の挙動がすぐに見れるわけではありません。

# オブジェクトID衝突時の挙動

本当にオブジェクトIDを衝突させて挙動を見るのは難しいとわかったので、Gitのソースを改変して擬似的に衝突を起こして動作を見てみます。

## SHA1の値を定数にする

SHA1のハッシュ値を計算する処理を置き換えて、常に一定の値を返すようにします。

```const_sha1.diff
diff --git a/cache.h b/cache.h
index c958269..2c7919a 100644
--- a/cache.h
+++ b/cache.h
@@ -26,10 +26,30 @@
 #define platform_SHA1_Final    	SHA1_Final
 #endif
 
+#if 0
 #define git_SHA_CTX		platform_SHA_CTX
 #define git_SHA1_Init		platform_SHA1_Init
 #define git_SHA1_Update		platform_SHA1_Update
 #define git_SHA1_Final		platform_SHA1_Final
+#else
+typedef struct git_SHA_CTX {
+	unsigned data[20];
+} git_SHA_CTX;
+inline void git_SHA1_Init(git_SHA_CTX *c)
+{
+	for (int i = 0; i < 20; i++)
+		c->data[i] = i;
+}
+inline void git_SHA1_Update(git_SHA_CTX *c, const void *data, unsigned long len)
+{
+}
+inline void git_SHA1_Final(unsigned char *sha1, git_SHA_CTX *c)
+{
+	for (int i = 0; i < 20; i++) {
+		sha1[i] = c->data[i];
+	}
+}
+#endif
 
 #ifdef SHA1_MAX_BLOCK_SIZE
 #include "compat/sha1-chunked.h"
```

と変更した上でmakeして、挙動を確認してみます。

### blob に対して tree が衝突した場合

一つのファイルを git add した上でコミットすると

* git add 時に blob オブジェクトを生成
* git commit 時に
 * tree オブジェクトを生成
 * commit オブジェクトを生成

という流れで処理が進むはずですが、 commit オブジェクトを生成しようとしたところで、見てるものが tree オブジェクトじゃないよというエラーになります。

```
~/prog/tmp$ ls -a
.  ..
~/prog/tmp$ ../git/git-init
warning: templates not found /home/yoichi/share/git-core/templates
Initialized empty Git repository in /home/yoichi/prog/tmp/.git/
~/prog/tmp$ touch a
~/prog/tmp$ ../git/git-add a
~/prog/tmp$ find .git/objects/ -type f
.git/objects/00/0102030405060708090a0b0c0d0e0f10111213
~/prog/tmp$ git cat-file -t 000102030405060708090a0b0c0d0e0f10111213
blob
~/prog/tmp$ ../git/git-commit -m "msg"
fatal: 000102030405060708090a0b0c0d0e0f10111213 is not a valid 'tree' object
~/prog/tmp$ ../git/git-cat-file -t 000102030405060708090a0b0c0d0e0f10111213
blob
```
blob オブジェクトを生成した後に、tree オブジェクトを生成したつもりで先に進もうとしているが、実際には元々そのオブジェクトIDで生成されていた blob オブジェクトが維持されていて、不整合を検出しているようです。

### tree に対して commit が衝突した場合

空のコミットを生成しようとすると

* git commit 時に
 * tree オブジェクトを生成
 * commit オブジェクトを生成

という流れで処理が進むはずですが、commit オブジェクトを生成した上で refs を更新しようとしたところで、見てるものが commit オブジェクトじゃないよというエラーになります。

```
~/prog/tmp$ ls -a
.  ..
~/prog/tmp$ ../git/git-init
warning: templates not found /home/yoichi/share/git-core/templates
Initialized empty Git repository in /home/yoichi/prog/tmp/.git/
~/prog/tmp$ ../git/git-commit -m "msg" --allow-empty
fatal: cannot update ref 'refs/heads/master': trying to write non-commit object 000102030405060708090a0b0c0d0e0f10111213 to branch 'refs/heads/master'
~/prog/tmp$ find .git/objects/ -type f
.git/objects/00/0102030405060708090a0b0c0d0e0f10111213
~/prog/tmp$ ../git/git-cat-file -t 000102030405060708090a0b0c0d0e0f10111213
tree
```

tree オブジェクトを生成した後に、commit オブジェクトを生成したつもりで先に進もうとしているが、実際には元々そのオブジェクトIDで生成されていた tree オブジェクトが維持されていて、不整合を検出しているようです。

### tree に対して blob が衝突した場合

「tree に対して commit が衝突した場合」の手順で tree がある状態から git add すると、特にエラーになりませんが、 tree オブジェクトがそのまま維持されています。

```
~/prog/tmp$ touch a
~/prog/tmp$ ../git/git-add a
~/prog/tmp$ find .git/objects/ -type f
.git/objects/00/0102030405060708090a0b0c0d0e0f10111213
~/prog/tmp$ ../git/git-cat-file -t 000102030405060708090a0b0c0d0e0f10111213
tree
```

インデックスには a が追加されていて、本来は blob オブジェクトであるべきですが、 tree オブジェクトを指してしまっています。

```
~/prog/tmp$ ../git/git-ls-files --stage
100644 000102030405060708090a0b0c0d0e0f10111213 0	a
```

不整合なインデックスを元にコミットするとどうなるかが気になりますが、今回のソース改変ではコミットできないので、実際にどうなるかまでは確認できませんでした。

## SHA1の値を種類ごとの定数にする

次はオブジェクトの種類のチェックを回避するため、種類ごとの定数になるよう改変してみます。オブジェクトの先頭4バイトがオブジェクトの種類を表しているのでそれをそのままハッシュ値として返すようにしましょう。

```per_type_sha1.diff
diff --git a/cache.h b/cache.h
index c958269..09b9670 100644
--- a/cache.h
+++ b/cache.h
@@ -26,10 +26,41 @@
 #define platform_SHA1_Final    	SHA1_Final
 #endif
 
+#if 0
 #define git_SHA_CTX		platform_SHA_CTX
 #define git_SHA1_Init		platform_SHA1_Init
 #define git_SHA1_Update		platform_SHA1_Update
 #define git_SHA1_Final		platform_SHA1_Final
+#else
+typedef struct git_SHA_CTX {
+	int lim;
+	int idx;
+	unsigned data[20];
+} git_SHA_CTX;
+inline void git_SHA1_Init(git_SHA_CTX *c)
+{
+	c->lim = 4;
+	c->idx = 0;
+	for (int i = 0; i < 20; i++)
+		c->data[i] = 0;
+}
+inline void git_SHA1_Update(git_SHA_CTX *c, const void *data, unsigned long len)
+{
+	int i = 0;
+	while (c->idx < c->lim && i < len)
+	{
+		c->data[c->idx] = ((unsigned char*)data)[i];
+		c->idx++;
+		i++;
+	}
+}
+inline void git_SHA1_Final(unsigned char *sha1, git_SHA_CTX *c)
+{
+	for (int i = 0; i < 20; i++) {
+		sha1[i] = c->data[i];
+	}
+}
+#endif
 
 #ifdef SHA1_MAX_BLOCK_SIZE
 #include "compat/sha1-chunked.h"
```

### commit に対して commit が衝突した場合

２つの空のコミットをした場合、

* tree オブジェクトを生成
* 1つ目の commit オブジェクトを生成
* 1つ目の commit オブジェクトを親とする2つめの commit オブジェクトを生成

という流れで処理が進むはずですが、

```
~/prog/tmp$ ls -a
.  ..
~/prog/tmp$ ../git/git-init
warning: templates not found /home/yoichi/share/git-core/templates
Initialized empty Git repository in /home/yoichi/prog/tmp/.git/
~/prog/tmp$ ../git/git-commit -m "msg-first" --allow-empty
[master (root-commit) 636f6d6] msg-first
~/prog/tmp$ find .git/objects/ -type f
.git/objects/74/72656500000000000000000000000000000000
.git/objects/63/6f6d6d00000000000000000000000000000000
~/prog/tmp$ ../git/git-cat-file -t 7472656500000000000000000000000000000000
tree
~/prog/tmp$ ../git/git-cat-file -t 636f6d6d00000000000000000000000000000000
commit
~/prog/tmp$ ../git/git-log --oneline
636f6d6 msg-first
~/prog/tmp$ ../git/git-commit -m "msg-second" --allow-empty
[master 636f6d6] msg-first
```

2つ目のコミットは成功しているのに、本来表示されるべき msg-second ではなく、元々あった 1つ目のコミットが表示されています。新しいコミットはリポジトリに保存されずにコミット成功し、HEADは元からあったコミットを指しているようです。

# まとめ

以上から、Gitでハッシュが衝突したときの動作は

* オブジェクト書き込み時にハッシュが衝突した場合には
 * 書き込み自体はエラーとしない
 * 同じハッシュを持つ元のオブジェクトが維持される（上書きしない）
* 後続処理のオブジェクト読み込み時に
 * 種類が不整合ならエラーになる
 * 種類が整合していればエラーにならない

となっており、

* ハッシュが一致すれば実体も同じとみなす
* 一度リポジトリに格納したものは基本消さない

というポリシーに沿ったものと考えられます。
