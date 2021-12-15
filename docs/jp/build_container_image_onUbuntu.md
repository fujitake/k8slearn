## この記事について

DockerをEC2インスタンスにインストールし、自分でコンテナイメージをbuildする方法を紹介します。

ついでに`docker-compose`(v2ではない方)のインストール方法を紹介します。

## 前提条件

### ソフトウェア

- AWSにてEC2インスタンスを作成
- Ubuntu 20.04を利用
- インターネットアクセスを有効

## docker engine インストール手順

[Docker Engineの公式インストール手順](https://docs.docker.com/engine/install/ubuntu/)があります。

apt/apt-getでインストールする場合は、上記公式手順の通りに進めてください。常に最新がインストールされます。

`package.deb`を使ってインストールする場合は、説明がないと少しわかりにくいので、参考として紹介します。

### debを使ったdocker engineのインストール

[こちら](https://docs.docker.com/engine/install/ubuntu/)にある`Install from a package`でインストールする方法もある。

利用しているubuntuのversionを確認

```sh:
$ lsb_release -a

No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 20.04.2 LTS
Release:	20.04
Codename:	focal
```

[ここにアクセス](https://download.docker.com/linux/ubuntu/dists/)し、上記versionに適した`Packages`ファイルをダウンロード

テキストファイルになっており、中に `docker-ce, docker-ce-cli, containerd.io`のダウンロード用URLが記載されている。複数バージョン含まれており、適宜選択してダウンロードしてください。stableが安定版です。testとnightlyが開発中です。

debファイルを3つダウンロードする。

```sh:
curl -OL https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/docker-ce-cli_19.03.15~3-0~ubuntu-focal_amd64.deb
curl -OL https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/docker-ce_19.03.15~3-0~ubuntu-focal_amd64.deb
curl -OL https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/containerd.io_1.4.10-1_amd64.deb
```

それぞれインストール、docker-ceをインストールするにはcontainerdを先にインストールする必要がある。

```sh:
sudo dpkg -i docker-ce-cli_19.03.15~3-0~ubuntu-focal_amd64.deb
sudo dpkg -i containerd.io_1.4.10-1_amd64.deb
sudo dpkg -i docker-ce_19.03.15~3-0~ubuntu-focal_amd64.deb
```

### dockerのインストール後の設定

今の状態では`docker info`などのコマンド実行時のアクセス権限がないため、ユーザ `ubuntu` をグループ 'docker' に追加し、dockerを再起動

確認

```sh:
cat /etc/group |grep docker
```


```sh:
sudo groupadd docker
sudo usermod -aG docker ubuntu
newgrp docker
```

## コンテナイメージのビルド

### コンテナイメージをビルドするための準備

Dockerfileを用意します。

```sh:
FROM alpine:3.13.6
RUN apk add --no-cache curl
RUN apk add --no-cache postgresql-client
ENTRYPOINT ["/bin/ash"]
```

### コンテナイメージのローカルでのbuild方法

Dockerfileがあるディレクトリもしくは、ファイルを指定して、imageをbuild

```sh:
$ docker image build -t fujitake/alpine-f:1.2 .
Sending build context to Docker daemon  2.048kB
Step 1/4 : FROM alpine:3.13.6
3.13.6: Pulling from library/alpine
4e9f2cdf4387: Pull complete
Digest: sha256:2582893dec6f12fd499d3a709477f2c0c0c1dfcd28024c93f1f0626b9e3540c8
Status: Downloaded newer image for alpine:3.13.6
 ---> 12adea71a33b
Step 2/4 : RUN apk add --no-cache curl
 ---> Running in 89750020a587
fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/community/x86_64/APKINDEX.tar.gz
(1/5) Installing ca-certificates (20191127-r5)
(2/5) Installing brotli-libs (1.0.9-r3)
(3/5) Installing nghttp2-libs (1.42.0-r1)
(4/5) Installing libcurl (7.79.1-r0)
(5/5) Installing curl (7.79.1-r0)
Executing busybox-1.32.1-r6.trigger
Executing ca-certificates-20191127-r5.trigger
OK: 8 MiB in 19 packages
Removing intermediate container 89750020a587
 ---> aea65114326e
Step 3/4 : RUN apk add --no-cache postgresql-client
 ---> Running in 139dbe629058
fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/community/x86_64/APKINDEX.tar.gz
(1/8) Installing ncurses-terminfo-base (6.2_p20210109-r0)
(2/8) Installing ncurses-libs (6.2_p20210109-r0)
(3/8) Installing libedit (20191231.3.1-r1)
(4/8) Installing gdbm (1.19-r0)
(5/8) Installing libsasl (2.1.27-r10)
(6/8) Installing libldap (2.4.57-r1)
(7/8) Installing libpq (13.4-r0)
(8/8) Installing postgresql-client (13.4-r0)
Executing busybox-1.32.1-r6.trigger
OK: 12 MiB in 27 packages
Removing intermediate container 139dbe629058
 ---> 551f5cccfa90
Step 4/4 : ENTRYPOINT ["/bin/ash"]
 ---> Running in c68bd00087c8
Removing intermediate container c68bd00087c8
 ---> ec7088e17071
Successfully built ec7088e17071
Successfully tagged fujitake/alpine-f:1.2
```

imageの確認

```sh:
$ docker image ls
REPOSITORY          TAG       IMAGE ID       CREATED         SIZE
fujitake/alpine-f   1.2       ec7088e17071   3 minutes ago   12.1MB
alpine              3.13.6    12adea71a33b   4 weeks ago     5.61MB
```

## docker-composeのインストール方法

### インストールの条件

docker engineがインストール済みであること

dockerがルートではないユーザで実行できること

### インストール方法

下記コマンドにて、`linux/x86_64のv1.29.2`をインストールできます。

```sh:
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 確認

```sh:
$ docker-compose version
docker-compose version 1.29.2, build 5becea4c
docker-py version: 5.0.0
CPython version: 3.7.10
OpenSSL version: OpenSSL 1.1.0l  10 Sep 2019
```

## 参考
[Docker docs/Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

[Docker docs/Install Compose](https://docs.docker.com/compose/install/)

[Docker Compose Releases](https://github.com/docker/compose/releases)
