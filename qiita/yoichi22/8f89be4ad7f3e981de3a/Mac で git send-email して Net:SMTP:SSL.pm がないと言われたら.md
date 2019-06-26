# エラーの内容

Gmail の SMTP server で git send-email しようとしたらエラーになった。
macOS High Sierra バージョン 10.13.6 で git のバージョンは

```
$ /usr/bin/git --version
git version 2.15.2 (Apple Git-101.1)
```

エラーメッセージは

```
$ /usr/bin/git send-email 0001-add-readme.patch
... 略 ...
Can't locate Net/SMTP/SSL.pm in @INC (you may need to install the Net::SMTP::SSL module) (@INC contains: /Applications/Xcode.app/Contents/Developer/usr/../Library/Perl/5.18/darwin-thread-multi-2level /Applications/Xcode.app/Contents/Developer/usr/share/git-core/perl /Library/Perl/5.18/darwin-thread-multi-2level /Library/Perl/5.18 /Network/Library/Perl/5.18/darwin-thread-multi-2level /Network/Library/Perl/5.18 /Library/Perl/Updates/5.18.2/darwin-thread-multi-2level /Library/Perl/Updates/5.18.2 /System/Library/Perl/5.18/darwin-thread-multi-2level /System/Library/Perl/5.18 /System/Library/Perl/Extras/5.18/darwin-thread-multi-2level /System/Library/Perl/Extras/5.18 .) at /Applications/Xcode.app/Contents/Developer/usr/libexec/git-core/git-send-email line 1450.
```

# 解決方法

```
$ sudo -H cpan5.18 Net::SMTP::SSL
```

でモジュールをインストールしたら解決した。

# はまったこと

手元の環境では Homebrew でインストールした perl が存在していたため、バージョン明記して cpan5.18 とするのではなく cpan コマンドを実行したら、git send-email が使ってたのとは別のバージョンの perl に対してインストールしてて効かなかった。

# 教訓

エラーメッセージはちゃんと読もう（ちゃんと読んでいれば perl5.18 が使われているとわかった）

# 参考: Gmail を使う準備

.gitconfig に以下を記載

```
[sendemail]
        smtpencryption = tls
        smtpserver = smtp.gmail.com
        smtpuser = ユーザ名@gmail.com
        smtpserverport = 587
```

二段階認証の設定をしている場合は https://support.google.com/mail/answer/185833?hl=ja を参考にアプリパスワードを発行しておく
