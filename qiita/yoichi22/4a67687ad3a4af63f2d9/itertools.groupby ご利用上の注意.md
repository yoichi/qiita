# イテレーション中でグループを参照すること

https://docs.python.org/ja/3/library/itertools.html#itertools.groupby より
> 返されるグループはそれ自体がイテレータで、 groupby() と iterable を共有しています。もととなる iterable を共有しているため、 groupby() オブジェクトの要素取り出しを先に進めると、それ以前の要素であるグループは見えなくなってしまいます。


良い例：

```python
>>> import itertools
>>> groups = []
>>> for _, g in itertools.groupby([1, 1, 2, 2, 3, 3]):
...     groups.append(list(g))
...
>>> for g in groups:
...     print(g)
...
[1, 1]
[2, 2]
[3, 3]
```

悪い例：

```python
>>> import itertools
>>> groups = []
>>> for _, g in itertools.groupby([1, 1, 2, 2, 3, 3]):
...     groups.append(g)
...
>>> for g in groups:
...     print(list(g))
...
[]
[]
[]
```

# 事前にソート済みであること

https://docs.python.org/ja/3/library/itertools.html#itertools.groupby より
> 通常、iterable は同じキー関数でソート済みである必要があります。

ソート済みでないと使えないというわけではないが、例えば以下のどちらを期待してるかによって、必要なら事前にソートしてから groupby を呼ぶ。

事前にソートしない例：

```python
>>> import itertools
>>> groups = []
>>> for _, g in itertools.groupby([1, 1, 2, 2, 3, 3, 2, 1]):
...     groups.append(list(g))
...
>>> groups
[[1, 1], [2, 2], [3, 3], [2], [1]]
```

事前にソートする例：

```python
>>> import itertools
>>> groups = []
>>> for _, g in itertools.groupby(sorted([1, 1, 2, 2, 3, 3, 2, 1])):
...     groups.append(list(g))
...
>>> groups
[[1, 1, 1], [2, 2, 2], [3, 3]]
```

# 確認環境

Python 3.7.2 で動作確認しました。
