# 二項演算子だと思って右側に勝手なものを書くとシンタックスエラーになる

## MySQLの場合

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.15 MySQL Community Server - GPL

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> select 2 is 1;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '1' at line 1
mysql> select 2 is not 1;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '1' at line 1
mysql> 
```

## PostgreSQLの場合

```
psql (11.2 (Debian 11.2-1.pgdg90+1))
Type "help" for help.

postgres=# select 2 is 1;
ERROR:  syntax error at or near "1"
LINE 1: select 2 is 1;
                    ^
postgres=# select 2 is not 1;
ERROR:  syntax error at or near "1"
LINE 1: select 2 is not 1;
                        ^
postgres=# 
```

# `IS NULL` とか `IS NOT NULL` という単項演算子は存在する

## MySQLの場合

```
mysql> select 2 is null;
+-----------+
| 2 is null |
+-----------+
|         0 |
+-----------+
1 row in set (0.00 sec)

mysql> select 2 is not null;
+---------------+
| 2 is not null |
+---------------+
|             1 |
+---------------+
1 row in set (0.00 sec)
```

## PostgreSQLの場合

```
postgres=# select 2 is null;
 ?column?
----------
 f
(1 row)

postgres=# select 2 is not null;
 ?column?
----------
 t
(1 row)
```
