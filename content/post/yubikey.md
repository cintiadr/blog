+++
title = "Yubikey setup OSX"
date = "2018-02-07T20:00:20+02:00"
tags = ['security']
draft = true
+++

Stuff.<!--more-->

brew install pinentry-mac
brew install gpgtools

gpg —edit-card

admin

passwd
change pin

change admin pin

generate

$ ssh-add -L
...
The agent has no identities.
$ ssh-add -L
...
ssh-rsa AAAA...yu/V cardno:000...



$ cat ~/.gnupg/gpg-agent.conf
enable-ssh-support
default-cache-ttl 3600
default-cache-ttl-ssh 3600
max-cache-ttl 7200
max-cache-ttl-ssh 7200
pinentry-program /usr/local/bin/pinentry-mac

~/.profile
if ! pgrep -q gpg-agent; then
  gpg-agent --daemon > ~/.gnupg/gpg.info
fi
eval $(cat ~/.gnupg/gpg.info)

https://rnorth.org/gpg-and-ssh-with-yubikey-for-mac
https://www.gnupg.org/howtos/card-howto/en/ch03s03.html
