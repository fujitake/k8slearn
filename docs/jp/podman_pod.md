## この記事について

`podman pod`を`docker-compose`の代わりに使ってみる、ということを目的に試した記録です。

## 前提条件

事前にpodmanをインストール済みであること

## 本文

### podの作成とコンテナのpodへの追加

podを作成

```sh:
podman pod create --name wp-pod -p 8080:80
```

上記で作成したpodにmysqlコンテナを追加

```sh:
podman run \
-d --restart=always --pod=wp-pod \
-e MYSQL_ROOT_PASSWORD="root_pass_fB3uWvTS" \
-e MYSQL_DATABASE="wordpress_db" \
-e MYSQL_USER="user" \
-e MYSQL_PASSWORD="user_pass_Ck6uTvrQ" \
--name db docker.io/mysql:5.7
```

さらにwordpressコンテナを追加、下記のDB_HOSTは、podの名称であり、コンテナ名称ではないことに注意

```sh:
podman run \
-d --restart=always --pod=wp-pod \
-e WORDPRESS_DB_HOST="wp-pod:3306" \
-e WORDPRESS_DB_NAME="wordpress_db" \
-e WORDPRESS_DB_USER="user" \
-e WORDPRESS_DB_PASSWORD="user_pass_Ck6uTvrQ" \
-e VIRTUAL_HOST="apigee.vantiqjp.com" \
--name=wordpress docker.io/wordpress:latest
```

### アクセスを確認

```sh:
curl -L http://localhost:8080
```

上記コマンドの実行結果が下記のような感じになれば、成功なので、ブラウザでも確認して下さい。

`Error establishing a database connection`となる場合は、wordpressコンテナがmysqlコンテナに接続できていないので、DB_HOSTなどを確認すると謎が解けるかと。

```sh:
<!DOCTYPE html>
<html lang="en-US" xml:lang="en-US">
<head>
	<meta name="viewport" content="width=device-width" />
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<meta name="robots" content="noindex,nofollow" />
	<title>WordPress &rsaquo; Installation</title>
```

### 設定した内容を確認

`podman pod ps`の実行結果、コンテナ数が3(自分で追加したものが二つmysqlとwordpress、もう一つはpauseコンテナが動作している)

```sh:
POD ID        NAME        STATUS      CREATED         INFRA ID      # OF CONTAINERS
8ffe8de97da3  wp-pod      Running     13 minutes ago  517cbc37b808  3
```

`podman ps -a`の実行結果

```sh:
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS             PORTS                 NAMES
517cbc37b808  k8s.gcr.io/pause:3.5                                      11 minutes ago  Up 11 minutes ago  0.0.0.0:8080->80/tcp  8ffe8de97da3-infra
6a47dc314ca3  docker.io/library/mysql:5.7         mysqld                11 minutes ago  Up 11 minutes ago  0.0.0.0:8080->80/tcp  db
d95046b996dd  docker.io/library/wordpress:latest  apache2-foregroun...  5 minutes ago   Up 5 minutes ago   0.0.0.0:8080->80/tcp  wordpress
```

`podman ps -a --pod`の実行結果

```sh:
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS             PORTS                 NAMES               POD ID        PODNAME
517cbc37b808  k8s.gcr.io/pause:3.5                                      22 minutes ago  Up 22 minutes ago  0.0.0.0:8080->80/tcp  8ffe8de97da3-infra  8ffe8de97da3  wp-pod
6a47dc314ca3  docker.io/library/mysql:5.7         mysqld                22 minutes ago  Up 22 minutes ago  0.0.0.0:8080->80/tcp  db                  8ffe8de97da3  wp-pod
d95046b996dd  docker.io/library/wordpress:latest  apache2-foregroun...  16 minutes ago  Up 16 minutes ago  0.0.0.0:8080->80/tcp  wordpress           8ffe8de97da3  wp-pod
```

### podの停止と削除の方法

`podman pod ps`にて状態を確認

```sh:
POD ID        NAME        STATUS      CREATED         INFRA ID      # OF CONTAINERS
8ffe8de97da3  wp-pod      Running     29 minutes ago  517cbc37b808  3
```

`podman pod stop wp-pod`にて上記podを停止 (なぜか毎回エラーがでる)

```
ERRO[54427] accept tcp [::]:8080: use of closed network connection
8ffe8de97da381722dc706465d00a2e5df28979f6efae3c42b817bef100784c8
```

`podman pod ps`にて再度確認すると停止(Exited)となっている

```sh:
POD ID        NAME        STATUS      CREATED         INFRA ID      # OF CONTAINERS
8ffe8de97da3  wp-pod      Exited      30 minutes ago  517cbc37b808  3
```

`podman pod rm wp-pod`にて削除

```sh:
8ffe8de97da381722dc706465d00a2e5df28979f6efae3c42b817bef100784c8
```

podを削除すると、`podman ps -a`でもpodに含まれていたコンテナが全て消える

```sh:
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
```

## おまけ、Kubernetes用のyamlを出力

`podman generate kube wp-pod > wp-pod.yaml`にて得られた結果

```yaml:
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-3.4.2
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2021-12-16T15:12:50Z"
  labels:
    app: wp-pod
  name: wp-pod
spec:
  containers:
  - args:
    - mysqld
    image: docker.io/library/mysql:5.7
    name: db
    ports:
    - containerPort: 80
      hostPort: 8080
    resources: {}
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: 51e28f0757850335c9480c23e3874c222512e303c8cb970df913e535161894f8-pvc
  - args:
    - apache2-foreground
    image: docker.io/library/wordpress:latest
    name: wordpress
    resources: {}
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
    volumeMounts:
    - mountPath: /var/www/html
      name: 43e9662be9b425c309c49a19b0e6ba29ed0f0c154cf11c478b776776990f1d9b-pvc
  restartPolicy: Always
  volumes:
  - name: 51e28f0757850335c9480c23e3874c222512e303c8cb970df913e535161894f8-pvc
    persistentVolumeClaim:
      claimName: 51e28f0757850335c9480c23e3874c222512e303c8cb970df913e535161894f8
  - name: 43e9662be9b425c309c49a19b0e6ba29ed0f0c154cf11c478b776776990f1d9b-pvc
    persistentVolumeClaim:
      claimName: 43e9662be9b425c309c49a19b0e6ba29ed0f0c154cf11c478b776776990f1d9b
status: {}

```

## 備考

今回のルートレスモードにて実行したため、以下の制約がある
- well-knownポートの利用ができないこと
- ローカルマシンのボリュームをマウントできない(コンテナ内部にデータを保存)

## 参考

ググると、podman-compose、podmanとdocker-composeなど、色々な情報が出てくるが、`podman pod`の情報を参考に試すと良い。ただし、古い情報もあるため、トライアンドエラーになることが多いかと。。。
