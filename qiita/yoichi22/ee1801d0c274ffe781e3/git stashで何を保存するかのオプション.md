# 概要

git stashのオプション指定によってどう挙動が変わるか確認、整理します。[^1]

[^1]: [git pullでerror: The following untracked …やYour local changes…が出たときの原因と対処](https://iucstscui.hatenablog.com/entry/2020/05/15/090631) で git stash の -a オプションを知ったのがきっかけです。

git version 2.26.2 で確認。

# オプションなし

## リポジトリにあるファイルの変更

保存される。

```
% git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README

no changes added to commit (use "git add" and/or "git commit -a")
% git stash
Saved working directory and index state WIP on master: 2be296f add
% git status
On branch master
nothing to commit, working tree clean
% git stash pop
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (c3341440d3d936bcde11b70521bfe50b6c8cf289)
```

## ステージされた変更

保存される。popするとアンステージされた状態に戻る。
（git stash pop に --index オプションを付けるとステージされた状態に戻る）

```
% git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   README

% git stash
Saved working directory and index state WIP on master: 2be296f add
% git status
On branch master
nothing to commit, working tree clean
% git stash pop
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (6f7975d49efb05229d126100234347dacce702ff)
% git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README

no changes added to commit (use "git add" and/or "git commit -a")
```
## ステージされた新規ファイル

保存される。popするとステージされた状態に戻る。

```
% git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   NEWS

% git stash
Saved working directory and index state WIP on master: 2be296f add
% git status
On branch master
nothing to commit, working tree clean
% git stash pop
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   NEWS

Dropped refs/stash@{0} (06f0eade21e57da6d18f3b0b4e9672d8f37b324d)
% git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   NEWS
```

## 追跡されてないファイル

保存されない。

```
% git status
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	NEWS

nothing added to commit but untracked files present (use "git add" to track)
% git stash
No local changes to save
```

## 無視指定されてるファイル

保存されない。

```
% git status --ignored
On branch master
Ignored files:
  (use "git add -f <file>..." to include in what will be committed)
	README~

nothing to commit, working tree clean
% git stash
No local changes to save
```

# -u オプションを付けた場合

## 追跡されてないファイル

保存される。popすると追跡されてない状態に戻る。

```
% git status
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	NEWS

nothing added to commit but untracked files present (use "git add" to track)
% git stash -u
Saved working directory and index state WIP on master: 2be296f add
% git status
On branch master
nothing to commit, working tree clean
% git stash pop
Already up to date!
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	NEWS

nothing added to commit but untracked files present (use "git add" to track)
Dropped refs/stash@{0} (4e498a9757baa17fbed2e558f9d5a833c3be6ed7)
```

## 無視指定されてるファイル

保存されない。

```
% git status --ignored
On branch master
Ignored files:
  (use "git add -f <file>..." to include in what will be committed)
	README~

nothing to commit, working tree clean
% git stash -u
No local changes to save
```

## 空ディレクトリ

消える。gitは空ディレクトリを管理しないので。

```
% ls -a emptydir                                                              ./  ../
% git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README

no changes added to commit (use "git add" and/or "git commit -a")
% git stash -u
Saved working directory and index state WIP on master: 2be296f add
% ls -a emptydir
ls: emptydir: No such file or directory
% git stash pop
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (cfae721ccf166d2631c40b66e8984e80c0a5baef)
% ls -a emptydir
ls: emptydir: No such file or directory
```

# -a オプションを付けた場合

## 追跡されてないファイル

-u オプションを付けた場合と同様に保存される。

## 無視指定されてるファイル

保存される。

```
% git status --ignored
On branch master
Ignored files:
  (use "git add -f <file>..." to include in what will be committed)
	README~

nothing to commit, working tree clean
% git stash -a
Saved working directory and index state WIP on master: 2be296f add
% git status --ignored
On branch master
nothing to commit, working tree clean
% git stash pop
Already up to date!
On branch master
nothing to commit, working tree clean
Dropped refs/stash@{0} (e287d64ca99f582a530a3cdcd021d12c39c07cb2)
% git status --ignored
On branch master
Ignored files:
  (use "git add -f <file>..." to include in what will be committed)
	README~

nothing to commit, working tree clean
```

## 空ディレクトリ

-u の場合と同様に消える。

# -k (--keep-index) オプションを付けた場合

## ステージされた変更、ステージされた新規ファイル

共に保存される。ステージング領域の状態は維持される。

```
% git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   NEWS
	modified:   README

% git stash -k
Saved working directory and index state WIP on master: 2be296f add
% git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   NEWS
	modified:   README
```

