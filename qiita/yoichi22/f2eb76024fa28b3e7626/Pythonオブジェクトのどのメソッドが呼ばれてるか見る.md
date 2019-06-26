MagicMock を使い、 `mock_calls` の値を確認すればいい。
例えば `len()` 関数ではオブジェクトの `__len__()` メソッドが呼ばれる：

```python
Python 3.7.2 (default, Dec 27 2018, 07:35:52)
[Clang 10.0.0 (clang-1000.11.45.5)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> from unittest.mock import MagicMock
>>> m = MagicMock()
>>> len(m)
0
>>> m.mock_calls
[call.__len__()]
```

for では `__iter__()` が呼ばれる：

```python
>>> m = MagicMock()
>>> for i in m:
...     print(i)
...
>>> m.mock_calls
[call.__iter__()]
```

なお、記録されるのはメソッド呼び出しであり、属性値を読み出しただけだと記録されない。

```
>>> m = MagicMock()
>>> print(m.__doc__)

    MagicMock is a subclass of Mock with default implementations
    of most of the magic methods. You can use MagicMock without having to
    configure the magic methods yourself.

    If you use the `spec` or `spec_set` arguments then *only* magic
    methods that exist in the spec will be created.

    Attributes and the return value of a `MagicMock` will also be `MagicMocks`.

>>> m.mock_calls
[]
```
