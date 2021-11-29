## この記事について

Docker buildxを使ってAMD64とARM64に対応したコンテナイメージを一括作成する方法を紹介します。

(2021/11/29)`postgresql-client`のインストールがAMD64とARM64の両方でできるように手順を修正

(2021/10/2)この手順でAMD64とARM64に対応したコンテナを作成することができるが、`postgresql-client`のインストールに失敗しており、期待する結果を得られていない。
詳細は[こちら](https://gitlab.alpinelinux.org/alpine/aports/-/issues/12406)を参照


## 前提条件

### ソフトウェア

- AWSにてEC2インスタンスを作成
- Ubuntu 20.04を利用
- インターネットアクセスを有効

## コンテナイメージのビルド

### コンテナイメージをビルドするための準備

Dockerfileを用意

2021/11/29、`multiarch/qemu-user-static`を使うことでlinux/arm64のビルドが成功することを確認

~~2021/10/2現在、`RUN apk add --no-cache postgresql-client`がlinux/arm64で失敗することがわかっている~~

```sh:
FROM alpine:3.8.5
RUN apk add --no-cache curl
RUN apk add --no-cache postgresql-client
ENTRYPOINT ["/bin/ash"]
```

### buildxを使ってマルチCPUアーキテクチャに対応したイメージを作成

ubuntuにdockerをインストールした場合、`buildx`が含まれないため、個別にインストールする必要がある

(2021/10/2)公式ドキュメントには、debでインストールすると`buildx`は含まれるという記載がdocker docsにあったが、試したところ、含まれていなかった

[docker/buildx](https://github.com/docker/buildx/releases)ページから自分が使っているUbuntuのCPUアーキテクチャのバイナリファイルのURLを取得

```sh:
$ curl -OL https://github.com/docker/buildx/releases/download/v0.6.3/buildx-v0.6.3.linux-amd64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   633  100   633    0     0   2705      0 --:--:-- --:--:-- --:--:--  2705
100 58.4M  100 58.4M    0     0  22.1M      0  0:00:02  0:00:02 --:--:-- 26.8M
```

ダウンロードした `buildx` を `~/.docker/cli-plugins/` に移動し、実行権限を付与

```sh:
mkdir ~/.docker
mkdir ~/.docker/cli-plugins
mv buildx-v0.6.3.linux-amd64 ~/.docker/cli-plugins/docker-buildx
chmod a+x ~/.docker/cli-plugins/docker-buildx
```

`buildx` を設定。

```sh:
docker buildx install
```

AMD64とARM64に対応したビルダーを作成

```sh:
docker buildx create --name multiarch --driver docker-container --platform linux/amd64,linux/arm64
```

作成したビルダー(下記では、multiarchがビルダー名)を確認

```sh:
$ docker buildx ls
NAME/NODE    DRIVER/ENDPOINT             STATUS   PLATFORMS
multiarch    docker-container                     
  multiarch0 unix:///var/run/docker.sock inactive linux/amd64*, linux/arm64*
default *    docker                               
  default    default                     running  linux/amd64, linux/386
```

上記にて作成したビルダーを使うよう指定

```sh:
docker buildx use multiarch
```

作成したビルダーの起動を確認

```sh:
$ docker buildx inspect --bootstrap
[+] Building 8.1s (1/1) FINISHED                                                                                                                
 => [internal] booting buildkit                                                                                                            8.0s
 => => pulling image moby/buildkit:buildx-stable-1                                                                                         7.1s
 => => creating container buildx_buildkit_multiarch0                                                                                       0.9s
Name:   multiarch
Driver: docker-container

Nodes:
Name:      multiarch0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Platforms: linux/amd64*, linux/arm64*, linux/386
```

コンテナイメージをpushするコンテナレジストリを指定してログイン

*複数のCPUアーキテクチャに対応したイメージを作成して`--push`でリポジトリに登録する方が利便性が高い、というかそれ以外に利用シーンがあるのかな？*

```sh:
docker login quay.io
Username:
Password:

Login Succeeded
```

multiarch/qemu-user-staticをインストール、これがないと違うCPU Arch上で動作させるDockerfile内のRUNコマンド失敗する

```sh:
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

buildxを使って、コンテナイメージをビルドし、上記にてログインした`quay.io`のリポジトリにイメージをpushする

```sh:
$ docker buildx build --platform linux/amd64,linux/arm64 -t quay.io/fujitake/alpine-f:1.3 --push .
[+] Building 20.1s (11/11) FINISHED                                                                                                                                    
 => [internal] load build definition from Dockerfile                                                                                                              0.0s
 => => transferring dockerfile: 125B                                                                                                                              0.0s
 => [internal] load .dockerignore                                                                                                                                 0.0s
 => => transferring context: 2B                                                                                                                                   0.0s
 => [linux/arm64 internal] load metadata for docker.io/library/alpine:3.8.5                                                                                       0.6s
 => [linux/amd64 internal] load metadata for docker.io/library/alpine:3.8.5                                                                                       0.6s
 => CACHED [linux/amd64 1/2] FROM docker.io/library/alpine:3.8.5@sha256:2bb501e6173d9d006e56de5bce2720eb06396803300fe1687b58a7ff32bf4c14                          0.0s
 => => resolve docker.io/library/alpine:3.8.5@sha256:2bb501e6173d9d006e56de5bce2720eb06396803300fe1687b58a7ff32bf4c14                                             0.0s
 => CACHED [linux/arm64 1/2] FROM docker.io/library/alpine:3.8.5@sha256:2bb501e6173d9d006e56de5bce2720eb06396803300fe1687b58a7ff32bf4c14                          0.0s
 => => resolve docker.io/library/alpine:3.8.5@sha256:2bb501e6173d9d006e56de5bce2720eb06396803300fe1687b58a7ff32bf4c14                                             0.0s
 => [linux/amd64 2/2] RUN apk add --no-cache curl postgresql-client                                                                                               1.9s
 => [linux/arm64 2/2] RUN apk add --no-cache curl postgresql-client                                                                                               3.9s
 => exporting to image                                                                                                                                           15.4s
 => => exporting layers                                                                                                                                           2.2s
 => => exporting manifest sha256:05e2c16a122ab894e5b5edd9004bdf32b65ba4cf1094dd54164b5fa3db6d08ce                                                                 0.0s
 => => exporting config sha256:6d570cfd805b737200755ad531a2d45e45a293cc6a4252daf8e78e7fbda05e0f                                                                   0.0s
 => => exporting manifest sha256:01c2f539f236443dcaa90bcb4ef812946f260b050d54069238b9eebc638f4ea0                                                                 0.0s
 => => exporting config sha256:936bdd8c8d01cd56e777b43f2b85cf158c98708979d8f11d9a9b6d8af587c6e0                                                                   0.0s
 => => exporting manifest list sha256:d3971d046321f32b2606c072c4272ea3acf87142d08137460e0753c43b0e4968                                                            0.0s
 => => pushing layers                                                                                                                                             6.6s
 => => pushing manifest for quay.io/fujitake/alpine-f:1.3@sha256:d3971d046321f32b2606c072c4272ea3acf87142d08137460e0753c43b0e4968                                 6.5s
 => [auth] fujitake/alpine-f:pull,push token for quay.io                                                                                                          0.0s
 => [auth] fujitake/alpine-f:pull,push token for quay.io
```



## 参考
[Docker docs/Docker buildx](https://matsuand.github.io/docs.docker.jp.onthefly/buildx/working-with-buildx/)

multiarchで対応する場合は、これがキモ
[multiarch/qemu-user-static](https://github.com/multiarch/qemu-user-static)

こっちでもできるかも
[tonistiigi/binfmt](https://github.com/tonistiigi/binfmt#installing-emulators)
