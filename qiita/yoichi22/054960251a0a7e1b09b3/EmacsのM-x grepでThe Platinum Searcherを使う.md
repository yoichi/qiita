[The Platinum Searcher](https://github.com/monochromegane/the_platinum_searcher) (pt) は複数のエンコーディングのテキストファイルが混在する中で素早くファイル中のキーワードを含む行を見つけてくれるツールです。
Emacs 上で pt を使う方法として、pt の README では pt.el が紹介されています。ここではそれを使わずに Emacs 標準機能の [M-x grep](https://www.gnu.org/software/emacs/manual/html_node/emacs/Grep-Searching.html) で使う方法を紹介します。

# macOS

```
(require 'grep)
(grep-apply-setting 'grep-command "pt --nogroup ")
(grep-apply-setting 'grep-use-null-device nil)
```

確認環境

* https://emacsformacosx.com/ から取得した Emacs 26.3
* pt version 2.2.0

# Windows

```
(require 'grep)
(grep-apply-setting 'grep-command "echo . | xargs pt ")
```

確認環境

* http://ftp.gnu.org/gnu/emacs/windows/ から取得した Emacs 26.3
* SourceTree からインストールした Git Bash に入ってる echo, xargs
* pt version 2.2.0

