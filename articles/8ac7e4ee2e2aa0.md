---
title: "簡単Gentoo入門"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gentoo", "systemd", "linux"]
published: false
published_at: 2022-10-22 19:00 
---

vagrant cloudにあるgentooはnomultibuildという
使い辛いものだったりします。

この方法ならkernelのビルドも、ディスクのフォーマット、パーティションもgrubでのOS
の手動インストールも/etc/fstabの設定もいりません。

## ホスト側の設定

Debianやfedoraなど慣れた好きなOSを使ってください。
ネットワークの設定がかなり楽になるので、
ホスト側はsystemd-networkd推奨。
natやブリッジアダプターの設定が不要になる。

## gentooのビルドのダウンロード

systemd-nspawnで設定するので、
Gentooはsystemdのアーカイブを利用してください。

```bash
wget https://ftp.jaist.ac.jp/pub/Linux/Gentoo/releases/amd64/autobuilds/current-stage3-amd64-systemd/stage3-amd64-systemd-20221016T170545Z.tar.xz
# /var/lib/machines/gentooディレクトリが作成され、そこにgentooが展開される。
machinectl import-tar stage3-amd64-systemd-20221016T170545Z.tar.xz gentoo
```

## mirrorの設定

外から設定したほうが楽なのでホストから設定しておく。

```bash
echo 'GENTOO_MIRRORS="rsync://ftp.iij.ad.jp/pub/linux/gentoo/ https://ftp.jaist.ac.jp/pub/Linux/Gentoo/ rsync://ftp.jaist.ac.jp/pub/Linux/Gentoo/ https://ftp.riken.jp/Linux/gentoo/ rsync://ftp.riken.jp/gentoo/"' >> /var/lib/machines/gentoo/etc/portage/make.conf
```

## 仮想環境に入る

systemd-nspawn -D /var/lib/machines/gentoo
と実行して仮想環境に入りましょう。

### rootのパスワードの設定

```bash
passwd

# 作業用一般ユーザーの作成
USER=yourname

# su でrootユーザーに変更できる権限を付与
useradd -m $USER
passwd $USER
# 管理者グループに追加
usermod -aG wheel $USER
```

### make.confの設定

-march=nativeとしておきましょう。

実行時の速度が上がります。

指定されなない場合はx86_64としてビルドされます。
ぶっちゃけ気持ち早くなる程度だと思います。

他のPCに仮想環境を使いまわす場合は
指定しないで置きましょう。

### レポジトリの同期

/var/db/repos/gentooが無いと言われると思いますが、ない場合はちゃんと新たに作られるので大丈夫です。

```bash
emerge-webrsync
```

### profileの決定

systemd (stable)が設定されていることを確認。

eselect profile list

### OSを最新の状態にアップデート

```bash
emerge -uDN @world
```

### localeの設定

gentooでは

```bash
echo en_US.UTF-8 UTF-8 >> /etc/locale.gen
echo ja_JP.UTF-8 UTF-8 >> /etc/locale.gen
# localeの作成
locale-gen

eselect locale list

eselect locale set :w

```

### networkの設定

systemd-network systemd-resolve
は予めインストールされており、サービスとしても有効化されています。

### 仮想環境から抜ける

やり残したことが無いか確認してから、仮想環境から抜けます。

```bash
exit
```

## 仮想環境として使えることの確認

```bash
machinectl start gentoo

machinectl login
```

## ネットワークの疎通確認

```bash
# ipアドレスで会話できることの確認
ping -c 3 8.8.8.8

# 名前解決がなされていることの確認
ping -c www.google.com
```

## バックアップの作成

この状態のgentooを使いまわすなら下のように
backupを取りましょう。

```bash
machinectl export-tar --format=xz gentoo /path/to/gentoo.tar.xz
```

cpでバックアップを取っても良いですが、容量的にはこのようにしたほうが良いかと。

## まとめ

どうでしたか？
debianやfedoraなどインストール難易度が低いLinuxOSをホストとして、
systemd-nspawnを使うとLinuxの知識はあまり問われなくなるので
だいぶ楽だと思います。
systemd-nspawnは仮想環境にしてはかなり軽いので、スペックが低いOS
でも使えるでしょう。

バックアップも外からコピーするだけで済むのでかなり楽です。

もっとGentooの勉強をしたい人は[これ](https://zenn.dev/oto/articles/107a316362eb0f)やGentooのクックブックを参照してください。
