## 概要

flake8 3.7.0以降では per-file-ignore を指定するとファイル単位で無視する警告を制御できるようになった。例えばCIでプロジェクト内の全ファイルをflake8にかけていて、一部ファイルで特定の警告を抑止したい場合に使える。

## 基本

[Configuration Locations](http://flake8.pycqa.org/en/latest/user/configuration.html#configuration-locations) にあるように、設定はファイル `.flake8` に書ける。

per-file-ignores には、ファイル名のパターンと無視する警告のパターンをコロンで区切って記述する。例えば

```shell-session
$ echo 'import os# fixme' > foo.py
$ echo 'import os# fixme' > boo.py
$ cat .flake8
[flake8]
select=E,W,F
per-file-ignores=
    ./foo.py:F401
```

この場合、./foo.pyでF401が抑止されるので

```shell-session
$ flake8
./boo.py:1:1: F401 'os' imported but unused
./boo.py:1:10: E261 at least two spaces before inline comment
./foo.py:1:10: E261 at least two spaces before inline comment
```

となり以下の警告が出力されなくなっている：

```
./foo.py:1:1: F401 'os' imported but unused
```

## 複数ファイルの警告抑止

複数のファイルパターンで抑止したい場合、いくつかの記法が使える

### 複数行

設定を複数行書ける

```shell-session
$ cat .flake8
[flake8]
select=E,W,F
per-file-ignores=
    ./foo.py:F401
    ./boo.py:F401
$ flake8
./boo.py:1:10: E261 at least two spaces before inline comment
./foo.py:1:10: E261 at least two spaces before inline comment
```

### カンマ区切り

ファイルパターンをカンマ区切りで並べられる

```shell-session
$ cat .flake8
[flake8]
select=E,W,F
per-file-ignores=
    ./foo.py,./boo.py:F401
$ flake8
./boo.py:1:10: E261 at least two spaces before inline comment
./foo.py:1:10: E261 at least two spaces before inline comment
```

### ワイルドカード

`?` や `*` で任意の文字にマッチさせることができる。 `?` は任意の一文字にマッチする。

```shell-session
$ cat .flake8
[flake8]
select=E,W,F
per-file-ignores=
    ./?oo.py:F401
$ flake8
./boo.py:1:10: E261 at least two spaces before inline comment
./foo.py:1:10: E261 at least two spaces before inline comment
```

`*` は任意階層のディレクトリにもマッチするので、特定のディレクトリ以下の全てのファイルを対象としたい場合にも使える。

## 複数種類の警告抑止

コロンの右側の警告のパターンについては、selectやignoreと同様に警告のプレフィックスをカンマ区切りで指定できる。

```shell-session
$ cat .flake8
[flake8]
select=E,W,F
per-file-ignores=
    ./foo.py:E2,F
    ./boo.py:F
$ flake8
./boo.py:1:10: E261 at least two spaces before inline comment
```

この場合、./foo.py では E2* と F* が、./boo.py では F* が抑止される。

## 確認に使ったバージョン

```shell-session
$ flake8 --version
3.7.8 (mccabe: 0.6.1, pycodestyle: 2.5.0, pyflakes: 2.1.1) CPython 3.7.2 on Darwin
```
