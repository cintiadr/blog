+++
title = "Setting SSH keys from Yubikey (OSX)"
date = "2018-10-01T00:00:20+02:00"
tags = ['security', 'yubikey']
draft = false
+++

I purchased a [yubikey](https://www.yubico.com/) a few months ago,
as I needed a secure way to 'carry' by ssh private keys with me.

There are several tutorials on the internet, but [none](https://rnorth.org/gpg-and-ssh-with-yubikey-for-mac) of [them](https://www.gnupg.org/howtos/card-howto/en/ch03s03.html) actually worked for me (on OSX). So let me write down all the steps I took, so my future self will know what to do.

<!--more-->

I use [homebrew](https://brew.sh/), so I installed the following packages:

```
brew install pinentry-mac
brew install gpgtools
```

Now you need to configure your yubikey. You need to change the yubikey pin, the yubikey admin pin and generate a GPG key pair.

Insert your yubikey to your USB and run:

```
gpg —edit-card
 > admin
 > passwd
 > change pin
 > change admin pin
 > generate
```

Make sure to save those pins in a safe location (e.g. your password manager).

Remove your yubikey.

Configure the gpg-agent:

```
$ cat ~/.gnupg/gpg-agent.conf
enable-ssh-support
default-cache-ttl 3600
default-cache-ttl-ssh 3600
max-cache-ttl 7200
max-cache-ttl-ssh 7200
pinentry-program /usr/local/bin/pinentry-mac
```

_Important lines: to enable ssh support (from GPG agent) and which software to use to
type pin (`pinentry`)._


Configure your `~/.profile` to contain the following lines:

```
if ! pgrep -q gpg-agent; then
  gpg-agent --daemon > ~/.gnupg/gpg.info
fi
eval $(cat ~/.gnupg/gpg.info)
```

Open a new terminal.
If you type

```
$ ssh-add -L
...
The agent has no identities.
```
No keys are available.

If you add your yubikey, the ssh agent should add a new identity, and the public key:

```
$ ssh-add -L
...
ssh-rsa AAAA...yu/V cardno:000...
```

If you attempt to use the key, `pinentry` popup will show up, and you'll have to type the
pin to unlock the yubikey and use the private key.
