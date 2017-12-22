# この話のゴール

* Python の for ループとは何かを知る(わかった気になる)
* リスト内包表記とジェネレータ表現の違いを知る

# for ループ

```py:for_loop.py
for i in [0, 1, 2, 3, 4]:
    print(i)
```

実行すると

```
% python for_loop.py
0
1
2
3
4
```

```for i in``` の後ろのモノがどう使われているかを知りたい。
(上の例の場合、何となく ```[0, 1, 2, 3, 4]``` の要素が順に取り出されているようだが)

## てきとうなオブジェクトを入れてみる

実行時エラーを参考にどう叩かれているのか見てみる

```py:duck.py
class Duck(object):
    pass


if __name__ == '__main__':
    for i in Duck():
        print(i)
```

実行すると iterable じゃないと怒られる

```
% python duck.py
Traceback (most recent call last):
  File "duck.py", line 6, in <module>
    for i in Duck():
TypeError: 'Duck' object is not iterable
```

* iterable とは ```__iter__()``` メソッドを持つオブジェクトである
 * そういう定義なのでひとまず信じる

## iterable を実装してみる

実装するよう言われたので実装する。ただし空のオブジェクトを返すだけ

```py:duck.py
class DuckIter(object):
    def __init__(self):
        pass


class Duck(object):
    def __iter__(self):
        return DuckIter()


if __name__ == '__main__':
    for i in Duck():
        print(i)
```

エラーが変わった（一歩前進！）

```
% python duck.py
Traceback (most recent call last):
  File "duck.py", line 12, in <module>
    for i in Duck():
TypeError: iter() returned non-iterator of type 'DuckIter'
```

* iterator とは ```next()``` メソッドを持つオブジェクトである
 * Python3 の場合は ```__next__()```メソッド

## iterator を実装してみる

ガーガーと3回鳴くようにしてみる。

```py:duck.py
class DuckIter(object):
    def __init__(self):
        self._count = 3

    def next(self):
        if self._count > 0:
            self._count -= 1
            return "quack"
        raise StopIteration()


class Duck(object):
    def __iter__(self):
        return DuckIter()


if __name__ == '__main__':
    for i in Duck():
        print(i)
```

実行してもエラーはもう出ない

```
% python duck.py
quack
quack
quack
```

アヒルが鳴いた!

## ダックタイピング

If it walks like a duck and quacks like a duck, it must be a duck
https://ja.wikipedia.org/wiki/ダック・タイピング

オブジェクトに何ができるかはオブジェクトそのものが決定する
(それらがどのような継承関係を持っているかとは無関係に)

先ほどの例の場合、

* Duck: ```__iter__()``` メソッドを持っているので iterable と見做された。
* DuckIter: ```next()``` メソッドを持っているので iterator と見做された。

これにより、for ループを回すことができた：

```py
for i in Duck():
    print(i)
```

この表記は以下とだいたい同じ

```py
iter = Duck().__iter__()
while True:
    try:
        i = iter.next()
        print(i)
    except StopIteration:
        break
```

つまり、for ループの処理の流れは
1. iterator を取得
2. iterator の next() メソッドを呼ぶ
3. StopIteration でなければ「ループの中の処理」をして再び 2. へ

## ちょっと練習

list から、逆順の iterator を持つオブジェクトを作ってみよう

```py
>>> lst = [0, 1, 2, 3, 4]
>>> rev = RevList(lst)
>>> for r in rev:
...     print(r)
...
4
3
2
1
0
```

ヒント:

* ```__iter__()``` で「```next()``` メソッドを持つオブジェクト」を取得できるように
* ```next()``` で元のリストの末尾から順に返す

## 解答1

* 素直にヒントに従う
* 元のリストをコピーせず、そのまま参照

```py
class RevListIter(object):
    def __init__(self, lst):
        self._orig = lst
        self._i = len(lst)

    def next(self):
        if self._i > 0:
            self._i -= 1
            return self._orig[self._i]
        raise StopIteration()


class RevList(object):
    def __init__(self, lst):
        self._orig = lst

    def __iter__(self):
        return RevListIter(self._orig)


if __name__ == '__main__':
    lst = [0, 1, 2, 3, 4]
    rev = RevList(lst)
    for r in rev:
        print(r)
```

## 解答2

* step=-1のスライスで逆順のリストを作る
* そのリストの iterator を返す

```py
class RevList(object):
    def __init__(self, lst):
        self._orig = lst

    def __iter__(self):
        return self._orig[::-1].__iter__()


if __name__ == '__main__':
    lst = [0, 1, 2, 3, 4]
    rev = RevList(lst)
    for r in rev:
        print(r)
```

## 2つの解答の違い

* 解答1は iterator を作る時ではなく、使う時に都度要素を生成している
* 解答2は iterator を作る時に逆順のリストを生成している

# リスト内包表記とジェネレータ表現

## もう一つ練習

整数のリストの各要素を二乗したイテレータをつくろう

```py
>>> lst = [0, 1, 2, 3, 4]
>>> for i in なにか(lst):
...     print(i)
...
0
1
4
9
16
```

## 2つの解答例

```py
>>> lst = [0, 1, 2, 3, 4]
>>> for i in [x*x for x in lst]:
...     print(i)
...
0
1
4
9
16
>>> for i in (x*x for x in lst):
...     print(i)
...
0
1
4
9
16
```

どちらも解になっている。2つの違いを見ていこう。

## リスト内包表記

記法

```py
[f(x) for x in lst]
```

サンプルコード

```py:list_comprehension.py
def f(x):
    print("%d*%d" % (x, x))
    return x*x

if __name__ == '__main__':
    lst = [0, 1, 2, 3, 4]
    x = [f(x) for x in lst]
    print(type(x))
    for i in x:
        print(i)
```

実行結果

```
% python list_comprehension.py
0*0
1*1
2*2
3*3
4*4
<type 'list'>
0
1
4
9
16
```

* list を返す
* for ループよりも前に計算して、結果のリストを生成
* for ループでは値を順に取り出すだけ

## ジェネレータ表現

記法

```py
(f(x) for x in lst)
```

サンプルコード

```py:generator_expression.py
def f(x):
    print("%d*%d" % (x, x))
    return x*x

if __name__ == '__main__':
    lst = [0, 1, 2, 3, 4]
    x = (f(x) for x in lst)
    print(type(x))
    for i in x:
        print(i)
```

実行結果

```
% python generator_expression.py
<type 'generator'>
0*0
0
1*1
1
2*2
4
3*3
9
4*4
16
```

* generator を返す
* for ループで使うより前には計算してない
* for ループの中で、値を取り出す際に都度計算

# まとめ

* Python は duck typing である
 * ```__iter__()```, ```next()``` を実装することで、iterable が出来上がる
 * Python3 では ```__next()__```
 * for ループは、 iterator から順に値を取り出して処理を回す
* listからiterableを作る記法
 * list comprehension: 作る時に計算して結果のlistを生成
 * generator expression: 作る時に計算しない。使う時に都度計算
* 関連: map 関数
 * Python2 ではlist comprehensionに相当→list を返す
 *  Python3 ではgenerator expression に相当→iterator を返す
