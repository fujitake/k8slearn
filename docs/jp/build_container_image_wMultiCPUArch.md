## この記事について

Docker buildxを使ってAMD64とARM64に対応したコンテナイメージを一括作成する方法を紹介します。

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

2021/10/2現在、`RUN apk add --no-cache postgresql-client`がlinux/arm64で失敗することがわかっている

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

*複数のCPUアーキテクチャに対応したイメージを作成しても、ローカルマシンのCPUアーキテクチャが複数対応できるわけではないため、`--push`でリポジトリに登録する方が利便性が高い*

```sh:
docker login quay.io
Username:
Password:

Login Succeeded
```

buildxを使って、コンテナイメージをビルドし、上記にてログインした`quay.io`のリポジトリにイメージをpushする

```sh:
$ docker buildx build --platform linux/amd64,linux/arm64 -t quay.io/fujitake/alpine-f:1.2 --push .
[+] Building 14.3s (12/12) FINISHED                                                                                                             
 => [internal] load build definition from Dockerfile                                                                                       0.0s
 => => transferring dockerfile: 107B                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                          0.0s
 => => transferring context: 2B                                                                                                            0.0s
 => [linux/arm64 internal] load metadata for docker.io/library/alpine:3.7.3                                                                0.6s
 => [linux/amd64 internal] load metadata for docker.io/library/alpine:3.7.3                                                                0.7s
 => [linux/amd64 1/2] FROM docker.io/library/alpine:3.7.3@sha256:8421d9a84432575381bfabd248f1eb56f3aa21d9d7cd2511583c68c9b7511d10          0.0s
 => => resolve docker.io/library/alpine:3.7.3@sha256:8421d9a84432575381bfabd248f1eb56f3aa21d9d7cd2511583c68c9b7511d10                      0.0s
 => [linux/arm64 1/2] FROM docker.io/library/alpine:3.7.3@sha256:8421d9a84432575381bfabd248f1eb56f3aa21d9d7cd2511583c68c9b7511d10          0.0s
 => => resolve docker.io/library/alpine:3.7.3@sha256:8421d9a84432575381bfabd248f1eb56f3aa21d9d7cd2511583c68c9b7511d10                      0.0s
 => CACHED [linux/amd64 2/2] RUN apk add --no-cache curl                                                                                   0.0s
 => CACHED [linux/arm64 2/2] RUN apk add --no-cache curl                                                                                   0.0s
 => exporting to image                                                                                                                    13.5s
 => => exporting layers                                                                                                                    0.0s
 => => exporting manifest sha256:cf384cf772d99bb53542bbdf7c3b57338af9f2662b7009d624ab597e473a3f2e                                          0.0s
 => => exporting config sha256:2e0e076a1f7f1f96ecdcf4b20d697a70a0ea3f0ddcc91f17ac16376737e17fe0                                            0.0s
 => => exporting manifest sha256:c6f5162efa7381a1ad69a59648c1a590ca4652788ec05e7e2b1906ce6c7fcba4                                          0.0s
 => => exporting config sha256:1ec6a3b32601879a3f646c6474e82a0e8aa632b8a386d94a83fa0e780a4b65d6                                            0.0s
 => => exporting manifest list sha256:0cbbab723523cd094d6f18581a8fc6519728f49de97d0da791b79bbba4dae00a                                     0.0s
 => => pushing layers                                                                                                                      7.5s
 => => pushing manifest for quay.io/fujitake/alpine-f:1.2@sha256:0cbbab723523cd094d6f18581a8fc6519728f49de97d0da791b79bbba4dae00a          6.0s
 => [auth] fujitake/alpine-f:pull,push token for quay.io                                                                                   0.0s
 => [auth] fujitake/alpine-f:pull,push token for quay.io                                                                                   0.0s
 => [auth] fujitake/alpine-f:pull,push token for quay.io     
```



## 参考
[Docker docs/Docker buildx](https://matsuand.github.io/docs.docker.jp.onthefly/buildx/working-with-buildx/)
