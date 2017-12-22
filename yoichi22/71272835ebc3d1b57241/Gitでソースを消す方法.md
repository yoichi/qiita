# 目的

* Gitの履歴の保持方法を知る
* ソース（の差分）を消失する方法を知る
* 意図せず消さないための注意点を考える

---
# Git

* 分散型のバージョン管理システム
* 作者: [Linus Torvalds](https://jp.linux.com/news/linuxcom-exclusive/428524-lco2015041401)
* メンテナ: [濱野純](http://gihyo.jp/dev/serial/01/alpha-geek/0040)
* 元々 Linux kernel のソース管理の為に作られた

---
# 履歴

ImmutableなコミットのDAG(Directed ascyclic graph, 有向非巡回グラフ) で履歴を管理

![dag.png](https://qiita-image-store.s3.amazonaws.com/0/64323/0a087a20-40a8-3036-9a8e-646d8e3597bf.png)

---
# コミット

各々のコミットは、ソースツリーを参照している(ImmutableなTree, Blobで構成される)

![commit.png](https://qiita-image-store.s3.amazonaws.com/0/64323/342f9839-d281-a6a6-e304-f205794e32f9.png)

---
# リポジトリの構造

* Immutableなオブジェクトが格納されてる
  * 操作によりオブジェクトが追加される
  * 記録した情報は基本消えない
* オブジェクト間の参照
  * コミットは親コミットを参照
  * コミットはルートツリーを参照
  * Treeは子のTree,Blobたちを参照
* ブランチ、タグはコミットへのポインタ

---
# 基本的な操作

`git add ${FILES} && git commit`

* Tree, Blob たちを記録
* ルートツリーと親コミットを参照する Commit を記録

過去に記録したものには手を加えない

---
# いろいろな操作

* 過去のある時点のソースコードを取得する
  * `git checkout ${COMMIT_SHA1}`
* ファイルを消してコミット
  * `git rm ${FILE} && git commit`
* 過去の変更を打ち消すコミットを生成する
  * `git revert ${COMMIT_SHA1}`

これらも安全な操作（リポジトリから何も消さない）

---
# 安全側に倒した設計

ソース（の差分）が消える操作を（オプションもしくは事前条件で）抑止

例: 変更を加えているファイルが checkout で更新されそう

```
$ git checkout work
error: Your local changes to the following files would be overwritten by checkout:
	readme.txt
Please commit your changes or stash them before you switch branches.
Aborting
```

---
# 何も消えないの？

* .git/ (リポジトリ)を壊さない範囲で
* ミスで消えないよう配慮はされている
* これから消し方を説明します！
  * そもそも記録してない情報は消える
  * 記録したものも消そうと思えば消せる

---
# 記録してない情報

* git add してないファイル
* git add してない差分

これらはリポジトリに入っていないので消せる

---
# 管理してないファイルを消す

```
$ git clean
fatal: clean.requireForce defaults to true and neither -i, -n, nor -f given; refusing to clean
```

デフォルト(clean.requireForce=true)では引数なしは弾かれる

* `-n` で何が消えるか確認
* `-f` で実際に消す
* `-d` とか `-x` もよく使う
* ビルドで生成されたファイルを消すときなど

---
# ファイルの変更を消す

```
$ git checkout ${PATH}
```

* 指定したファイルは index と同じ内容になる
* HEAD にするなら `git checkout HEAD ${PATH}`
* 変更しかけた1ファイルを元に戻す

---
# ファイルたちの変更を消す

```
$ git reset --hard
```

* HEAD と同じ内容になる
* 変更しかけたファイルたち(と index)を元に戻す

---
# ブランチの削除

```
$ git branch -d work
error: The branch 'work' is not fully merged.
If you are sure you want to delete it, run 'git branch -D work'.
```

* upstream もしくは HEAD にマージされてなければ一旦弾いてくれる
* どうすれば迂回できるかはメッセージに出てる

それでも先に進む？

---
# ブランチの削除

`-D` または `-d -f` でブランチを削除

* 消えるのはブランチだけ
* コミットはその時点では消えない
* `git gc` (GC = garbage collection)
* どこからも参照されていないコミットは gc で消える
  * 参照されていれば消えない

---
# 参照されていれば消えない

* いろいろやり直せることも思い出す
* ローカルブランチ気軽に使おう
* 例えば以下の手順でバックアップ作れる

```
$ git add ${FILES}
$ git commit
$ git branch backup
$ git reset HEAD~1
```

---
# リモートの話

```
$ git push ${REMOTE} ${BRANCH}
```

* リモートに履歴をバックアップできる
* ローカルで消えても戻せる可能性
* 手元の .git/ が壊れた場合にも助かるかも

---
# まとめ

* git を使ったソースの消し方を見た
* 基本的には履歴を安全に保持してくれる
* 消える可能性があるのは
  * コミットしてないもの
  * タグやブランチから参照されてないもの
* 知った上で使えば大丈夫
  * 必要に応じてバックアップするとよいかも
