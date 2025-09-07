---
layout: post
title: "Making ssh-agent Persistent in Windows 11 WSL2 with Keychain"
date:   2025-08-18 19:25:00 +0100
categories: Linux WSL
tags: wsl2 windows ssh keychain
---

# The Problem

If you use `ssh-agent` with an encrypted SSH key, or use it for *agent forwarding*, you've probably noticed that even if you start an agent session with `eval $(ssh-agent -s)`, **it doesn't persist** when you open a new terminal window. It doesn't even work with `tmux`.

# The Solution

Luckily, it's quite simple: **`keychain`**.

Install `keychain`:

```bash
# add keychain
sudo apt-get install keychain
```

Edit your ~/.bashrc, or the corresponding rc file for your shell, and add the following at the end of the file.

> Note: If you're not sure which shell you use, run echo $SHELL and you'll see if you're using bash or zsh.

Here's how my ~/.bashrc looks:

```bash
# To load the SSH key
/usr/bin/keychain -q --nogui $HOME/.ssh/id_rsa
source $HOME/.keychain/$(hostname)-sh
```

The next time you open a new shell, it will ask for your SSH key password just once, and from then on, keychain will take care of loading your key in subsequent sessions.
