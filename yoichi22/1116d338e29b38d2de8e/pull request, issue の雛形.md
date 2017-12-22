# 背景

[OSS Gate](http://oss-gate.doorkeeper.jp/) のワークショップに参加したのですが、issue を作成するときのテンプレートが前回と変わっていて、これってどこで設定してるのかな？と思って調べました。

# GitHub では

https://github.com/blog/2111-issue-and-pull-request-templates

でテンプレートについて紹介されています。リポジトリの最上位ディレクトリに、

* ISSUE_TEMPLATE.md
* PULL_REQUEST_TEMPLATE.md

があると、それぞれのファイルの内容が issue あるいは pull request の作成時に本文に挿入されます。最上位ディレクトリを散らかしたくない場合は

* .github/ISSUE_TEMPLATE.md
* .github/PULL_REQUEST_TEMPLATE.md

としてもよいとのこと。実際、 https://github.com/oss-gate/workshop/ では .github/ISSUE_TEMPLATE.md が用意されていました。

なお、使われるのは https://github.com/ユーザ名/リポジトリ名/settings/branches で設定される Default branch の先端のファイルです。

# Bitbucket では

issue は上がってる (https://bitbucket.org/site/master/issues/11571/custom-pull-request-description-template) けどまだ対応されていないようです。
