# 概要

[ghq](https://github.com/x-motemen/ghq/) がサポートしている各VCS(Version Control Systems)での動作を見てみようと考えた。VCS自体のソースはセルフホストしてるはずなので、それらをサンプルとして取得、更新の動作を確認する。

確認環境:

```
yoichinakayama@penguin:~$ uname -a
Linux penguin 5.4.40-04224-g891a6cce2d44 #1 SMP PREEMPT Tue Jun 23 20:13:49 PDT 2020 aarch64 GNU/Linux
yoichinakayama@penguin:~$ ghq --version
ghq version 1.1.3 (rev:HEAD)
```

# Git

[Kernel.org git repositories](https://git.kernel.org/) からgit.gitのリンク先の https://git.kernel.org/pub/scm/git/git.git/ を指定する。

## get

```
yoichinakayama@penguin:~$ ghq get https://git.kernel.org/pub/scm/git/git.git/
     clone https://git.kernel.org/pub/scm/git/git.git/ -> /home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git
       git clone --recursive https://git.kernel.org/pub/scm/git/git.git/ /home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git.git
Cloning into '/home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git.git'...
remote: Enumerating objects: 12656, done.
remote: Counting objects: 100% (12656/12656), done.
remote: Compressing objects: 100% (867/867), done.
remote: Total 288989 (delta 12128), reused 12011 (delta 11788), pack-reused 276333
Receiving objects: 100% (288989/288989), 66.96 MiB | 1.18 MiB/s, done.
Resolving deltas: 100% (218191/218191), done.
Checking out files: 100% (3777/3777), done.
Submodule 'sha1collisiondetection' (https://github.com/cr-marcstevens/sha1collisiondetection.git) registered for path 'sha1collisiondetection'
Cloning into '/home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git.git/sha1collisiondetection'...
remote: Enumerating objects: 6, done.        
remote: Counting objects: 100% (6/6), done.        
remote: Compressing objects: 100% (6/6), done.        
remote: Total 887 (delta 0), reused 4 (delta 0), pack-reused 881        
Receiving objects: 100% (887/887), 611.42 KiB | 317.00 KiB/s, done.
Resolving deltas: 100% (564/564), done.
Submodule path 'sha1collisiondetection': checked out '855827c583bc30645ba427885caa40c5b81764d2'
```

## list

```
yoichinakayama@penguin:~$ ghq list|grep git.kernel.org
git.kernel.org/pub/scm/git/git.git
```

## update

```
yoichinakayama@penguin:~$ ghq list|grep git.kernel.org|ghq get -u
     clone https://git.kernel.org/pub/scm/git/git.git -> /home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git
       git clone --recursive https://git.kernel.org/pub/scm/git/git.git /home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git
Cloning into '/home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git'...
remote: Enumerating objects: 12656, done.
remote: Counting objects: 100% (12656/12656), done.
remote: Compressing objects: 100% (867/867), done.
remote: Total 288989 (delta 12128), reused 12011 (delta 11788), pack-reused 276333
Receiving objects: 100% (288989/288989), 66.96 MiB | 1.39 MiB/s, done.
Resolving deltas: 100% (218191/218191), done.
Checking out files: 100% (3777/3777), done.
Submodule 'sha1collisiondetection' (https://github.com/cr-marcstevens/sha1collisiondetection.git) registered for path 'sha1collisiondetection'
Cloning into '/home/yoichinakayama/ghq/git.kernel.org/pub/scm/git/git/sha1collisiondetection'...
remote: Enumerating objects: 6, done.        
remote: Counting objects: 100% (6/6), done.        
remote: Compressing objects: 100% (6/6), done.        
remote: Total 887 (delta 0), reused 4 (delta 0), pack-reused 881        
Receiving objects: 100% (887/887), 611.42 KiB | 380.00 KiB/s, done.
Resolving deltas: 100% (564/564), done.
Submodule path 'sha1collisiondetection': checked out '855827c583bc30645ba427885caa40c5b81764d2'
yoichinakayama@penguin:~$ ghq list|grep git.kernel.org
git.kernel.org/pub/scm/git/git.git
git.kernel.org/pub/scm/git/git
```

更新しようとしたら別ディレクトリに再取得されてしまった。最初に ghq get するときに末尾の / を除いて指定すれば避けられるが、git コマンドだと

```
yoichinakayama@penguin:~$ git clone https://git.kernel.org/pub/scm/git/git.git/
Cloning into 'git'...
```

と末尾の .git/ を取り除いたパスに取得するので、ghqでも回避できそう。関連するgitの実装は

https://github.com/git/git/blob/101b3204f37606972b40fc17dec84560c22f69f6/builtin/clone.c#L241

のあたり。

(追記) ghq v1.1.4 で直っています ([pull request](https://github.com/x-motemen/ghq/pull/291))

# Subversion

[Source Code](https://subversion.apache.org/source-code.html) の Checking Out Subversion に書かれている `svn co https://svn.apache.org/repos/asf/subversion/trunk subversion` にあるURLを使う。

## get

```
yoichinakayama@penguin:~$ ghq get https://svn.apache.org/repos/asf/subversion/trunk
     clone https://svn.apache.org/repos/asf/subversion/trunk -> /home/yoichinakayama/ghq/svn.apache.org/repos/asf/subversion/trunk
       svn checkout https://svn.apache.org/repos/asf/subversion/trunk /home/yoichinakayama/ghq/svn.apache.org/repos/asf/subversion
...
Checked out revision 1879249.
```

## list and update

```
yoichinakayama@penguin:~$ ghq list|grep subversion
svn.apache.org/repos/asf/subversion
yoichinakayama@penguin:~$ ghq list|grep subversion|ghq get -u
    update /home/yoichinakayama/ghq/svn.apache.org/repos/asf/subversion
       svn update
Updating '.':
At revision 1879249.
yoichinakayama@penguin:~$ 
```

# Mercurial

[Mercurial downloads](https://www.mercurial-scm.org/downloads) の The main development repository として記載されているURL  https://www.mercurial-scm.org/repo/hg を使う。

## get

```
yoichinakayama@penguin:~$ ghq get https://www.mercurial-scm.org/repo/hg
     clone https://www.mercurial-scm.org/repo/hg -> /home/yoichinakayama/ghq/www.mercurial-scm.org/repo/hg
        hg clone https://www.mercurial-scm.org/repo/hg /home/yoichinakayama/ghq/www.mercurial-scm.org/repo/hg
requesting all changes
adding changesets
adding manifests                                                                                                                                                              
adding file changes                                                                                                                                                           
added 45005 changesets with 86775 changes to 3569 files (+1 heads)                                                                                                            
new changesets 9117c6561b0b:2632c1ed8f34
updating to bookmark @
2102 files updated, 0 files merged, 0 files removed, 0 files unresolved     
```

## list and update

```
yoichinakayama@penguin:~$ ghq list|grep mercurial
www.mercurial-scm.org/repo/hg
yoichinakayama@penguin:~$ ghq list|grep mercurial|ghq get -u
    update /home/yoichinakayama/ghq/www.mercurial-scm.org/repo/hg
        hg pull --update
pulling from https://www.mercurial-scm.org/repo/hg
searching for changes
no changes found
```


# Bazaar

## get

[Bazaar](https://code.launchpad.net/~bzr-pqm/bzr/bzr.dev) の `bzr branch lp:bzr` の引数をそのまま指定してみるがうまくいかない。

```
yoichinakayama@penguin:~$ ghq get lp:bzr
     clone ssh://lp/bzr -> /home/yoichinakayama/ghq/lp/bzr
     error failed to get "lp:bzr": unsupported VCS, url=ssh://lp/bzr: Get ssh://lp/bzr?go-get=1: unsupported protocol scheme "ssh"
yoichinakayama@penguin:~$ ghq get --vcs=bzr lp:bzr
     clone ssh://lp/bzr -> /home/yoichinakayama/ghq/lp/bzr
       bzr branch ssh://lp/bzr /home/yoichinakayama/ghq/lp/bzr
bzr: ERROR: Unsupported protocol for url "ssh://lp/bzr": bzr supports bzr+ssh to operate over ssh, use "bzr+ssh://lp/bzr".
     error failed to get "lp:bzr": /usr/bin/bzr: exit status 3
```

まずは bzr で取得してみる。

```
yoichinakayama@penguin:~$ bzr branch lp:bzr
You have not informed bzr of your Launchpad ID, and you must do this to
write to Launchpad or access private data.  See "bzr help launchpad-login".
...
yoichinakayama@penguin:~$ cat bzr/.bzr/branch/branch.conf 
parent_location = http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/
```

これかな？

```
yoichinakayama@penguin:~$ ghq get http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/
     clone http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/ -> /home/yoichinakayama/ghq/bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev
     error failed to get "http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/": unsupported VCS, url=http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/: no go-import meta tags detected
yoichinakayama@penguin:~$ ghq get --vcs=bzr http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/
     clone http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/ -> /home/yoichinakayama/ghq/bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev
       bzr branch http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/ /home/yoichinakayama/ghq/bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev
Branched 6622 revisions.
```

行けた。

## list and update

```
yoichinakayama@penguin:~$ ghq list |grep bzr
bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev
yoichinakayama@penguin:~$ ghq list|grep bzr|ghq get -u
    update /home/yoichinakayama/ghq/bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev
       bzr pull --overwrite
Using saved parent location: http://bazaar.launchpad.net/~bzr-pqm/bzr/bzr.dev/
No revisions or tags to pull.   
```


# Darcs

http://darcs.net/Development
の `darcs clone --lazy http://darcs.net` を参考に

## get

```
yoichinakayama@penguin:~$ ghq get http://darcs.net
     clone http://darcs.net -> /home/yoichinakayama/ghq/darcs.net
     error failed to get "http://darcs.net": unsupported VCS, url=http://darcs.net: no go-import meta tags detected
```

自動判定は失敗する

```
yoichinakayama@penguin:~$ ghq get --vcs=darcs http://darcs.net
     clone http://darcs.net -> /home/yoichinakayama/ghq/darcs.net
     darcs get http://darcs.net /home/yoichinakayama/ghq/darcs.net
Welcome to the darcs screened repository.

If you would like to contribute, please read our guide for contributors:
http://darcs.net/Development/GettingStarted

Thanks and happy hacking!
**********************
Copying patches, to get lazy repository hit ctrl-C...                           
^CUsing lazy repository.

Finished cloning.  
```

待ちきれなくて ctrl-C で止めたけど、待ってればいつか終わったのかな。ghqの実装を見ると、 --shallow オプションを付けると --lazy をつけて darcs clone するようだ。

## list and update

```
yoichinakayama@penguin:~$ ghq list|grep darcs.net
darcs.net
yoichinakayama@penguin:~$ ghq list|grep darcs.net|ghq get -u
     clone https://github.com/yoichi/darcs.net -> /home/yoichinakayama/ghq/github.com/yoichi/darcs.net
       git clone --recursive https://github.com/yoichi/darcs.net /home/yoichinakayama/ghq/github.com/yoichi/darcs.net
Cloning into '/home/yoichinakayama/ghq/github.com/yoichi/darcs.net'...
Username for 'https://github.com': ^C
```

階層構造がないのでプロジェクト名と解釈されてしまっている。

[Development/GettingStarted](http://darcs.net/Development/GettingStarted) に書かれている http://darcs.net/releases/branch-2.12 とかだと大丈夫

```
yoichinakayama@penguin:~$ rm -rf ~/ghq/darcs.net
yoichinakayama@penguin:~$ ghq get --shallow --vcs=darcs http://darcs.net/releases/branch-2.12
     clone http://darcs.net/releases/branch-2.12 -> /home/yoichinakayama/ghq/darcs.net/releases/branch-2.12
     darcs get --lazy http://darcs.net/releases/branch-2.12 /home/yoichinakayama/ghq/darcs.net/releases/branch-2.12
Finished cloning.                                                               
yoichinakayama@penguin:~$ ghq list|grep darcs.net
darcs.net/releases/branch-2.12
yoichinakayama@penguin:~$ ghq list|grep darcs.net|ghq get -u
    update /home/yoichinakayama/ghq/darcs.net/releases/branch-2.12
     darcs pull
Pulling from "http://darcs.net/releases/branch-2.12"...
No remote patches to pull in!
```

# Fossil

[Fossil Self-Hosting Repositories](https://www.fossil-scm.org/home/doc/trunk/www/selfhost.wiki) の three publicly accessible repositories for the Fossil source code の一番上の https://www.fossil-scm.org/ を使う

## get

```
yoichinakayama@penguin:~$ ghq get https://www.fossil-scm.org/
     clone https://www.fossil-scm.org/ -> /home/yoichinakayama/ghq/www.fossil-scm.org
     error failed to get "https://www.fossil-scm.org/": unsupported VCS, url=https://www.fossil-scm.org/: no go-import meta tags detected
```

自動判定できないので、vcsを明示的に指定する。

```
yoichinakayama@penguin:~$ ghq get --vcs=fossil https://www.fossil-scm.org/
     clone https://www.fossil-scm.org/ -> /home/yoichinakayama/ghq/www.fossil-scm.org
    fossil clone https://www.fossil-scm.org/ /home/yoichinakayama/ghq/www.fossil-scm.org/.fossil
Round-trips: 8   Artifacts sent: 0  received: 47004
Clone done, sent: 2102  received: 34054505  ip: 2.0.1.187
...
project-name: Fossil
repository:   /home/yoichinakayama/ghq/www.fossil-scm.org/.fossil
local-root:   /home/yoichinakayama/ghq/www.fossil-scm.org/
config-db:    /home/yoichinakayama/.fossil
project-code: CE59BB9F186226D80E49D1FA2DB29F935CCA0333
checkout:     cd061779d2c192c239e1eb6d0e9254d8193ffa7b 2020-06-27 17:05:41 UTC
parent:       9ef2e5e57b5db1f32141eff5d5aec0c96dee83d5 2020-06-27 15:51:45 UTC
child:        ff735265175830b0073804b395b2f90e6f0869a5 2020-06-27 17:15:31 UTC
tags:         trunk
comment:      Typos in the help text and the change log. (user: drh)
check-ins:    13965
```

# list and update

```
yoichinakayama@penguin:~$ ghq list|grep fossil
www.fossil-scm.org
yoichinakayama@penguin:~$ ghq list|grep fossil|ghq get -u
     clone https://github.com/yoichi/www.fossil-scm.org -> /home/yoichinakayama/ghq/github.com/yoichi/www.fossil-scm.org
       git clone --recursive https://github.com/yoichi/www.fossil-scm.org /home/yoichinakayama/ghq/github.com/yoichi/www.fossil-scm.org
Cloning into '/home/yoichinakayama/ghq/github.com/yoichi/www.fossil-scm.org'...
Username for 'https://github.com': ^C
```

階層構造がないのでプロジェクト名と解釈されてしまっている。

```
yoichinakayama@penguin:~$ curl -v https://www.fossil-scm.org/
...
< HTTP/1.1 301 Permanent Redirect
< Connection: keep-alive
< Date: Sun, 28 Jun 2020 01:20:13 +0000
< Location: https://www.fossil-scm.org/home
< Content-length: 0
< 
* Curl_http_done: called premature == 0
* Connection #0 to host www.fossil-scm.org left intact
```

リダイレクトされてたのでそちらのURLで取得し直せば問題ない

```
yoichinakayama@penguin:~$ rm -rf ~/ghq/www.fossil-scm.org
yoichinakayama@penguin:~$ ghq get https://www.fossil-scm.org/home
     clone https://www.fossil-scm.org/home -> /home/yoichinakayama/ghq/www.fossil-scm.org/home
     error failed to get "https://www.fossil-scm.org/home": unsupported VCS, url=https://www.fossil-scm.org/home: no go-import meta tags detected
yoichinakayama@penguin:~$ ghq get --vcs=fossil https://www.fossil-scm.org/home
     clone https://www.fossil-scm.org/home -> /home/yoichinakayama/ghq/www.fossil-scm.org/home
    fossil clone https://www.fossil-scm.org/home /home/yoichinakayama/ghq/www.fossil-scm.org/home/.fossil
...
project-name: Fossil
repository:   /home/yoichinakayama/ghq/www.fossil-scm.org/home/.fossil
local-root:   /home/yoichinakayama/ghq/www.fossil-scm.org/home/
config-db:    /home/yoichinakayama/.fossil
project-code: CE59BB9F186226D80E49D1FA2DB29F935CCA0333
checkout:     cd061779d2c192c239e1eb6d0e9254d8193ffa7b 2020-06-27 17:05:41 UTC
parent:       9ef2e5e57b5db1f32141eff5d5aec0c96dee83d5 2020-06-27 15:51:45 UTC
child:        ff735265175830b0073804b395b2f90e6f0869a5 2020-06-27 17:15:31 UTC
tags:         trunk
comment:      Typos in the help text and the change log. (user: drh)
check-ins:    13965
yoichinakayama@penguin:~$ ghq list|grep fossil
www.fossil-scm.org/home
yoichinakayama@penguin:~$ ghq list|grep fossil|ghq get -u
    update /home/yoichinakayama/ghq/www.fossil-scm.org/home
    fossil update
Autosync:  https://www.fossil-scm.org/home
Round-trips: 1   Artifacts sent: 0  received: 0
Pull done, sent: 424  received: 1453  ip: 2.0.1.187
-------------------------------------------------------------------------------
checkout:     cd061779d2c192c239e1eb6d0e9254d8193ffa7b 2020-06-27 17:05:41 UTC
tags:         trunk
comment:      Typos in the help text and the change log. (user: drh)
changes:      None. Already up-to-date
```

# CVS

[Concurrent Versions System - CVS Repositories](https://savannah.nongnu.org/cvs/?group=cvs) の `cvs -z3 -d:pserver:anonymous@cvs.savannah.nongnu.org:/sources/cvs co <modulename>` でソースを取得できる。Browse Sources Repositoryのリンク先からmodulename=ccvsを指定すればいいのだけど、ghqでは対応していない

[Add a dummy CVS backend to recognize and skip CVS working directories #115](https://github.com/x-motemen/ghq/pull/115)

動作確認しておく。

```
yoichinakayama@penguin:~$ mkdir -p ghq/cvs.savannah.nongnu.org/sources/cvs
yoichinakayama@penguin:~$ cd $_
yoichinakayama@penguin:~/ghq/cvs.savannah.nongnu.org/sources/cvs$ cvs -z3 -d:pserver:anonymous@cvs.savannah.nongnu.org:/sources/cvs co ccvs
...
```

```
yoichinakayama@penguin:~$ ghq list|grep ccvs
cvs.savannah.nongnu.org/sources/cvs/ccvs
yoichinakayama@penguin:~$ ghq list|grep ccvs|ghq get -u
    update /home/yoichinakayama/ghq/cvs.savannah.nongnu.org/sources/cvs/ccvs
     error failed to get "cvs.savannah.nongnu.org/sources/cvs/ccvs": CVS update is not supported
```

# 考察

VCSの自動判定ができないものがあった。

* Bazaar
* Darcs
* Fossil

ghq getで指定するURLが悩ましいものがあった。

* Bazaar の lp:bzr みたいなの
  * →vcs固有のURLの推定ができればいいのかな
* CVS (対応してないけど)

ghq getで取得されたものに対し、ghq list|ghq get -uがうまく動作しない場合があった。

* URL末尾に / があるため、末尾の .git が取り除かれない場合
  * →末尾の / を取り除く処理を入れればよいかな
* 階層構造にならないため、プロジェクト名指定と解釈されてしまう場合
  * →ghq getのときに階層構造を作ればよいかな

