---
title: "manjaroのbootstrapを作ってみた。"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux", "仮想環境", "Arch Linux", "manjaro"]
published: true
---

## bootstrapとは？

既にosが入っているPCにchroot仮想環境を使って、osをインストールするときの処理を手伝ってくれるもの。
一般に/mnt/osnameにディレクトリを作って、そこに/dev/sdxなどデバイスをマウントさせて、そこにインストールするという手法です。
chroot環境はvagrant,docker環境に比べて要求されるスペックが低いので、
メモリ4Gなどの低スペックでも仮想環境を楽に動かすことができます。

## manjaroのbootstrapが古くdefualtはunstable

manjaroにはmanjaro-bootstrapがあるが、
manjaro-bootstrapのプロジェクトのソースコードはgithubとgitlabにあり、
github側は既にarchive済み、gitlabのソースコードはブランチがunstableに決めうちされていて、
どちらかというと、manjaro linux開発者用のものだろう。

なので、一般manjaroユーザー用のbootstrapを作るなら、自分で書く必要があります。
0から書くのは大変なので、manjaroのupstreamにあたるarch linux のbootstrap
を参考にしたい。

## manjaro-bootstrap

archlinuxにはarch-bootstrapというbootstrap用のスクリプトが
あり、archwikiにgithubへのリンクがある。
[archwiki](https://wiki.archlinux.jp/index.php/%E6%97%A2%E5%AD%98%E3%81%AE_Linux_%E3%81%8B%E3%82%89%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB)
[github arch-bootstrap](https://github.com/tokland/arch-bootstrap)

これをフォークしてmanjaro用のbootstrapを作った。
[manjaro-bootstrap](https://github.com/KatsutoshiOtogawa/manjaro-bootstrap)

## 要件

ホスト側がlinux osであること。

## インストール方法

githubからダウンロードしてきて下記のコマンドで適切な場所にインストールしてください。

```bash
wget https://raw.githubusercontent.com/KatsutoshiOtogawa/manjaro-bootstrap/main/manjaro-bootstrap.sh
sudo install -m 755 manjaro-bootstrap.sh /usr/local/bin/manjaro-bootstrap
```

## 使い方

基本的にはフォーク元のarchlinux-bootstrapと使い方同じです。

```bash
# ヘルプ表示
manjaro-bootstrap -h
```

使い方流れ

```bash
# mount用ディレクトリ作成
sudo mkdir /mnt/manjaro
# デバイスファイルにマウント(あらかじめext4などでフォーマットしておくこと。)
sudo mount /dev/sdbx /mnt/manjaro
# インストール
sudo manjaro-bootstrap -r repositoryのurl -a x86_64などcpu /mnt/manjaro
# プロセスやデバイスファイルなどをchroot環境にマウント
sudo mount --types proc /proc myarch/proc
sudo mount --rbind       /sys myarch/sys
sudo mount --make-rslave      myarch/sys
sudo mount --rbind       /dev myarch/dev
sudo mount --make-rslave      myarch/dev
sudo mount --bind        /run myarch/run
sudo mount --make-slave       myarch/run

# chroot
sudo /mnt/manjaro /bin/bash
pacman -S manjaro_release # など自分に必要なものをインストールしていく。
...
```

chroot抜ける場合。

```bash
$ exit
# chroot抜けた後
$ sudo umount -l /mnt/manjaro/dev{/pts,/shm,}
$ sudo umount -R /mnt/manjaro
```
