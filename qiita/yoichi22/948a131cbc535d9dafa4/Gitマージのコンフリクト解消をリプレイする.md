## コンフリクト解消でありがちなミス

Gitのマージでコンフリクトが発生すると焦りますよね？

コンフリクト解消でありがちなミスとして、コンフリクトしたファイルのみに気をとられてしまい、マージするブランチで加えていた別のファイルの変更をコミットし忘れることがあります。[^1]

[^1]: CUIでgitコマンドを使っている場合は込み入った操作をしないとそうはならないのですが、IDE上でマージした場合に割とやってしまうようです。

起きてしまったミスをただ悔やんでも仕方がありません。どういうことをしてしまったのかを正確に把握し、ふりかえって学ぶことが大切です。そこで、コンフリクトの解消をリプレイして、どういうことをしてしまったのかを確認してみましょう。

## マージを再現する

マージコミットには、通常2つの親コミットが付随しています。[^2]

[^2]: コンフリクト解消ミスの例: https://github.com/yoichi/sandbox/commit/c84ed0117bec79caee974b3d799926d408e5c22b

![merge-commit.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/64323/58a59b01-e2a7-f7c5-4e73-5149c0f18f0f.png)

この例では、マージコミット c84ed01 に対して 3fc67bf と 13bab57 が親コミットです。

```
sandbox(conflict-demo-master)$ git log -1
commit c84ed0117bec79caee974b3d799926d408e5c22b (HEAD -> conflict-demo-master, origin/conflict-demo-master)
Merge: 3fc67bf 13bab57
Author: Yoichi Nakayama <yoichi.nakayama@gmail.com>
Date:   Fri May 1 21:25:51 2020 +0900

    Merge branch 'conflict-demo-topic' into conflict-demo-master

    Conflicts:
            a.txt
```

１つ目の親(上の例では 3fc67bf) をチェックアウトし、2つ目の親(上の例では 13bab57)を指定することで、マージを再現することができます。

```
sandbox(conflict-demo-master)$ git checkout 3fc67bf
Note: switching to '3fc67bf'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

HEAD is now at 3fc67bf change in master branch
sandbox((3fc67bf...))$ git merge 13bab57
Auto-merging a.txt
CONFLICT (content): Merge conflict in a.txt
Automatic merge failed; fix conflicts and then commit the result.
sandbox((3fc67bf...)|MERGING)$
```

マージしてコンフリクトした状態が再現できました。

## コンフリクト解消した後との差分を見る

まず、マージ操作によって加えられた変更を見てみましょう。マージ直前のコミットは HEAD で参照できるので、それと比較します。

```
sandbox((3fc67bf...)|MERGING)$ git diff HEAD
diff --git a/a.txt b/a.txt
index 609d907..e6fa5ab 100644
--- a/a.txt
+++ b/a.txt
@@ -1 +1,5 @@
+<<<<<<< HEAD
 change-in-master-branch
+=======
+change-in-topic-branch
+>>>>>>> 13bab57
diff --git a/b.txt b/b.txt
index e69de29..0cb8ef1 100644
--- a/b.txt
+++ b/b.txt
@@ -0,0 +1 @@
+change-in-topic-branch
```

マージしてきたブランチにあった a.txt, b.txt の変更と、コンフリクトが発生したファイルではコンフリクトマーカー("<<<<<<< HEAD", "=======", ">>>>>>> 13bab57")が挿入されています。

次に、作業ツリーから、コンフリクト解消したコミットまでの差分を確認しましょう。 マージコミットのコミットIDを指定し、そこへの差分を見るため -R オプションを付けます。

```
sandbox((3fc67bf...)|MERGING)$ git diff -R c84ed01
diff --git b/a.txt a/a.txt
index e6fa5ab..1172d0b 100644
--- b/a.txt
+++ a/a.txt
@@ -1,5 +1,2 @@
-<<<<<<< HEAD
 change-in-master-branch
-=======
 change-in-topic-branch
->>>>>>> 13bab57
diff --git b/b.txt a/b.txt
index 0cb8ef1..e69de29 100644
--- b/b.txt
+++ a/b.txt
@@ -1 +0,0 @@
-change-in-topic-branch
```

a.txt のコンフリクトマーカーの削除に加えて、マージしてきたブランチで b.txt に加えた変更が消されてしまっていたこと [^3] が分かりました。

[^3]: サンプルに仕込んでいたコンフリクト解消ミスです。

## まとめ

* Gitのコミットにはマージでコンフリクトが発生したかどうかの情報は残っていない
* マージコミットの親コミットの情報を使うとマージ操作をリプレイできる
* マージ直前からの差分、マージコミットへの差分を見ると、マージとコンフリクト解消で何をしたかを検証できる

