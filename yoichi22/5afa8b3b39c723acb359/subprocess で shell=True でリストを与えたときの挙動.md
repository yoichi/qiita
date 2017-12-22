# 発端

embulk とか digdag のようにシェルスクリプトに jar が埋め込まれたファイルを Python の subprocess を利用して shell=False で実行しようとすると

```pycon
>>> import subprocess
>>> subprocess.check_call(["digdag"])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/Cellar/python/2.7.13/Frameworks/Python.framework/Versions/2.7/lib/python2.7/subprocess.py", line 181, in check_call
    retcode = call(*popenargs, **kwargs)
  File "/usr/local/Cellar/python/2.7.13/Frameworks/Python.framework/Versions/2.7/lib/python2.7/subprocess.py", line 168, in call
    return Popen(*popenargs, **kwargs).wait()
  File "/usr/local/Cellar/python/2.7.13/Frameworks/Python.framework/Versions/2.7/lib/python2.7/subprocess.py", line 390, in __init__
    errread, errwrite)
  File "/usr/local/Cellar/python/2.7.13/Frameworks/Python.framework/Versions/2.7/lib/python2.7/subprocess.py", line 1024, in _execute_child
    raise child_exception
OSError: [Errno 8] Exec format error
```

のように失敗する。shell=True としたら回避できるのだが、プログラムへの引数の渡し方が shell が False (default) の場合と True の場合と違っていたので調べた。

# 疑問に思ったこと

shell=False の場合にリストを与えると、コマンドとその引数として処理してくれる。例えば

```sample.sh
#!/bin/sh
echo "$1", "$2", "$3" > out.txt
```
というシェルスクリプトを subprocess.check_call を使って呼び出すコード

```caller_l0.py
import subprocess
cmd = ["./sample.sh", "1starg", "2ndarg", "3rdarg"]
subprocess.check_call(cmd, shell=False)
```

を実行すると、

```shell-session
$ python caller_l0.py
$ cat out.txt
1starg, 2ndarg, 3rdarg
```

と引数が意図通りに渡っているのがわかる。それをそのまま shell=True にして、

```caller_l1.py
import subprocess
cmd = ["./sample.sh", "1starg", "2ndarg", "3rdarg"]
subprocess.check_call(cmd, shell=True)
```

を実行すると、

```
$ python caller_l1.py
$ cat out.txt
, ,
```

リストの第二要素以降はどこ行ったんだ？

# 調べたこと

subprocess に渡す args パラメータについては、 [公式ドキュメント](https://docs.python.org/3/library/subprocess.html) に以下のように書かれている:

```
args is required for all calls and should be a string, or a sequence of program arguments.
Providing a sequence of arguments is generally preferred, as it allows the module to take
care of any required escaping and quoting of arguments (e.g. to permit spaces in file names).
If passing a single string, either shell must be True (see below) or else the string must
simply name the program to be executed without specifying any arguments.
```

* str もしくは list を指定せねばならない
* 一般に list が推奨
 * 引数のエスケープとかクォートをしてくれる
* str を指定するのは以下の場合に限る
 * shell=True で呼ぶ場合
 * 引数なしでプログラム名だけを指定する場合

shell=True で list を与えた場合についての説明はここにはない。もう少し探すと

```
If args is a sequence, the first item specifies the command string, and any additional
items will be treated as additional arguments to the shell itself.
That is to say, Popen does the equivalent of:

  Popen(['/bin/sh', '-c', args[0], args[1], ...])
```

とあり、MacOS では /bin/sh は bash なので man すると

```
-c string If  the  -c  option  is  present, then commands are read from
          string.  If there are arguments after the  string,  they  are
          assigned to the positional parameters, starting with $0.
```

と書いてあるのだが、よくわからなかったので検索して https://stackoverflow.com/questions/1711970/cant-seem-to-use-bash-c-option-with-arguments-after-the-c-option-string にたどり着き、 -c に与えた文字列中の \$0, \$1, ... が置換されるのだと理解した。


```caller_l1_.py
import subprocess
cmd = ["./sample.sh $0 $1 $2", "1starg", "2ndarg", "3rdarg"]
subprocess.check_call(cmd, shell=True)
```

```shell-session
$ python caller_l1_.py
$ cat out.txt
1starg, 2ndarg, 3rdarg
```

## sh -c の後の引数

sh -c の後に複数引数を渡した時の挙動について、bash 以外でも確認したところ、Ubuntu の dash, FreeBSD の sh でも同じ挙動をしていた。そのことから何か使い道があるのだろうと思った。

```
$ bash -c '$0 $1, $2, $3' echo '1-0 1-1' 2 3
1-0 1-1, 2, 3
```

ではクォートしたものをあてはめてくれて、　subprocess 経由でも同様に

```
$ python -c 'import subprocess;subprocess.check_call(["$0 $1, $2, $3", "echo", "1-0 1-1", "2", "3"], shell=True)'
1-0 1-1, 2, 3
```

となるが、これだったら最初から文字列展開しておけばいいし、一方で、

```caller_l1__.py
import subprocess
cmd = ["$0 $1 $2 $3", "./sample.sh", "1-1 1-2", "2", "3"]
subprocess.check_call(cmd, shell=True)
```

だと置換された後のものが sample.sh に渡るだけなので

```shell-session
$ python caller_l1__.py
$ cat out.txt
1-1, 1-2, 2
```

となってしまうし、結局のところ -c の後に複数引数を与えるとよい場面はよくわからなかった。

# まとめ

Python の subprocess で shell=True でリストを渡したときの挙動は sh -c の後に複数の引数を渡した時の挙動に準ずる。ただしその使い所はよくわからなかった（誰か教えて）。

shell=True の時はリストではなく文字列を指定するのが素直な気がした。

```caller_s1.py
import subprocess
cmd = "./sample.sh 1starg 2ndarg 3rdarg"
subprocess.check_call(cmd, shell=True)
```

```shell-session
$ python2.7 caller_s1.py
$ cat out.txt
1starg, 2ndarg, 3rdarg
```
