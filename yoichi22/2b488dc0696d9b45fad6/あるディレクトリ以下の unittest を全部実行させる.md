Python の unittest モジュールを用いた単体テストを複数のファイルに分けて書いていて、それらを全部実行したいとき、TestLoader.discovery() を使えばできる。

```
discover(self, start_dir, pattern='test*.py', top_level_dir=None)
```

デフォルトのパターンだと、start_dir で指定したディレクトリ以下の test で始まり .py で終わるファイルたちを見つけてくる。

次のスクリプトに引数としてディレクトリを与えると、見つけたテストを全部実行する。

```py:runner.py
#!/usr/bin/env python
import sys
from unittest import TestLoader
from unittest import TextTestRunner


def main(path):
    loader = TestLoader()
    test = loader.discover(path)
    runner = TextTestRunner()
    runner.run(test)


if __name__ == '__main__':
    if len(sys.argv) != 2:
        print('usage: %s path' % sys.argv[0])
        sys.exit(1)
    main(sys.argv[1])
```

例えば

```
% find tests -type f -name \*.py
tests/hoge/__init__.py
tests/hoge/test_hoge.py
tests/test_fuga.py
```
というディレクトリ構成だとする。`__init__.py` を配置しているのはサブディレクトリからも探してもらうため。スクリプトを実行してみると以下のようになる。

```
% ./runner.py tests
FF
======================================================================
FAIL: test_1 (hoge.test_hoge.TestHoge)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/python-study/unittest/tests/hoge/test_hoge.py", line 6, in test_1
    self.assertEqual(0, 1)
AssertionError: 0 != 1

======================================================================
FAIL: test_1 (test_fuga.TestFuga)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/python-study/unittest/tests/test_fuga.py", line 6, in test_1
    self.assertEqual(0, 1)
AssertionError: 0 != 1

----------------------------------------------------------------------
Ran 2 tests in 0.000s

FAILED (failures=2)
```

unittest.main に頼るなら以下のような shell script でも同じことができる。

```bash:runner.sh
#!/bin/sh
python -m unittest discover $1
```
