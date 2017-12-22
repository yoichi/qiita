Emacs 上の ddskk で SKK-JISYO.L を使って日本語入力しており、git のコミットメッセージに絵文字を入れるのが流行っているようなので入力できるように設定してみました。

環境: Emacs 25.1 / macOS Sierra

# うまく行った手順

1. [skk-emoji-jisyo](https://github.com/uasi/skk-emoji-jisyo) の SKK-JISYO.emoji.utf8 をダウンロードする。
2. `nkf -w -Lu SKK-JISYO.L > SKK-JISYO.L.utf8` で large-jisyo のエンコーディングを utf8 にする
3. `nkf -w -Lu ~/.skk-jisyo > ~/.skk-jisyo.utf8` で個人辞書のエンコーディングを utf8 にする
4. https://github.com/skk-dev/skktools を git clone して `./configure && make` して skkdic-expr2 をビルド
5. `skkdic-expr2 SKK-JISYO.L.utf8 SKK-JISYO.emoji.utf8 > SKK-JISYO.L+emoji.utf8` で辞書ファイルを連結
6. 絵文字が表示できるように http://users.teilar.gr/~g1951d/ から Symbola をダウンロードしてインストール

7. ~/.emacs.d/init.el に以下の設定を入れる

```el
(setq skk-large-jisyo "辞書置き場/SKK-JISYO.L+emoji.utf8")
(setq skk-jisyo "~/.skk-jisyo.utf8")
(setq skk-jisyo-code 'utf-8)
```

![スクリーンショット 2016-10-17 5.16.09.png](https://qiita-image-store.s3.amazonaws.com/0/64323/8ca461e4-2114-b078-2d81-e19d87dc9769.png)

![スクリーンショット 2016-10-17 5.17.15.png](https://qiita-image-store.s3.amazonaws.com/0/64323/a6dccc10-79e9-8db5-7917-44ab4394a39b.png)


# ハマったことなど

* SKK-JISYO.L のエンコーディングが euc-jp だったので単純に連結すると壊れた
* 辞書ファイルは cat で単に連結しただけでは駄目で、skkdic-expr2 などを使う必要があった
* 以下を参考にしました。
 * [ponkotuyの日記 / 辞書結合 ～SKK-JISYO.LLを作ってみる～](http://d.hatena.ne.jp/ponkotuy/20101214/1292343807)
 * [My備忘録 / Emacsで使用するSKKの辞書の文字コードをutf-8にする](http://arat.xyz/wordpress/?p=129)
 * [Linux とかでも Unicode 絵文字を表示するためのフォント](http://qiita.com/polamjag/items/7295a15fca6a9eeb5d84)
* Github でコミットメッセージに絵文字を出すだけなら単に `:sushi:` とか入れとけばよいことに後で気付いた
