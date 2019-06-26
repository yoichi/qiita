# まとめ

パスワードとか入ったファイルをコミットしちゃった時の対処で「git rm じゃ駄目、git filter-branch 使え」という記事を何度か目にしてる。わかってる人はわかってると思うけど補足。

* git rm して commit しても履歴には残ってる
* git filter-branch でローカルのrefsから辿れる履歴から消すことはできる
  * ローカルリポジトリには commit は残ってるので reflog から辿って戻せる
* push してたら force push してもリモートリポジトリには残ってる
  * force push するまでに他の人が fetch してなかったとしても取り出せる可能性がある

セキュリティの観点では最後のが気になると思うので、リモートが GitHub の場合に実験してみる。

# 準備

## 「パスワード」が入ったファイルをプッシュする

```
$ echo "secret password" > password.txt
$ git add password.txt
$ git commit -m "add password.txt"
[master 661598e] add password.txt
 1 file changed, 1 insertion(+)
 create mode 100644 password.txt
$ git rev-parse HEAD
661598e05d5e5faef7c1fc75a4f3c5a2c88368a1
$ git push origin master
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 4 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 305 bytes | 305.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To github.com:yoichi/test_fetch_sha1.git
   43fe66d..661598e  master -> master
```

## filter-branch して force-push

```
$ git filter-branch --tree-filter "rm -f password.txt"
Rewrite 661598e05d5e5faef7c1fc75a4f3c5a2c88368a1 (2/2) (0 seconds passed, remaining 0 predicted)
Ref 'refs/heads/master' was rewritten
$ git push origin master --force
Enumerating objects: 1, done.
Counting objects: 100% (1/1), done.
Writing objects: 100% (1/1), 196 bytes | 196.00 KiB/s, done.
Total 1 (delta 0), reused 0 (delta 0)
To github.com:yoichi/test_fetch_sha1.git
 + 661598e...7568dd0 master -> master (forced update)
```

# clone し直して checkout

clone し直した時、 refs から辿れないコミットは取得されない。でもこれはリモートで消えていることの根拠にはならない。

```
$ git clone git@github.com:yoichi/test_fetch_sha1.git
Cloning into 'test_fetch_sha1'...
remote: Enumerating objects: 4, done.
remote: Counting objects: 100% (4/4), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 4 (delta 1), reused 3 (delta 0), pack-reused 0
Receiving objects: 100% (4/4), done.
Resolving deltas: 100% (1/1), done.
$ git checkout 661598e05d5e5faef7c1fc75a4f3c5a2c88368a1
fatal: reference is not a tree: 661598e05d5e5faef7c1fc75a4f3c5a2c88368a1
```

# fetch sha1

git fetch で SHA1 の値を与えても弾かれる。でもこれはリモートで消えていることの根拠にはならない。

```
$ git fetch origin 661598e05d5e5faef7c1fc75a4f3c5a2c88368a1
error: Server does not allow request for unadvertised object 661598e05d5e5faef7c1fc75a4f3c5a2c88368a1
```

# GitHub の commit のページを開く

あ、見えた

https://github.com/yoichi/test_fetch_sha1/commit/661598e05d5e5faef7c1fc75a4f3c5a2c88368a1

![スクリーンショット 2019-03-03 14.23.15.png](https://qiita-image-store.s3.amazonaws.com/0/64323/f8a76d84-d95b-b0e6-7470-9f1d2d64330d.png)

# じゃあどうすればいいの？

履歴から消すことは二の次で、誤ってプッシュしてしまった情報自体を使えないものにする（パスワード変更するとか）のが第一です。
以下のヘルプの「Warning」のところも参照のこと
https://help.github.com/en/articles/removing-sensitive-data-from-a-repository
