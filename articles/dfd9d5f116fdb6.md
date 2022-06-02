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

```txt:ords_volume/conn_string.txt
CONN_STRING=sys/manager@db:1521/XEPDB1
```

## まとめ

情報は少ないが、意外と環境構築が簡単なことがわかると思う。
