---
title: "vscodeのdevcontainer用Imageの作り方"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "Docker", "devcontainer"]
published: true
published_at: 2022-06-12 19:00 
---

vscodeのdevcontainer使ってますか？

もし使っていないなら、プロジェクトに導入するだけで開発効率爆上がりですので、
ぜひ導入しましょう。
これとDockerのExtensionだけでその他のエディタからVscodeに移行するだけの
メリットがあります。
Docker独特のクセのある操作や開発手法をVscodeが吸収してくれます。

## devcontainer用のImageがない場合

MSはdebianやubuntuのOsだけ入っているImageや、
DebianベースのGoやnodejsなどのプログラミング言語が入っているImageなど色々用意しています。

ただ、rhelやOracle linuxなどのOSや
無料だが、再配布禁止のOSを使いたい場合は自分たちでDocker Imageをビルドする
必要があります。
特にRhel系を使うなら、必須の知識でしょう。

日本語や英語で検索してもやり方が間違っているものや、
今はその作りかたで作れないなどがありあるので、

今回はその方法を書きます。

## build方法

### Dockerfile.pre

まずvscodeのImage作成用のDockerfileを準備します。
Dockerfile.preという名前で作成します。

```Dockerfile:Dockerfile.pre
ARG VARIANT="8.6"
FROM container-registry.oracle.com/os/oraclelinux:${VARIANT}

RUN yum update -y

# vscode devcontainer だとプリインストールされているので必須。
RUN yum install git -y

ARG INSTALL_ZSH=false
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN git clone --depth 1 https://github.com/microsoft/vscode-dev-containers.git -b v0.238.0

RUN bash vscode-dev-containers/script-library/common-redhat.sh "${INSTALL_ZSH}" "${USERNAME}" "${USER_UID}" "${USER_GID}" \
    && yum clean all && rm -rf vscode-dev-containers

WORKDIR /workspace
```

devcontainer用のImage作成用のスクリプトはmicrosoftのgithubにあるので、
そこのスクリプトをダウンロードして実行することにより、devcontainer用のImageに調整されます。
[devcontainer repo](https://github.com/microsoft/vscode-dev-containers)
script-library/配下のディレクトリにあります。

スクリプトファイルをタグ指定してgit cloneで持ってきていますが、
そうしないといつのコミットのものなのを使ってDockerfileをビルドしたのか
わからないので、問題発生時の切り分けが困難です。タグ指定したものを持って来る事をおすすめします。
--depth 1という指定はgit cloneでダウンロード速度を上げるために使用しています。
こうしないとgitの履歴と依存関係をすべて持ってくるのでダウンロードにかなり時間がかかります。
今回はいつのtagなのかわかればいいので不要のはずです(~~tagを削除して上書きしてるという邪悪な運用をしてない限り~~)。

今回はrhel系のOracle linuxなのでscript-library/common-redhat.shを実行することになります。
OSごとにどのスクリプトを実行すべきなのかは、script-library/README.mdに書いてあるので、
参照してください。

引数については上のDockerfileを特にいじらずに使えば良いです。
なぜかzshに関するフラグがありますが、普通の人はいらないでしょう。

最後のWORKDIR /workspaceは重要です。
これがないと、.devcontainer作成時にvscodeユーザーが存在しないというエラーが発生します。
おそらくdevcontainerが/workspaceというディレクトリが存在すること前提の作りになっている
のでしょう。

### dockerfile.preビルド用スクリプト

下記のファイルを作成して実行すると良いと思います。
変数をいじることにより、レジストリやバージョンを指定してください。

*再配布禁止のソフトを含むなら必ずPrivateのレポジトリに
アップロードすることには気をつけてください。*

```bash:build_image.sh
#!/bin/bash
#
# build docker image for vscode devcontainer.

set -eu

## variavles
docker_registry=katsutoshiotogawa
docker_image_name=oracle
variant=8.6

# build and push docker iamge.
docker build . \
    -f Dockerfile.pre \
    -t ${docker_registry}/${docker_image_name}:oracle-linux-${variant} \
    --build-arg VARIANT=${variant}

docker image push ${docker_registry}/${docker_image_name}:oracle-linux-${variant} 
```

### devcontainer/Dockerfile

DockerImageのpushが終わったら、.devcontainer/Dockerfile
のFromの部分を上で作成したImageを指定してください。

```Dockerfile:.devcontaienr/Dockerfile
# See here for image contents: https://github.com/microsoft/vscode-dev-containers/tree/v0.238.0/containers/java/.devcontainer/base.Dockerfile

# [Choice] Java version (use -bullseye variants on local arm64/Apple Silicon): 11, 17, 11-bullseye, 17-bullseye, 11-buster, 17-buster
ARG VARIANT="8.6"
FROM katsutoshiotogawa/oracle:oracle-linux-${VARIANT}

# [Option] Install Maven
# ARG INSTALL_MAVEN="false"
# ARG MAVEN_VERSION=""
# # [Option] Install Gradle
# ARG INSTALL_GRADLE="false"
# ARG GRADLE_VERSION=""
# RUN if [ "${INSTALL_MAVEN}" = "true" ]; then su vscode -c "umask 0002 && . /usr/local/sdkman/bin/sdkman-init.sh && sdk install maven \"${MAVEN_VERSION}\""; fi \
#     && if [ "${INSTALL_GRADLE}" = "true" ]; then su vscode -c "umask 0002 && . /usr/local/sdkman/bin/sdkman-init.sh && sdk install gradle \"${GRADLE_VERSION}\""; fi

```

## まとめ

どうでしょう？
情報は少ないですが、割と簡単だったと思います。

これと同じ要領で、本番がvpsみたいにサーバーにdbもアプリケーションも入っている状態で、
本番と同じような構成で開発したいなら、
Docker hubからmysqlやpostgresqlのImageを取ってきて、
それをdevcontainer用のImageに変更するとかなり近い環境が作れると思います。
(systemdやcronだけ困るのでローカルサーバーでは開発だけやって、
systemdやcronの部分は別途開発サーバーでやるとかで解決可能。)

特にdb + ストアドで運用している部分があるかなりの大規模なら開発効率的にも割と有効です。

## reference

[Microsoft devcontainer repo](https://github.com/microsoft/vscode-dev-containers)

[Ian mitchell article](https://ianmitchell.dev/blog/creating-devcontainers-for-vs-code-and-github-codespaces)

## ソースコード

最新のソースコードはgithubにあります。

[create custom devcontainer](https://github.com/KatsutoshiOtogawa/create_custom_devcontainer)
