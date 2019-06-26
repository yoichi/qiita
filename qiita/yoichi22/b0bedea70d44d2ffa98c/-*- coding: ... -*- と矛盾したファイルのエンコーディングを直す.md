
Emacs で -*- coding: ... -*- と矛盾したファイルを開いた場合、例えば、

```
-*- coding: utf-8 -*-
```

とかいてあるのに shift_jis でエンコードされたファイルを開いて文字化けした状態で、 `C-x RET r` で実際のエンコーディングを指定すると正しく表示されるようになる。
その状態でファイルを編集して保存すれば、coding: と整合したエンコーディングで保存される。

参考: http://www.gnu.org/software/emacs/manual/html_node/emacs/Specify-Coding.html

# 余談

反対に、coding: と矛盾させて（その指定によらないエンコーディングで）保存するにはどうすればと思ったのですが、以下の手順で行けました。

1. `C-x RET f` でバッファエンコーディングを指定 (例えば shift_jis)
2. `M-:` で以下を評価

```
(let ((coding-system-for-write 'shift_jis)) (save-buffer))
```
