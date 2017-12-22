magit で差分表示したときに、今いる hunk 内で行内の違い（空白の違いも含め）に色付けする設定

```el
;; 今居るhunkの行内の差分に色付けする
(setq magit-diff-refine-hunk t)
;; 空白の差を無視しない
(setq smerge-refine-ignore-whitespace nil)
```

![magit-diff-refine-hunk.png](https://qiita-image-store.s3.amazonaws.com/0/64323/177b1747-53cf-47f9-4d88-f58991c1b192.png)


上の設定だとカーソルが居るhunkのみ色付けされることに注意。magit-diff-refine-hunk に 'all を指定すれば全てのhunksに色付けが適用される。
