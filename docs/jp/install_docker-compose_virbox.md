# Docker Compose を使って、Vantiq Edge を起動するための準備


## virtulbox に Docker Engine と Docker Compose (V1) をインストールする手順

## はじめに

virtualbox の Ubuntu の環境に DockerCompose をインストールする手順について記述します。  
このドキュメントの手順は、次の環境において確認しました。
- OS: mac OS Mojave 10.14.6
- VertualBox: 6.1.30 r148432 (Qt5.6.3)
- Ubuntu: 20.04.3 LTS (Focal Fossa)
- Docker Engine: 20.10.11
- Docker Compose: 1.29.2

＊ ここでは、Docker Compose は V1 を対象としています。

## Docker Engine

参考;  
[Docker Engine インストール（Ubuntu 向け）](https://matsuand.github.io/docs.docker.jp.onthefly/engine/install/ubuntu/)  
[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

### インストール手順  
インストール方法はいくつかありますが、推奨されている "Dcoker のリポジトリをセットアップして、そこからインストールする方法" について記載します。

### リポジトリのセットアップ
1. `apt` のパッケージ インデックスを更新します。そして `apt` が HTTPS 経由でリポジトリにアクセスし、パッケージをインストールできるようにます。  

```sh
sudo apt-get update

sudo apt-get install \
>  apt-transport-https  \
>  ca-certificates \
>  curl \
>  gnupg \
>  lsb-release
```

  ＊ 「E: dpkg was interrupted, you must manually run `sudo dpkg --configure -a` to correct the problem.」 というエラーが出力された場合は、`sudo dpkg --configure -a` を実行してください。

2. Docker の公式 GPG キーを追加します。  

```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

3. stable (安定版) のリポジトリを設定します。  

```sh
echo \
>  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
>  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Docker Engine のインストール

1. 次のコマンドでインストールします。

```sh
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io
```

2. Docker Engine が正しくインストールされていることを確認します。

```sh
sudo docker version
```
```sh
sudo docker run hello-world
```

### コマンド実行時のアクセス権の追加
参考;  
[Linux インストール後の作業](https://matsuand.github.io/docs.docker.jp.onthefly/engine/install/linux-postinstall/)  
[Post-installation steps for Linux](https://docs.docker.com/engine/install/linux-postinstall/)

1. ユーザー ("ubuntu") をグループ `docker` に追加し、再起動します。

```sh
sudo groupadd docker

sudo usermod -aG docker ubuntu

reboot
```

  ＊ 仮想環境なので、仮想マシンも再起動する必要があります。

2. docker を sudo 無しで実行できるか確認します。

```sh
docker run hello-world
```
以上で、Docker Engine のインストールができました。

## Docker Compose

参考;  
[Docker Compose のインストール](https://matsuand.github.io/docs.docker.jp.onthefly/compose/install/)  
[Install Docker Compose](https://docs.docker.com/compose/install/)

### インストール手順

1. Dcocker Compose の最新バージョンを確認します (以下の手順は、V1 を対象としています)。  
[こちら](https://github.com/docker/compose/releases) を参照して、インストール対象のバージョンをメモしてください。  

2. Docker Compose の stable (安定版) をダウンロードします。

```sh
 sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
 ```

  ＊ '1.29.2' の箇所は、インストール対象とするバージョンに書き換えてください。  

 3. 実行バイナリに対して実行権限を与えます。

 ```sh
 sudo chmod +x /usr/local/bin/docker-compose
 ```

 4. Docker Compose が正しくインストールされていることを確認します。

```sh
docker-compose --version
```

以上で、Docker Compose のインストールができました。

### Note:
- ここまでの作業 (Docker Engine と Docker Compose のインストール) でおよそ 17GB 使用しています。vertualBox のハードディスク容量は、このことを考慮して設定してください。   
