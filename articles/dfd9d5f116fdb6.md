---
title: "dockerでのoracle Application Express(APEX) のローカル環境の作成"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux", "docker", "oracle", "Database", "ローコード"]
published: true
---

ネットであまり、apexでのローカル環境構築方法が出てこず、
公式のgithub, oracle apexのドキュメントでも説明不足なのでここに書く。
今回はdockerかつoracle-21-xeで行う。

## ライセンスについて

Oracleのライセンスは複雑かつ、面倒臭いが、

OTNにあるOracleのソフトウェアは商用利用でなければ、XE以外の
バージョンやソフトもタダで使う事ができる。

今回はXEとApexを使うので制限はつくが、当然商用利用可能。

[oracle container registry](https://container-registry.oracle.com/ords/f?p=113:10::::::)

また、oracle社はOTNライセンスのソフトウェア含めて再配布不可能としているため、
ビルドしたImageを使う場合は、*Imageをprivateなレポジトリ*に置くこと。

## oracleのcontainer registry

下記のレジストリからdocker imageをpullして使うことになる。

[oracle container registry](https://container-registry.oracle.com/ords/f?p=113:10::::::)

## 準備

vscode, dockerをインストールしておく。

vscodeは必須ではないが、Dockerfile, docker-compose.ymlを右クリックから
ビルド、立ち上げができるため、簡単のために導入しておく。

また、[oracle container registry](https://container-registry.oracle.com/ords/f?p=113:10::::::)にログインしておくこと。

## docker compose

```yml:docker-compose.yml
version: '3.8'

volumes:
  oracle-data:

services:

  apex:
    image: container-registry.oracle.com/database/ords:21.4.1
    restart: unless-stopped
    ports:
      - 8181:8181
    depends_on:
      - db
    volumes:
      - ./ords_volume/:/opt/oracle/variables:ro
  db:
    image: container-registry.oracle.com/database/express:21.3.0-xe
    restart: unless-stopped
    container_name: oracle
    shm_size: 2g
    ports:
      - 1521:1521
    volumes:
      - oracle-data:/opt/oracle/oradata
    environment:
      - ORACLE_PWD=manager
      - ORACLE_CHARACTERSET=AL32UTF8
```

## apexからの接続

docker-composeと同じディレクトリに下記の構成で
接続文字列の入ったテキストを置いておくと、
docker-compose起動時に自動的に接続しにいく。

Oracleのドキュメントも分かりづらく、ネットで探してもCDBなのかPDBなのか
はっきりしない記事が多いが、
PDBでしかApexは今は動かないため、CDBでなくてPDBに接続しに行くこと。

```txt:ords_volume/conn_string.txt
CONN_STRING=sys/manager@db:1521/XEPDB1
```

## docker-compose立ち上げ

最初にdocker-composeを立ち上げたとき、
apex側から、oracle dbに接続しに行って、
設定を変えるのでしばらく待つ。
dbの立ち上げ自体にもそれなりに時間かかるので、
合わせて、10分以上待つことになる。

## apex開発環境用DB

http:8181に接続できたら、この状態で一度
oracle dbの方のcontainerをコミットして
imageを作っておくと良いだろう。
apexの初期設定が反映されたoracle dbのコンテナを作ることができる。
これをapex開発環境用DBとすると良い。

```bash
docker container commit <commit id> <tagname>

# example 
docker container commit c3785de katsutoshiotogawa/oracle:21.3.0-xe-apex-dev
```

imageをどこかのレポジトリにpushしないとDockerfile
から参照できないのでpushすること。

ライセンスの問題で必ずprivate レポジトリ。

```bash
docker image push <tagname>

# example
docker image push katsutoshiotogawa/oracle:21.3.0-xe-apex-dev
```

## apexアカウント初期状態

下の設定になる。
パスワードはログインしたら変更させられる。

- Workspace: internal
- User:      ADMIN
- Password:  Welcome_1

## まとめ

情報は少ないが、意外と環境構築が簡単なことがわかると思う。

## ソースコード

最新のソースコードはここにあります。

[github](https://github.com/KatsutoshiOtogawa/oracle_apex_docker)
