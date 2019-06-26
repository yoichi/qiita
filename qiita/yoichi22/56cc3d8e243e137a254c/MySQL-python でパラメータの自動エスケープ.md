# まとめ

Python の MySQL-python (MySQLdb) モジュールを使ってクエリを構築するときは `Cursor.execute(self, query, args=None)` の説明にある placeholder を活用しよう。

```
query -- string, query to execute on server
args -- optional sequence or mapping, parameters to use with query.

Note: If args is a sequence, then %s must be used as the
parameter placeholder in the query. If a mapping is used,
%(key)s must be used as the placeholder.
```

# テスト用のテーブル

テスト用に以下のようなテーブルを作っておく。

```
mysql> select * from testdb.person;
+------+--------+
| id   | name   |
+------+--------+
|    1 | foo    |
|    2 | bar    |
+------+--------+
```

# 悪い例

```py:bad.py
import MySQLdb


def select(name):
    connection = MySQLdb.connect(db='testdb', user='testuser')
    cursor = connection.cursor()
    cursor.execute("select * from person where name='%s'" % name)
    print("[query]")
    print(cursor._last_executed)
    print("[result]")
    result = cursor.fetchall()
    for rec in result:
        print(rec)
```

`select("foo")` とすると、いい感じに動いているように見える。

```
[query]
select * from person where name='foo'
[result]
(1L, 'foo')
```

しかし、 `select("foo' or name=name-- ")` のようにすると

```
[query]
select * from person where name='foo' or name=name-- '
[result]
(1L, 'foo')
(2L, 'bar')
```

と SQL injection ができてしまう。

# 良い例

```py:good.py
import MySQLdb


def select(name):
    connection = MySQLdb.connect(db='testdb', user='testuser')
    cursor = connection.cursor()
    cursor.execute("select * from person where name=%s", name)
    print("[query]")
    print(cursor._last_executed)
    print("[result]")
    result = cursor.fetchall()
    for rec in result:
        print(rec)
```

変えたのは `cursor.execute()` の引数の部分のみ。

`select("foo")` とすると、先程の例と同じように動作する。

```
[query]
select * from person where name='foo'
[result]
(1L, 'foo')
```

`select("foo' or name=name-- ")` としてもちゃんとエスケープしてくれる。

```
[query]
select * from person where name='foo\' or name=name-- '
[result]
```

# 参考文献

http://stackoverflow.com/questions/1947750/does-python-support-mysql-prepared-statements
