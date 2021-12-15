## はじめに

この資料は、docker-composeを使ったweb server + DBシステムのSSL/TLS Proxy対応のガイドです。

## 前提条件
### ソフトウェア

- Linuxマシンにdocker engineとdocker-composeを用意していること
- インターネットアクセスが可能であること
- ドメインを所有しており、FQDNを決めることができること
- 上記ドメインのDNSゾーン設定ができること

### 事前設定

取得済みのドメインを利用したFQDNを決めており、そのFQDNに対応したサーバ証明書を用意する

テストを目的とした3ヶ月間有効なサーバ証明書でもよければ、[こちら](https://github.com/fujitake/k8slearn/blob/main/docs/jp/how_to_create_ssl_certificate.md)を参照して作成下さい

### 必要な環境の用意

ワーキング ディレクトリを作成し`compose.yaml` (推奨とのこと)を用意する。`compose.yml`でも動作する。古い資料では`docker-compose.yaml`もしくは`docker-compose.yml`を使うとガイドされているケースが多いが、後方互換性のためのサポートとなっている。複数のファイルがある場合`compose.yaml`が優先される。

```sh:
mkdir working-dir
cd working-dir
touch compose.yaml
vi compose.yaml
```

下記ファイルを`compose.yaml`として用意

ちなみに、ググると様々なサンプルが見つかるが、すでに廃止された`version`が残っていたりするケースが多く、見つかる。[最新の仕様はこちら(英語のみ)](https://github.com/compose-spec/compose-spec/blob/master/spec.md)です。

```yaml:
services:
  db:
    image: mysql:5.7
    #container_name: "mysql57"
    volumes:
      - ./db/mysql:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root_pass_fB3uWvTS
      MYSQL_DATABASE: wordpress_db
      MYSQL_USER: user
      MYSQL_PASSWORD: user_pass_Ck6uTvrQ
    networks:
      - back

  wordpress:
    image: wordpress:latest
    #container_name: "wordpress"
    volumes:
      - ./wordpress/html:/var/www/html
      - ./php/php.ini:/usr/local/etc/php/conf.d/php.ini
    restart: always
    depends_on:
      - db
    expose:
      - "8080"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress_db
      WORDPRESS_DB_USER: user
      WORDPRESS_DB_PASSWORD: user_pass_Ck6uTvrQ
      VIRTUAL_HOST: apigee.vantiqjp.com
    networks:
      - front
      - back

  nginx-proxy:
    image: jwilder/nginx-proxy:latest
    container_name: nginx-proxy
    privileged: true
    ports:
      - "443:443"
    volumes:
      - ./data/htpasswd:/etc/nginx/htpasswd
      - ./data/conf.d:/etc/nginx/conf.d
      - /etc/nginx/vhost.d
      - /usr/share/nginx/html
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - ./certs:/etc/nginx/certs
    restart: always
    networks:
      - front

networks:
  front:
    driver: bridge
  back:
    driver: bridge
```

ワーキング ディレクトリにてディレクトリ`certs`を作成

```sh:
mkdir certs
```

別途作成した証明書を`compose.yaml`のVIRTUAL_HOSTにて指定したホスト名(FQDN)に合わせて`fqdn.crt`とする。秘密鍵は、`fqdn.key`とする。

```sh:
cp fullchain1.pem apigee.vantiqjp.com.crt
cp privkey1.pem apigee.vantiqjp.com.key
```

### コンテナの起動

用意した環境を起動するには、ワーキング ディレクトリにて下記コマンドを実行する。

```sh:
docker-compose up -d
```

### アクセスの確認

下記コマンドにて`docker-compose`の起動状態の確認ができる。

```sh:
docker-compose ps
       Name                     Command               State                      Ports                    
----------------------------------------------------------------------------------------------------------
nginx-proxy          /app/docker-entrypoint.sh  ...   Up      0.0.0.0:443->443/tcp,:::443->443/tcp, 80/tcp
nx-img_db_1          docker-entrypoint.sh mysqld      Up      3306/tcp, 33060/tcp                         
nx-img_wordpress_1   docker-entrypoint.sh apach ...   Up      80/tcp, 8080/tcp      
```

次にhttpsによるアクセスを確認。DNS設定等を行う前に以下のコマンドにて確認する。`-k`を付与しているため、ホスト名と証明書のFQDNが合致しなくても結果が返る。同様に`-H`にてホスト名を指定している。

```sh:
curl -L -k -H "Host: apigee.vantiqjp.com" https://localhost
```

DNSにAレコードとして登録するか、`/etc/hosts`に`127.0.0.1 FDQN`を追加し、名前解決ができるように設定する。

```sh:
curl -L https://apigee.vantiqjp.com
```

下記結果の例のように結果が適切に得られた場合、作業終了となる。

```sh:
<!DOCTYPE html>
<html lang="en-US" xml:lang="en-US">
<head>
	<meta name="viewport" content="width=device-width" />
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<meta name="robots" content="noindex,nofollow" />
	<title>WordPress &rsaquo; Installation</title>
以下省略
```


### コンテナの停止

起動した環境の停止を行う場合は下記コマンドで実行可能

```sh:
docker-compose down
```

## 参考

composeの最新仕様はこちらを確認する方が確実です、英文ですが。

[compose specification](https://github.com/compose-spec/compose-spec/blob/master/spec.md)

今回利用したnginx-proxyは下記になります。設定に関する情報も記載があります。

[jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy)

このガイドでは、利用しておりませんが、下記を使えば、Let's Encryptの登録、自動更新ができるようです。必要に応じて、お試し下さい。

[nginxproxy/acme-companion](https://hub.docker.com/r/nginxproxy/acme-companion)
