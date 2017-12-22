# この話のゴール

* hashable という用語を理解する
* 辞書における hashable の役割を理解する

# 辞書

Python の組み込み型の一つに辞書 (dict) があります。

```pycon
>>> d = {"zero": False, "one": True}
>>> d["zero"]
False
>>> d["one"]
True
>>> type(d)
<class 'dict'>
```

辞書はプログラム中で明示的に使わなくても、モジュール

```pycon
>>> type(__builtins__.__dict__)
<class 'dict'>
>>> import os
>>> type(os.__dict__)
<class 'dict'>
```

や、class の属性

```pycon
>>> class MyClass(object):
...     def __init__(self):
...         self.a = 1
...
>>> x = MyClass()
>>> x.__dict__
{'a': 1}
```

など、いろんなところで暗黙的に使われています。

# 辞書使用時の型エラー

辞書のキーとして指定するとエラーになるものがあります。例えば

```pycon
>>> a = dict()
>>> type(a)
<class 'dict'>
>>> a[[0]] = 1
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```

のようにリストを辞書のキーとして使おうとすると、 unhashable な型なので駄目と怒られてしまいます。
どうしてそのような制約があるのかを見ていきましょう。

# hash とは

https://ja.wikipedia.org/wiki/ハッシュ関数

```
ハッシュ関数 (ハッシュかんすう、hash function) あるいは要約関数とは、
あるデータが与えられた場合にそのデータを代表する数値を得る操作、または、
その様な数値を得るための関数のこと。
```

「データを代表する」の意味は、hash 関数で得られる値が

```
a == b (a.__eq__(b) == True) ならば、 hash(a) == hash(b) (a.__hash__() == b.__hash__())
```

を満たしているということ。

## 辞書におけるハッシュ関数の意義

辞書においては、値の設定、取得の際に、```__eq__``` による全体の探索を避けて、低コストで処理したいという要求に応えるためハッシュ関数が利用されています。

* 値の設定時に上書きするか追加するかを素早く判断
 * ハッシュ値が一致しかつ ```__eq__``` で一致するものものがない→新しいキーを追加
 * ハッシュ値が一致しかつ ```__eq__``` で一致するものがある→既存のキーの値を上書き
* 値の取得時に一致するものを素早く見付ける
 * ハッシュ値が一致しかつ ```__eq__``` で一致するものがない→エラー
 * ハッシュ値が一致しかつ ```__eq__``` で一致するものがある→値を返す

CPython の dict の実装では、

* ハッシュ値に基づく slot を順に探索する→ 全体の探索を避ける
 * 内部で辞書要素は空き要素ありの配列に格納されているが、その一部のみを走査
 * 詳細は dictobject.c の実装を見て下さい
* ```__eq__``` による比較の前にハッシュ値を比較→ ```__eq__``` のコストを避ける
* 辞書要素の挿入時にハッシュ値をキャッシュ→ ハッシュの再計算のコストを避ける

という工夫がされています。

# ```__hash__``` メソッドがあれば hashable？

```__hash__``` メソッドがありさえすれば hashable と言えるのでしょうか？ドキュメントによると

http://docs.python.jp/2.7/glossary.html#term-hashable

```
ハッシュ可能なオブジェクトとは、生存期間中変わらないハッシュ値を持ち (__hash__() メソッドが必要)、...
```

と、ハッシュ値が生存期間中変わらないことが必要とされています。
しかしそんなことは一般には判定できないため、辞書の値設定の実装では、ハッシュ関数が呼び出せるかどうかのみがチェックされています。

builtin オブジェクトの分類は

* hashable: int, str, tuple, frozenset
* unhashable: list, dict, set

となっていますが、ここで hashable の方に入っているものは、ハッシュ値が生存期間中変わらないことが保証されています。では、ユーザ定義オブジェクトの場合はどうでしょうか？

# ユーザ定義オブジェクトの場合

## unhashable なキー

http://docs.python.jp/2.7/reference/datamodel.html#object.__hash__

```
クラスが変更可能なオブジェクトを定義しており、 __cmp__() または __eq__() メソッドを
実装している場合、 __hash__() を定義してはなりません。これは、辞書の実装においてハッシュ
値が変更不能であることが要求されているからです (オブジェクトのハッシュ値が変化すると、
キーが誤ったハッシュバケツ: hash bucket に入っていることになってしまいます)。
```

実際にキーの型チェックをすり抜ける例を試してみます。
以下の例では UnhashableInFact はハッシュ関数を持っているが、インスタンスの生存期間中でハッシュ値が変化します。

```py:wrong_bucket.py
# -*- coding: utf-8 -*-
class UnhashableInFact(object):
    def __init__(self, n):
        self.n = n

    def __hash__(self):
        return hash(self.n)

    def __eq__(self, other):
        return self.n == other.n


if __name__ == '__main__':
    a = UnhashableInFact(1)
    b = UnhashableInFact(2)
    mydict = {a: "value for 1", b: "value for 2"}
    print(mydict[a], mydict[b]) # (1)
    a.n = 2                     # →ハッシュ値が変わる
    print(mydict[a], mydict[b]) # (2)
    c = UnhashableInFact(1)
    print(c in mydict)          # (3)
```

これを実行すると、以下の様な動作になりました。

```
% python wrong_bucket.py
('value for 1', 'value for 2') # (1) それぞれのバケツから値を取り出せる
('value for 2', 'value for 2') # (2) いずれも "value for 2" の方を取り出してしまう
False                          # (3) 元のキーと等しいものを持ってきても "value for 1" が取り出せない
```

何故このような動作になってしまったのでしょうか？
辞書のエントリは

```c:cpython/Include/dictobject.h
typedef struct {
    /* Cached hash code of me_key.  Note that hash codes are C longs.
     * We have to use Py_ssize_t instead because dict_popitem() abuses
     * me_hash to hold a search finger.
     */
    Py_ssize_t me_hash;
    PyObject *me_key;
    PyObject *me_value;
} PyDictEntry;
```

となっており、

* (2) の理由は、辞書のエントリに計算済みハッシュ値をキャッシュしているため
 * 辞書の実装の詳細を知らないと、動作が予測できない
 * → hashable であれば、このようなことは起きない
* (3) の理由は、辞書のエントリ中のキーが書き変わってしまっているため
 * 動作の予測はできるが混乱を招く可能性あり

## unhashable であることを表明するには

http://docs.python.jp/2.7/reference/datamodel.html#object.__hash__

```
親クラスから __hash__() メソッドを継承して、 __cmp__() か __eq__() の意味を
変更している(例えば、値ベースの同値関係から同一性ベースの同値関係に変更する)
クラスのハッシュ値は妥当ではなくなるので、 __hash__ = None をクラス定義に書く
事で、明示的にハッシュ不可能であることを宣言できます。
```

試してみよう。

```py:unhashable.py
class Unhashable(object):
    __hash__ = None

    def __init__(self, n):
        self.n = n

    def __eq__(self, other):
        return self.n == other.n


if __name__ == '__main__':
    a = Unhashable(1)
    {a: "value for 1"}
```

こうしておくと、辞書のキーにしたときに弾いてもらえます：

```
% python unhashable.py
Traceback (most recent call last):
  File "unhashable.py", line 13, in <module>
    {a: "value for 1"}
TypeError: unhashable type: 'Unhashable'
```

なお、Python3 では、```__eq__``` をオーバーライドすると勝手に None にしてくれます

http://docs.python.jp/3/reference/datamodel.html#object.__hash__

```
__eq__() をオーバーライドしていて __hash__() を定義していないクラスは、暗黙的
に None に設定された __hash__() を持ちます。クラスの __hash__() メソッドが
None の場合、そのクラスのインスタンスのハッシュ値を取得しようとすると適切な
TypeError が送出され、 isinstance(obj, collections.Hashable) をチェック
するとハッシュ不能なものとして正しく認識されます。
```

# hashable と immutable

http://docs.python.jp/2.7/glossary.html#term-immutable

```
固定の値を持ったオブジェクトです。変更不能なオブジェクトには、数値、文字列、
およびタプルなどがあります。これらのオブジェクトは値を変えられません。別の値
を記憶させる際には、新たなオブジェクトを作成しなければなりません。
不変オブジェクトは、固定のハッシュ値が必要となる状況で重要な役割を果たします。
辞書におけるキーがその例です。
```

immutable の使いどころとして、辞書におけるキーが挙げられている。

* hashable な builtin オブジェクトは immutable (immutable なので hash 値が変化しないと保証できている)
 * immutable: int, str, tuple, frozenset
 * mutable: list, dict, set
* ユーザ定義オブジェクトを immutable にするのはちょっと手がかかることがある
 * Python ではインスタンス変数の可視性を制限できないため
 * 頑張ればできないこともないらしい http://stackoverflow.com/questions/4996815/ways-to-make-a-class-immutable-in-python
 * non-public だと主張して運用で何とかすることは可能
https://www.python.org/dev/peps/pep-0008/#method-names-and-instance-variables
* immutable を辞書のキーに使えば、辞書自体をいじってないのに取り出されるオブジェクトが変わることはない
 * 前述の (3) のようなことが避けられる。 

# まとめ

* 辞書のキーには hashable を指定すること
 * hashable はオブジェクトの生存期間中ハッシュ値が変化しないもの
 * ユーザ定義オブジェクトではハッシュ値の不変性はプログラマーが保証する必要あり
 * 辞書のキーとして用いた時にチェックされるのはハッシュ関数有無のみ
* 組み込みオブジェクトの int, str, tuple, frozenset は hashable
 * immutable でもある
 * 辞書のキーとして安心して使える
