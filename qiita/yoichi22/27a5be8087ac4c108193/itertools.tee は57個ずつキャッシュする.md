Pythonでitertools.teeを使うと一つのイテレータを複数箇所で利用できますが、どうやってそれを実現しているか、メモリ効率の面でどうなっているかを見ます。

確認環境は Python 3.7.4 です。

## イテレータ

イテレータは、next関数を呼ぶことで順に要素を返し、要素が尽きるとStopIterationをスローします。

```python
Python 3.7.4 (default, Sep 12 2019, 15:40:15)
[GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> itr = iter(range(3))
>>> itr
<range_iterator object at 0x7f36e189bab0>
>>> next(itr)
0
>>> next(itr)
1
>>> next(itr)
2
>>> next(itr)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>>
```

順次要素を生成して返すという実装が可能なため、シーケンス全体をメモリ上に保持する必要がなく、うまく使えばメモリ効率を上げられるというのが特徴です。

## 複数箇所での利用

イテレータは一般に一度消費されてしまうと元に戻れないため、そのままでは複数回利用できません。

```python
>>> itr = iter(range(3))
>>> ','.join(map(str, itr))
'0,1,2'
>>> ','.join(map(str, itr))
''
```

一つのイテレータを複数箇所で使いたい場合、ひとつのやり方は、リストにしてしまうことです。

```python
>>> itr = iter(range(3))
>>> lst = list(itr)
>>> ','.join(map(str, lst))
'0,1,2'
>>> ','.join(map(str, lst))
'0,1,2'
```

ただしこうすると最初にイテレータを消費し全ての要素をメモリ上に保持することになるため、要素が多い場合にメモリ効率がよくありません。

一方、itertools.teeを使うと、一つのイテレータから複数の独立したイテレータを作り、それぞれを利用することができます

```python
>>> from itertools import tee
>>> itr = iter(range(3))
>>> itr0, itr1 = tee(itr)
>>> itr0
<itertools._tee object at 0x7fcd9d9f0320>
>>> itr1
<itertools._tee object at 0x7fcd9d9fa5a0>
>>> ','.join(map(str, itr0))
'0,1,2'
>>> ','.join(map(str, itr0))
''
>>> ','.join(map(str, itr1))
'0,1,2'
>>> ','.join(map(str, itr1))
''
>>>
```

## イテレータの消費のされ方

動きを見るため、独自のイテレータをクラスとして定義し、ログを出力してみます。

```python
>>> class Iterator(object):
...     def __init__(self):
...         print('__init__', self)
...         self._itr = iter(range(3))
...     def __iter__(self):
...         print('__iter__', self)
...         return self
...     def __next__(self):
...         print('__next__', self)
...         return next(self._itr)
... 
>>> itr0,itr1 = tee(Iterator())
__init__ <__main__.Iterator object at 0x7fcd9d9f5890>
__iter__ <__main__.Iterator object at 0x7fcd9d9f5890>
__iter__ <__main__.Iterator object at 0x7fcd9d9f5890>
>>> next(itr0)
__next__ <__main__.Iterator object at 0x7fcd9d9f5890>
0
>>> next(itr1)
0
>>> next(itr1)
__next__ <__main__.Iterator object at 0x7fcd9d9f5890>
1
>>> next(itr0)
1
>>>
```

最初に消費した方（一回目はitr0、二回目はitr1）では `__next__` のログが表示されているのに、他方（itr1で1要素目を取る時、itr0で2要素目を取る時）には表示されていません。つまり、teeによって作成されるイテレータは元のイテレータをラップして、最初に消費したときの戻りをキャッシュしておき、後の消費時にはキャッシュしていた値を返していると考えられます。

## キャッシュの生存期間

イテレータは一度に全ての要素をメモリ上に保持しないのが特徴と書いていました。teeを使った場合に最初に消費したときの戻りを全てキャッシュしているとすると、イテレータを消費していくにつれて、リストにしてしまった場合との差異はなくなっていくのでしょうか？というわけでキャッシュの生存期間を確認してみます。

```python:sample.py
class Foo(object):
    def __init__(self, n):
        self._n = n
        print(f"Foo.__init__, n={self._n}")
    def __str__(self):
        return f"Foo({self._n})"
    def __del__(self):
        print(f"Foo.__del__, n={self._n}")

class FooIterator(object):
    def __init__(self, n):
        self._n = n
    def __iter__(self):
        return self
    def __next__(self):
        n = self._n
        if n >= 0:
            self._n -= 1
            return Foo(n)
        raise StopIteration

if __name__ == '__main__':
    from itertools import tee
    itr0, itr1 = tee(FooIterator(100))
    for i in itr0:
        print('1st', i)
    for i in itr1:
        print('2nd', i)
```

```console
$ python sample.py
Foo.__init__, n=100
1st Foo(100)
Foo.__init__, n=99
1st Foo(99)
...
Foo.__init__, n=0
1st Foo(0)
2nd Foo(100)
2nd Foo(99)
...
2nd Foo(45)
2nd Foo(44)
Foo.__del__, n=100
Foo.__del__, n=99
...
Foo.__del__, n=45
Foo.__del__, n=44
2nd Foo(43)
2nd Foo(42)
...
2nd Foo(1)
2nd Foo(0)
Foo.__del__, n=0
Foo.__del__, n=43
...
Foo.__del__, n=3
Foo.__del__, n=2
Foo.__del__, n=1
$ 
```

"2nd Foo(44)" より手前では `__del__` のログが表示されていないこと、"2nd Foo(44)"と"2nd Foo(43)"の間に100-44+1=57個の消費されたイテレータに対する `__del__` のログが表示されていないことが見て取れます。

実際にcPythonのソースを確認すると、

```c:cpython/Modules/itertoolsmodule.c
#define LINKCELLS 57

typedef struct {
    PyObject_HEAD
    PyObject *it;
    int numread;                /* 0 <= numread <= LINKCELLS */
    PyObject *nextlink;
    PyObject *(values[LINKCELLS]);
} teedataobject;
```

という構造体でキャッシュを保持していて、一要素消費するごとにnumreadがインクリメントされ、valuesに戻りがキャッシュされる、LINKCELLS個たまったら次のteedataobjectを作ってnextlinkでリンクするという処理になっているので、teeで生成されたイテレータの複製のいずれでも消費され参照がなくなったオブジェクトは57個ずつ破棄されていくということになります。

## 結論

itertools.teeを使うとイテレータを複数箇所で使うことができますが、その際に消費済みリソースの解放は57個単位で行われており、一個一個の単位ではないものの、イテレータの特徴を保持している。したがって最初にリストにしてしまうのと比べるとメモリ効率の高い実装をすることが可能。
