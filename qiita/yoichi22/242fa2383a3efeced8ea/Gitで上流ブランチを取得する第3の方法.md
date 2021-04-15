## 概要

`git pull` した時にマージされるブランチ、いわゆる「上流ブランチ」が何かを知りたい時、`git status -sb` の出力を読んだり、ファイル `.git/config` の中身を読めばわかるのですが、そんなことしなくても、

```
$ git rev-parse --abbrev-ref @{u}
```

すれば、わかるんです。

# `git status -sb`

`-s` はshort formatで出力するためのオプション、`-b` はshort formatにおいてブランチ情報を出力するためのオプションで、出力の一行目にブランチの情報が現れます。

上流ブランチが設定されてない場合にはローカルブランチ名だけが出力されます。

```
$ git status -sb | head -1
## master
```

上流ブランチが設定されている場合にはローカルブランチ名の後に"..."、そのあとに上流ブランチ名が出力されます。

```
$ git status -sb
## master...origin/master
```

上流ブランチと同期されてない場合には、上流ブランチ名の後に、ローカルブランチと上流ブランチがコミットいくつ分離れているかが出力されます。上流ブランチより遅れている場合は

```
$ git status -sb
## master...origin/master [behind 7]
```

のように出力され、この状態で `git pull` すると origin/master のコミット7つがfast-forwardでマージできます。

```
$ git log --oneline master...origin/master|wc -l
       7
```

逆に手元のブランチの方が進んでいる場合は

```
$ git status -sb
## master...origin/master [ahead 1]
```

と出力され、この場合は git push するとローカルブランチと上流ブランチが同期されます。

一方、ローカルブランチと上流ブランチが分岐している場合、つまり `git pull --ff-only` が失敗し、`git pull` でマージコミットが生成される状況では

```
$ git status -sb
## master...origin/master [ahead 1, behind 7]
```

のように、ローカルブランチと上流ブランチの、共通の祖先を経由した距離が出力されます。

```
$ git log --oneline master...origin/master|wc -l
       8
```

また、設定された上流ブランチが削除されるなどして存在しなくなった場合には、

```
$ git status -sb
## feature...origin/feature [gone]
```

のようにgoneと出力されます。

## `.git/config`

master の上流ブランチが origin/master である場合、`.git/config` には以下の内容が記載されています。

```
[branch "master"]
        remote = origin
        merge = refs/heads/master
```

この値は以下のようにしても読み出せます。

```
$ git config branch.master.remote
origin
$ git config branch.master.merge
refs/heads/master
```

ここで branch.master.merge の値 refs/heads/master はリモートリポジトリでの表現であり、ローカルリポジトリでは refs/remotes/origin/master で表現されることに注意が必要です。

## `@{u}`


[gitrevisions(7)](https://git-scm.com/docs/gitrevisions) に説明があるように、`@{u}` もしくは `@{upstream}` で上流ブランチを参照できます。

```
$ git rev-parse --abbrev-ref @{u}
origin/master
$ git rev-parse --abbrev-ref @{upstream}
origin/master
```

先に説明した二つの方法では上流ブランチの表現を得るには出力を加工する必要がありましたが、この方法だとそのまま出力してくれるので簡単です。

