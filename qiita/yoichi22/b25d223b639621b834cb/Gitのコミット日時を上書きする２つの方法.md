## コミット日時を上書きしたい

Gitのコミットの、差分とコミットメッセージは維持しつつ、コミット日時を書き換えたいことが稀にある。
こないだあったのは、テストのためにローカルホストの日時を未来に変更したままでコミットしてしまい、「未来のコミットになってますよw」という指摘がコードレビューで出たとき。

## 普通にamendしても上書きされない

通常のamendでは日付は書き換わらない。

```
$ GIT_PAGER= git log -1
commit 64c414eaa5084c765c79d2e061ced3d5724d00dd (HEAD -> master)
Author: foo <foo@example.com>
Date:   Sun Sep 29 09:59:00 2019 +0900

    an example
$ date
日  9 29 10:02:03 JST 2019
$ git commit --amend
[master 8f3cf01] an example
 Date: Sun Sep 29 09:59:00 2019 +0900
 1 file changed, 0 insertions(+), 0 deletions(-)
 rename a => b (100%)
$ GIT_PAGER= git log -1
commit 8f3cf018573ad002c906e00bfa2facd38fc276ca (HEAD -> master)
Author: foo <foo@example.com>
Date:   Sun Sep 29 09:59:00 2019 +0900

    an example
```

これは、コミットには２つの日付 AuthorDate と CommitDate が記録されており、amendでは CommitDate の方が更新されるからである。実際、CommitDateの方は更新されている：

```
$ GIT_PAGER= git log -1 --pretty=fuller
commit 8f3cf018573ad002c906e00bfa2facd38fc276ca (HEAD -> master)
Author:     foo <foo@example.com>
AuthorDate: Sun Sep 29 09:59:00 2019 +0900
Commit:     foo <foo@example.com>
CommitDate: Sun Sep 29 10:02:08 2019 +0900

    an example
```

## 方法1: reset-authorオプション

お手軽なのは reset-author オプションを使うこと。以下のようにするとAuthorDateが現在日時になる

```
$ git commit --amend --reset-author
```

その名の通り、Authorもリセットされてしまうので、そうしたくない場合には使えない。
ちなみに reset-author オプションと一緒に author オプションを指定するとエラーになる。

```
$ git commit --amend --reset-author --author "foo <foo@example.com>"
fatal: Using both --reset-author and --author does not make sense
```

また、この方法だと、現在日時とすることしかできない。

## 方法2: dateオプション

date オプションで日時を指定するとそれが AuthorDate に設定される。例えば現在日時とするには

```
git commit --amend --date $(date --iso-8601=seconds)
```

日時の値を自分で設定すれば任意の日時にすることも可能。ただしこれで設定されるのは、AuthorDateであり、CommitDateの方は現在日時になる。CommitDateも合わせたい場合は、その後でgit rebaseのcommitter-date-is-author-dateオプションを使えば良い。

```
$ GIT_PAGER= git log -1 --pretty=fuller
commit 96fdf18c0d2580dacf6447d2669124684327de63 (HEAD -> master)
Author:     foo <foo@example.com>
AuthorDate: Mon Sep 30 06:14:11 2019 +0900
Commit:     foo <foo@example.com>
CommitDate: Sun Sep 29 10:14:11 2019 +0900

    an example
$ git rebase HEAD~ --committer-date-is-author-date
Current branch master is up to date, rebase forced.
First, rewinding head to replay your work on top of it...
Applying: an example
$ GIT_PAGER= git log -1 --pretty=fuller
commit c7fa00c0231b4a5505bd6fae2460dc8a65d08eb1 (HEAD -> master)
Author:     foo <foo@example.com>
AuthorDate: Mon Sep 30 06:14:11 2019 +0900
Commit:     foo <foo@example.com>
CommitDate: Mon Sep 30 06:14:11 2019 +0900

    an example
```

## 二つ以上前のコミットを対象にするには

`git rebase -i` で対象コミットをeditとして、amend commitの際に上で説明したいずれかの方法を使えば良い。

## 確認した環境

```
$ git --version
git version 2.19.1
$ date --version
date (GNU coreutils) 8.29
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by David MacKenzie.
```
