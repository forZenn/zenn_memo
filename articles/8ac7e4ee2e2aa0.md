---
title: "簡単Gentoo導入"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gentoo", "systemd", "linux"]
published: false
---


```bash
wget https://ftp.jaist.ac.jp/pub/Linux/Gentoo/releases/amd64/autobuilds/current-stage3-amd64-systemd/stage3-amd64-systemd-20221016T170545Z.tar.xz
machinectl import-tar stage3-amd64-systemd-20221016T170545Z.tar.xz gentoo
```

```bash

echo 'GENTOO_MIRRORS="rsync://ftp.iij.ad.jp/pub/linux/gentoo/ https://ftp.jaist.ac.jp/pub/Linux/Gentoo/ rsync://ftp.jaist.ac.jp/pub/Linux/Gentoo/ https://ftp.riken.jp/Linux/gentoo/ rsync://ftp.riken.jp/gentoo/"' >> /etc/portage/make.conf

emerge-webrsync
eselect profile list

```