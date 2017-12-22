wrapt は Graham Dumpleton (mod_wsgi の作者) による、ラッパー記述を補助してくれるライブラリです。

* ドキュメント: https://wrapt.readthedocs.io/
* ソース: https://github.com/GrahamDumpleton/wrapt
* PyPI: https://pypi.python.org/pypi/wrapt

functools.wraps のサンプル
 (https://docs.python.org/3/library/functools.html#functools.wraps) は wrapt を使うと次のように書けます(functools.wraps のように入れ子の関数を書かなくていい)

```pycon
>>> import wrapt
>>> @wrapt.decorator
... def my_decorator(wrapped, instance, args, kwargs):
...     print('Calling decorated function')
...     return wrapped(*args, **kwargs)
...
>>> @my_decorator
... def example():
...     """Docstring"""
...     print('Called example function')
...
>>> example()
Calling decorated function
Called example function
>>> example.__name__
'example'
>>> example.__doc__
'Docstring'
```

ラップ済みかどうかは wrapt.ObjectProxy のインスタンスかどうかで判定できます。

```pycon
>>> isinstance(example, wrapt.ObjectProxy)
True
```

インスタンスメソッドをラップする場合には、引数 instance にインスタンスが渡ってきます。

```pycon
>>> @wrapt.decorator
... def wrap_instance_method(wrapped, instance, args, kwargs):
...     print('method {} of {}'.format(wrapped.__name__, instance))
...     return wrapped(*args, **kwargs)
...
>>> class Example(object):
...     @wrap_instance_method
...     def example(self):
...         """DocString"""
...         print('Called example method')
...
>>> obj = Example()
>>> obj.example()
method example of <__main__.Example object at 0x10cdd53d0>
Called example method
```

ラッパー関数の引数が wrapped, instance, args, kwargs の4つと決まっていて、慣れてしまえばどう書けばいいんだっけと悩むことなしに一定のリズムで書いていけるのがとてもよいと感じています。

wrapt の利用事例としては、Webアプリのパフォーマンス監視サービス Datadog APM を使うためのライブラリ [dd-trace-py](https://github.com/DataDog/dd-trace-py) があります。ここでは計測対象のライブラリにフックを仕込むためにモンキーパッチをあてるのに使われています(私はそれで wrapt を知りました)。 モンキーパッチを補助する機能については
 https://github.com/GrahamDumpleton/wrapt/blob/develop/blog/11-safely-applying-monkey-patches-in-python.md に説明があるので興味のある方は参考にしてください。
