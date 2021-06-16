## はじめに
この手順を辿れば、ARM64 Kubernetes上でDocker Official ImageからデータベースPodを起動できます。

## この記事について
実践的にKubernetesを学習できればと思い、Pod作成の記事を書いてみました。

## 前提条件
この記事を書く時に使った環境は下記の通りです。

- [K3s: v1.20.6+k3s1](https://k3s.io)
- Raspberry Pi 4B 4GB Memory
- インターネットアクセス環境
- LinuxをCLIのみで触る程度の知識か根性

### ソフトウェア
- K3s: [v1.20.6+k3s1](https://k3s.io)  
- MongoDB: 4-bionic (docker.io/mongo:4-bionic)
- 本記事では32bit Raspbian OSやWindows環境、Mac環境は対象としていません、というか確認していません。

## 事前確認事項の整理
### ハードウェアのシステムアーキテクチャを確認
データベースPodをセットアップする対象がX86-64(AMD64)なのか、ARM64なのか確認  
Intel CPUやAMD CPUの場合はAMD64、Raspberry Piの場合はARM64  
コンテナイメージが対応していないシステムアーキテクチャのPodは起動しません。注意が必要です。

### Official Docker Imageの対応状況を確認
目的とするコンテナイメージが自分の持っているKubernetesシステムアーキテクチャに対応することを確認

docker hubでmysqlを検索すると、x86−64に対応することは確認できますが、ARM64対応は記載がありません。
![dockerlofficialimage_mysql.png](../../imgs/dockerlofficialimage_mysql.png)

同じくdocker hubでmongoを検索してみました。こちらはx86-64とARM64の両方に対応していることが確認できました。
![dockerlofficialimage_mongo.png](../../imgs/dockerlofficialimage_mongo.png)

### 提供されているDatabaseのバージョンを確認
目的とするDatabaseのバージョンを検索するなどして確認

同じくdocker hubでmongoの特定バージョンを検索した結果です。同じバージョンで、複数のアーキテクチャに対応したイメージを提供していることが確認できます。
![dockerhubtag_mongo.png](../../imgs/dockerhubtag_mongo.png)

同じくdocker hubでmysqlの特定バージョンを検索した結果です。amd64(x86-64)しか見つからないことが確認できます。
ということは、Raspberry Piで作ったKubernetesでは動作しない、ということになります。
![dockerhubtag_mysql.png](../../imgs/dockerhubtag_mysql.png)

### ソフトウェア提供元が定義しているサポートバージョンを確認
この手順は、オプションであり、スキップ可能ですが、知っておいた方が良いと思います。

ソフトウェアのサポートプラットフォームとバージョン及び依存関係を確認
ソフトウェア提供元が提供している、プロダクション環境で動作させる上での推奨プラットフォームとバージョンに関する情報を確認しまします。`supported version mysql`もしくは`suppored platforms mysql`などで検索すると見つけやすいです。
また、下記はいずれもコンテナ化した場合のサポート情報ではありません。

[MySQLのサポートプラットフォーム情報](https://www.mysql.com/support/supportedplatforms/database.html)
![suportedpf_mysql.png](../../imgs/suportedpf_mysql.png)

[MongoDBの推奨プラットフォーム情報/ARM64](https://docs.mongodb.com/manual/administration/production-notes/#std-label-prod-notes-recommended-platforms)
![suportedpf_mongo.png](../../imgs/suportedpf_mongo.png)

## デプロイの準備と実行
### 各コンテナイメージの環境変数(Environment Variables)を確認
コンテナイメージを起動する際に初期値として渡す環境変数の条件を確認します。コンテナイメージを作った時に決まる要素なので、都度確認する必要があります。そうでなければ、コンテナ(Pod)起動後に個別設定が必要となってしまうでしょうから、必須の確認項目となるかと。
Docker Official Imagesでは、以下のような形で環境変数が説明されています。これは提供されているイメージの説明ページによって記載方法が違うため、都度確認が必要となります。

Docker Official Images: MySQLの環境に関する説明
![dockerhub_envval_mysql.png](../../imgs/dockerhub_envval_mysql.png)


Docker Official Images: Mongoの環境変数に関する説明
![dockerhub_envval_mongo.png](../../imgs/dockerhub_envval_mongo.png)

### Pod作成準備
以降の手順については、システムアーキテクチャの差分はありません。MySQL(x86-64)の例のみ記載します。
ちなみに、こちらの例はセキュアではありません。
パスワードは都度個別にsecretで運用すべきですが、説明用としてsecretを使うものと直書きの両方を案内します。
下記のように設定するとMySQL ROOTのパスワードが`rootpass01`となります。適宜変更して使って下さい。

```shell:コマンド
kubectl create secret generic mysql-pass --from-literal=password=rootpass01
```

Pod作成用yamlの用意

```yaml:mysql-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: watashino-mysql
  labels:
    name: label-mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.7   # このように指定するとdocker hubのMySQL 5.7となります
    env:
    - name: MYSQL_USER       # 環境変数を直接指定
      value: user01          # ユーザー名
    - name: MYSQL_PASSWORD   # 環境変数を直接指定
      value: password01      # 上記にて設定するユーザーのパスワード
    - name: MYSQL_DATABASE   # 環境変数を直接指定
      value: database01      # データベース名称
    - name: MYSQL_ROOT_PASSWORD # secretを使った環境変数の指定
      valueFrom:                #
        secretKeyRef:           # secretsを参照
          name: mysql-pass      # secretsの名称
          key: password         # secretsのkey
    ports:
    - name: mysql
      containerPort: 3306    # コンテナのポート番号
      protocol: TCP
    volumeMounts:
    - name: watashino-mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: watashino-mysql-storage
    emptyDir: {}
```

### Podのデプロイと確認
上記で作ったsecretとyamlを使ってPodを作成します。

```shell:コマンド
kubectl apply -f mysql-pod.yaml
```

確認しましょう。

```shell:コマンド
kubectl get pods
NAME                                READY   STATUS      RESTARTS   AGE
watashino-mysql                     1/1     Running     0          3m

```

Podが起動したら中身を確認します。

```shell:コマンド
kubectl exec -it pods/watashino-mysql -- /bin/sh
# mysql -u user01 -h localhost -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 188
Server version: 5.7.34 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| database01         |
+--------------------+
2 rows in set (0.00 sec)

mysql> \q
Bye
#

```


## 備考
Podのアンインストール方法

```shell:コマンド
kubectl delete pods/watashino-mysql
```

Secretのアンインストール方法

```shell:コマンド
kubectl delete secrets/mysql-pass
```
