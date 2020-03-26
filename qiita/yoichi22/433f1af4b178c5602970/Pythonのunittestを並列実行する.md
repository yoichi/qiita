# 概要

やりたいこと：
Python標準のunittestモジュールを利用したテストコードを並列実行したい

やりかた：
テストランナーとして nose を使い、 `--processes` オプションで並列実行数を指定する

# 確認環境

```console
$ python --version
Python 3.7.2
$ nosetests --version
nosetests version 1.3.7
```
# 確認用サンプルコード

ここでは並列実行の効果を見たいので、time.sleep() でテストメソッドに1秒の待ちを入れておく。

```console
$ cat test_s1.py
import unittest
import time
class Test(unittest.TestCase):
    def test_method(self):
        time.sleep(1)
        self.assertTrue(True)
$ cat test_f1.py
import unittest
import time
class Test(unittest.TestCase):
    def test_method(self):
        time.sleep(1)
        self.assertFalse("ヒャッハー")
$ ls
test_f1.py  test_s1.py
$ cp test_s{1,2}.py
$ cp test_s{1,3}.py
$ ls
test_f1.py  test_s1.py  test_s2.py test_s3.py
```

# 標準のテストランナーを使う

シーケンシャルに実行されるので4秒かかる

```console
$ python -m unittest
F...
======================================================================
FAIL: test_method (test_f1.Test)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/yoichi/prog/python-study/nose/test_f1.py", line 6, in test_method
    self.assertFalse("ヒャッハー")
AssertionError: 'ヒャッハー' is not false

----------------------------------------------------------------------
Ran 4 tests in 4.024s

FAILED (failures=1)
```

# nosetests オプションなし

```console
$ nosetests
F...
======================================================================
FAIL: test_method (test_f1.Test)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/yoichi/prog/python-study/nose/test_f1.py", line 6, in test_method
    self.assertFalse("ヒャッハー")
AssertionError: 'ヒャッハー' is not false

----------------------------------------------------------------------
Ran 4 tests in 4.066s

FAILED (failures=1)
```

https://nose.readthedocs.io/en/latest/plugins/multiprocess.html より、デフォルトは並列実行しないので想定どおり。

```
--processes=NUM
  Spread test run among this many processes. Set a number equal to the number
  of processors or cores in your machine for best results. Pass a negative
  number to have the number of processes automatically set to the number of
  cores. Passing 0 means to disable parallel testing. Default is 0 unless
  NOSE_PROCESSES is set. 
```

# nosetests で並列実行

2並列だと2秒で終わる

```console
$ nosetests --processes=2
F...
======================================================================
FAIL: test_method (test_f1.Test)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/yoichi/prog/python-study/nose/test_f1.py", line 6, in test_method
    self.assertFalse("ヒャッハー")
AssertionError: 'ヒャッハー' is not false

----------------------------------------------------------------------
Ran 4 tests in 2.090s

FAILED (failures=1)
```

4並列だと1秒で終わる

```console
$ nosetests --processes=4
F...
======================================================================
FAIL: test_method (test_f1.Test)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/yoichi/prog/python-study/nose/test_f1.py", line 6, in test_method
    self.assertFalse("ヒャッハー")
AssertionError: 'ヒャッハー' is not false

----------------------------------------------------------------------
Ran 4 tests in 1.128s

FAILED (failures=1)
```

並列数を上げるとそれだけ同時利用するリソースが増えるので、実行環境に応じて適当な値を設定しましょう。

# 並列実行の単位

## 複数のTestCase

一つのファイルに複数のTestCaseクラスがある場合も並列実行してくれる

```console
$ cat test_s1.py
import unittest
import time
class Test1(unittest.TestCase):
    def test_method1(self):
        time.sleep(1)
        self.assertTrue(True)
class Test2(unittest.TestCase):
    def test_method1(self):
        time.sleep(1)
        self.assertTrue(True)
$ ls
test_s1.py
$ nosetests --processes=2
..
----------------------------------------------------------------------
Ran 2 tests in 1.073s

OK
```

## 複数のテストメソッド

メソッド単位での並列実行はしてくれない

```console
$ cat test_s1.py
import unittest
import time
class Test1(unittest.TestCase):
    def test_method1(self):
        time.sleep(1)
        self.assertTrue(True)
    def test_method2(self):
        time.sleep(1)
        self.assertTrue(True)
$ ls
test_s1.py
$ nosetests --processes=2
..
----------------------------------------------------------------------
Ran 2 tests in 2.073s

OK
```

