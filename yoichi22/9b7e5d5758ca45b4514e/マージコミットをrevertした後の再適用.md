[GitLab flow](https://postd.cc/gitlab-flow/)で運用していて、リリースしたものに問題があったのでproductionブランチ上でマージコミットをrevertしてリリースしたら、その後の修正リリースでどうすればよいか悩むことになりました。対処した方法と、そもそもロールバックリリースの方法が間違っていたということを説明します。

## 課題

* 前提：GitLab flowで運用
  * masterブランチからfeature branchを分岐して開発
  * productionブランチのコミットがデプロイ対象
* 状況
  * feature A, Bをマージしてリリースした
  * 問題があったので、productionブランチでrevertを行いロールバックした
* やりたいこと
  * feature Aは問題ないことがわかったので、まずそれのみ再リリースしたい
  * feature Bは問題があったので再修正の上で次にリリースしたい

## 準備

説明のためのサンプルリポジトリをセットアップします。

リポジトリ新規作成

```
revert_and_revert$ git init
Initialized empty Git repository in /Users/yoichi/prog/git-study/revert_and_revert/.git/
revert_and_revert(master)$ git commit --allow-empty -m "initialize repository with empty commit"
[master (root-commit) a48347d] initialize repository with empty commit
revert_and_revert(master)$ git branch production
```

各featureを開発し、masterにマージ

```
revert_and_revert(master)$ git branch feat_a
revert_and_revert(master)$ git branch feat_b
revert_and_revert(master)$ git checkout feat_a && echo feat_a > a.txt && git add a.txt && git commit -m 'implement feat_a'
Switched to branch 'feat_a'
[feat_a 304db62] implement feat_a
 1 file changed, 1 insertion(+)
 create mode 100644 a.txt
revert_and_revert(feat_a)$ git checkout feat_b && echo feat_b_with_defect > b.txt && git add b.txt && git commit -m 'implement feat_b'
Switched to branch 'feat_b'
[feat_b 5465c73] implement feat_b
 1 file changed, 1 insertion(+)
 create mode 100644 b.txt
revert_and_revert(feat_b)$ git checkout master
Switched to branch ‘master'
revert_and_revert(master)$ git merge --no-ff --no-edit feat_a
Merge made by the 'recursive' strategy.
 a.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 a.txt
revert_and_revert(master)$ git merge --no-ff --no-edit feat_b
Merge made by the 'recursive' strategy.
 b.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 b.txt
revert_and_revert(master)$ cat a.txt
feat_a
revert_and_revert(master)$ cat b.txt
feat_b_with_defect
```

リリース

```
revert_and_revert(master)$ git checkout production
Switched to branch 'production'
revert_and_revert(production)$ git merge --no-ff --no-edit master
Merge made by the 'recursive' strategy.
 a.txt | 1 +
 b.txt | 1 +
 2 files changed, 2 insertions(+)
 create mode 100644 a.txt
 create mode 100644 b.txt
```

ロールバックリリース

```
revert_and_revert(production)$ git revert --no-edit -m1 HEAD
[production 9feb753] Revert "Merge branch 'master' into production"
 Date: Sun Jun 17 06:12:20 2018 +0900
 2 files changed, 2 deletions(-)
 delete mode 100644 a.txt
 delete mode 100644 b.txt
```

## うまくいかなかったやり方

masterブランチでA以外の機能をロールバック

```
revert_and_revert(production)$ git checkout master
Switched to branch 'master'
revert_and_revert(master)$ git log --first-parent --oneline
57f9496 (HEAD -> master) Merge branch 'feat_b'
5871e26 Merge branch 'feat_a'
a48347d initialize repository with empty commit
revert_and_revert(master)$ git revert --no-edit -m1 57f9496
[master 38d6b46] Revert "Merge branch 'feat_b'"
 Date: Sun Jun 17 06:13:22 2018 +0900
 1 file changed, 1 deletion(-)
 delete mode 100644 b.txt
```

productionブランチへマージ→おや？何も入らないぞ。

```
revert_and_revert(master)$ git checkout production
Switched to branch ‘production'
revert_and_revert(production)$ git merge --no-ff master
Merge made by the 'recursive' strategy.
```

## うまくいったやり方

productionブランチからpre_productionブランチを分岐し、マージコミットのrevertのrevertをした上でmasterをマージ

```
revert_and_revert(production)$ git checkout -b pre_production
Switched to a new branch 'pre_production'
revert_and_revert(pre_production)$ git revert --no-edit HEAD
[pre_production 64d00fe] Revert "Revert "Merge branch 'master' into production""
 Date: Sun Jun 17 06:16:17 2018 +0900
 2 files changed, 2 insertions(+)
 create mode 100644 a.txt
 create mode 100644 b.txt
revert_and_revert(pre_production)$ git merge --no-edit --no-ff master
Removing b.txt
Merge made by the 'recursive' strategy.
 b.txt | 1 -
 1 file changed, 1 deletion(-)
 delete mode 100644 b.txt
```

productionブランチにそれをマージ→Aのみが入っているのでこれで再リリースできる。

```
revert_and_revert(pre_production)$ git checkout production
Switched to branch 'production'
revert_and_revert(production)$ git merge --no-edit --no-ff pre_production
Merge made by the 'recursive' strategy.
 a.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 a.txt
revert_and_revert(production)$ ls
a.txt
revert_and_revert(production)$ cat a.txt
feat_a
```

次のリリースに向けて、masterから分岐して、Bのrevertをrevertの上、修正をコミット。

```
revert_and_revert(production)$ git checkout master
Switched to branch 'master'
revert_and_revert(master)$ git checkout -b fix_b
Switched to a new branch 'fix_b'
revert_and_revert(fix_b)$ git log --first-parent --oneline
38d6b46 (HEAD -> fix_b, master) Revert "Merge branch 'feat_b'"
57f9496 Merge branch 'feat_b'
5871e26 Merge branch 'feat_a'
a48347d initialize repository with empty commit
revert_and_revert(fix_b)$ git revert --no-edit 38d6b46
[fix_b 0efb2fb] Revert "Revert "Merge branch 'feat_b'""
 Date: Sun Jun 17 06:19:52 2018 +0900
 1 file changed, 1 insertion(+)
 create mode 100644 b.txt
revert_and_revert(fix_b)$ cat b.txt
feat_b_with_defect
revert_and_revert(fix_b)$ echo feat_b > b.txt
revert_and_revert(fix_b)$ git commit -a -m "fix feat_b"
[fix_b 56146b3] fix feat_b
 1 file changed, 1 insertion(+), 1 deletion(-)
```

masterブランチにマージ

```
revert_and_revert(fix_b)$ git checkout master
Switched to branch 'master'
revert_and_revert(master)$ git merge --no-edit --no-ff fix_b
Merge made by the 'recursive' strategy.
 b.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 b.txt
revert_and_revert(master)$ cat b.txt
feat_b
```

productionブランチにマージすれば、修正版のBを含む次のリリースができる。

```
revert_and_revert(master)$ git checkout production
Switched to branch 'production'
revert_and_revert(production)$ git merge --no-edit --no-ff master
Merge made by the 'recursive' strategy.
 b.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 b.txt
revert_and_revert(production)$ cat b.txt
feat_b
```


## そもそも

GitLab flow ではマージを masterブランチ→productionブランチ の方向のみに限定するのが原則なので、それに反して production ブランチのみにあるコミット（マージコミット）をrevertした時点で負けで、単純なマージでproductionブランチに変更が入らない状態になってしまいました。

正解は、変更は常にmasterブランチの側で行うこと、つまりロールバックリリースの際もproductionブランチではなくmasterから分岐したブランチでロールバックし、それをproductionブランチへマージしてリリース、としていればよかったのだと思います。
