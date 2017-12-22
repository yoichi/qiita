[Requests](http://docs.python-requests.org/en/master/) v2.9.1 では json ライブラリとして

* simplejson があればそれを使う
* なければ json を使う

という実装になっており、json ライブラリの関数で使っているのは dumps, loads のみなので、それらの互換性のある他の json ライブラリ、例えば ujson を使うこともできる。

以下のようにモジュールを格納している変数を上書きすれば切り替えられる：

```py
import requests
import ujson

requests.models.complexjson = ujson
```

ただし、

* あくまでも今の Requests の実装でうまくいく方法に過ぎない（外部仕様として保証されているわけではない）ので、 Requests のバージョンアップでうまくいかなくなる可能性があること
* モジュール変数を書き換えているので、プロセス全体に影響してしまうこと

には注意。

# 参考文献

https://github.com/kennethreitz/requests/issues/1595
当時は requests.models.json という変数名だったが、こっそりリネームされていたようです。
以下のバージョンでは complexjson という名前になっています:

```
% git tag --contains fb6dade6
v2.8.0
v2.8.1
v2.9.0
v2.9.1
```

（明確な理由がなきゃ変えないといいつつ、不明確なコミットメッセージで変更されているのが気になったりしますが）
