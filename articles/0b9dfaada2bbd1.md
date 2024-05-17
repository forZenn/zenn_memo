---
title: "vagrant仮想環境でのOracle Rac サーバーの構築"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux", "vagrant", "oracle", "Database"]
published: true
---

oracle racサーバーと同じ環境をローカルマシンで作るのはかなり大変なため、
oracleはgithub上で簡単に環境構築するためのプロジェクトをいくつか作ってある。
ここではそのうち、RACサーバーの構築方法を書きます。

## ライセンスについて

Oracleのライセンスは複雑かつ、面倒臭いが、

OTNにあるOracleのソフトウェアは商用利用でなければ、XE以外の
バージョンやソフトもタダで使う事ができる。

今回使うGrid Infrastructureやoracle database(EE)もタダ。

## oracleのgithub

dockerとvagrant両方用意されているが、
dockerのほうが必要なメモリ要件が大きく、
各コンテナ8GB最低必要になる。
つまりRACにすると仮想環境だけで最低16GBのため、
16GBのマシンで動かせず、32GB必要になってしまう...

なのでここではvagrantの方のプロジェクトの説明をします。

```bash
gh repo clone oracle/vagrant-projects oracle-vagrant-projects
```

## ハードウェア要件

### メモリ

各インスタンス最低6GB必要なため、RACでやるなら、
仮想環境だけで12GB必要ということである。
つまり16GB以上のパソコンで無いとできない。
筆者はmac bookproの16GBで作業しているが、vscodeとブラウザを複数立ち上げると
たまに止まります...

### ディスク

空き容量もかなり大きい。80GBは余裕を持っていたほうが良いかと。

## 準備

cd oracle-vagrant-projects/OracleRAC

### mlocate

oracleのファイル構成が複雑なので、
検索を楽にするためにもインストールして置くと楽。
パッケージの容量少ないし、脆弱性出て外から攻撃できるような機能でもない。

下記のようにパッケージを追加しておくと便利。

```bash:scripts/02_install_os_packages.sh
...

# add package for searching file
yum install -y mlocate

...
```

### ソフトウェアダウンロード

公式レポジトリのREADMEに書かれている通り、

ORCL_software/配下に
Oracle Database 21c Grid Infrastructureと
Oracle Database 21c (21.3) for Linux x86-64
を置いておく。

尚、Oracle RAC  はOracele DatabaseがEEのバージョンで無いと
使うことができないので注意すること。

### 設定ファイル

config/vagrant.ymlに設定があるので、
それを変更すると良い。

## vagrant起動

下記のコマンドを押して起動。
1hぐらいかかるかも。

```bash
vagrant up
```

仮想環境立ち上げ後、

mlocateをインストールした場合は下記のコマンドを実行して
locateで検索できるようにしておくと楽です。

```bash
vagrant ssh node1 -c 'sudo updatedb'
vagrant ssh node2 -c 'sudo updatedb'
```

## 作成されるユーザー

oracle, gridが作成されるので、
仮想環境に直接入って操作する場合は下記のようにユーザーを変更して操作してください。

```bash
# node1の仮想環境に入る場合
vagrant ssh node1
sudo su - oracle
```

/home/oracle/.bash_profile
/home/grid/.bash_profile
にそれぞれORACLE_HOMEなどの設定が書かれており、
自動的に読み込まれます。

## ホストからポートなど疎通確認

nmapを使って下記のように開いているポートを確認すると、
初期状態では以下のポートが開いている事がわかります。

```bash
nmap 192.168.56.111

PORT     STATE SERVICE
22/tcp   open  ssh
111/tcp  open  rpcbind
1521/tcp open  oracle
5000/tcp open  upnp
8888/tcp open  sun-answerbook
```

## 接続方法

sqlplus, sqlclを使った方法を書く。
特に理由が無い限りsqlclを使うべきです。

昔ながらのsqlplusはcppで作られているため、自分で考えてzipを展開配置するなら
LD_LIBRARAY_PATHを考えて配置する必要があるため、CPPの知識が無い人からするとやや上級者向けです。
また、機能も貧弱です。

sqlclは最近Oracleに開発されたもので、一部cppの共有ライブラリがありますが、ほぼJavaのライブラリなので、
扱いやすく、共有ライブラリの配置に気を使わなくて良いようになっており、パスを通すだけでOS問わず使うことができます。

また、機能も豊富です。

後々、sqlplusは非推奨になるであろうと思われるので、早めにsqlclに移行しましょう。

### 直接指定する

ホスト名、サービス名(データベース名)
この場合は各ノードを指定して接続する事になる。

```bash
sqlplus username/passoworrd@hostname:port/servicename
sqlcl username/passoworrd@hostname:port/servicename

# example
sqlplus sys/welcome1@192.168.56.121 as sysdba
```

### tnsnames.oraを使う場合

クライアントPC側に下のようにtnsnames.oraのファイルをおいておきます。
サービス名には接続するデータベース名を書くこと。

```bash:TNS_ADMIN/tnsnames.ora
vagrant_cluster =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.56.111)(PORT = 1521))
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.56.121)(PORT = 1521))
    (CONNECT_DATA=(SERVICE_NAME=DB213H1))
  )
```

環境変数TNS_ADMINを設定していない場合、
\$ORACLE_HOME/network/admin/が使われる。

なので、TNS_ADMINか、ORACLE_HOMEどちらかの環境変数を予め設定しておく必要がある。

クライアント側は下のようにプロファイルに書くのが一般的だろう。

```bash:/.bashrc
export ORACLE_HOME=/home/user/Oracle
```

こうするとvagrant_clusterという接続名で接続できる。

```bash
sqlplus username/passoworrd@vagrant_cluster
sqlcl username/passoworrd@vagrant_cluster


# example
sqlplus sys/welcome1@vagrant_cluster as sysdba
```

## 参考

[Oracle rac ドキュメント](https://www.oracle.com/technetwork/jp/ondemand/db-technique/b-3-rac-1484714-ja.pdf)
[Oracle vagrant project](https://github.com/oracle/vagrant-projects)
