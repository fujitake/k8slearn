# **Lab 03-1 - Kubernetes の Configuration 関連のリソースについての実習**

**作業ディレクトリは *materials/lab03-1_configmap-secret* ディレクトリです。**

## ***Step 1 (環境変数として ConfigMap、Secret を渡す)***

1.  デプロイするPodのマニフェストファイル `sql-env-deployment.yml` の内容を確認します。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None     # PodのIPで直接アクセス
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:          # 環境変数を定義
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
```

2.  ターミナルで作業ディレクトリに移動し、`kubectl` コマンドでPodをデプロイします。

```sh
$ kubectl apply -f sql-env-deployment.yml
service/mysql created
deployment.apps/mysql created
```

 今回は mysql Pod をデプロイしています。また、そのPodにクラスタ内から mysql でアクセスするように Service もデプロイしています。

3.  デプロイ状況とPod内コンテナの環境変数を確認します。(`kubectl exec -it` に続くpod名は作業に合わせて変更してください)

```sh
$ kubectl get pod,svc
NAME READY STATUS RESTARTS AGE
pod/mysql-d8b849957-55q4d 1/1 Running 0 19m

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
service/kubernetes ClusterIP 100.64.0.1 <none> 443/TCP 7h12m
service/mysql ClusterIP None <none> 3306/TCP 38m

$ kubectl exec -it mysql-d8b849957-55q4d bash
root@mysql-d8b849957-55q4d:/# env | grep MYSQL_ROOT_PASSWORD
MYSQL_ROOT_PASSWORD=password
root@mysql-d8b849957-55q4d:/# exit
exit
```

 このように、マニフェスト内の`spec.template.spec.containers.env` に環境変数を直接定義することで、コンテナに任意の環境変数を定義できます。

4.  一度デプロイしたリソースを削除します。

```sh
$ kubectl delete -f sql-env-deployment.yml
service "mysql" deleted
deployment.apps "mysql" deleted
```

5.  デプロイするマニフェスト `sql-configmap-deployment.yml` を確認します。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
data:                 # 渡したい値を Key: Value の形式で定義
  MYSQL_ROOT_PASSWORD: passwordconf
  test_num_val: "100"
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None     # PodのIPで直接アクセス
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        envFrom:          # 環境変数を定義
        - configMapRef:   # ConfigMapを環境変数として渡す
            name: mysql-configmap
        ports:
        - containerPort: 3306
          name: mysql
```

6.  リソースをデプロイします。

```sh
$ kubectl apply -f sql-configmap-deployment.yml
configmap/mysql-configmap created
service/mysql created
deployment.apps/mysql created
```

7.  デプロイ状況とPod内コンテナの環境変数を確認します。

```sh
$ kubectl get pod,svc,configmap
NAME READY STATUS RESTARTS AGE
pod/mysql-86fcfcf8c-zp4bz 1/1 Running 0 3s

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
service/kubernetes ClusterIP 100.64.0.1 <none> 443/TCP 8h
service/mysql ClusterIP None <none> 3306/TCP 3s

NAME DATA AGE
configmap/mysql-configmap 2 3s

$ kubectl exec -it mysql-86fcfcf8c-zp4bz bash
root@mysql-86fcfcf8c-zp4bz:/# env | grep -e MYSQL_ROOT_PASSWORD -e test_num_val
MYSQL_ROOT_PASSWORD=passwordconf
test_num_val=100
root@mysql-86fcfcf8c-zp4bz:/# exit
$
```

 このように ConfigMap からPodへ環境変数として設定情報を渡すことができます。  
 また、ConfigMap で数値を渡す際には、_ダブルクォーテーション_ でくくる必要があります。  
 Kubernetes では設定情報などは ConfigMap を使って利用し、今回のようなデータベースのパスワードといった機密情報は Secretリソース を利用して管理します。

8.  一度デプロイしたリソースを削除します。

```sh
$ kubectl delete -f sql-configmap-deployment.yml
configmap "mysql-configmap" deleted
service "mysql" deleted
deployment.apps "mysql" deleted
```

9.  デプロイするマニフェスト `sql-secret-deployment.yml` を確認します。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque          # スキーマレスでsecret を定義するタイプ
data:                 # 渡したい値を Key: Value の形式で定義
  MYSQL_ROOT_PASSWORD: cGFzc3dvcmRzZWM=  # 「passwordsec」のbase64エンコード
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None     # PodのIPで直接アクセス
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        envFrom:          # 環境変数を定義
        - secretRef:      # Secretを環境変数として渡す
            name: mysql-secret
        ports:
        - containerPort: 3306
          name: mysql
```

10. リソースをデプロイします。

```sh
$ kubectl apply -f sql-secret-deployment.yml
secret/mysql-secret created
service/mysql created
deployment.apps/mysql created
```

11. デプロイ状況とPod内コンテナの環境変数を確認します。

```sh
$ kubectl get pod,svc,secret
NAME READY STATUS RESTARTS AGE
pod/mysql-56466cd95-25jr2 1/1 Running 0 7s

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
service/kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 17m
service/mysql ClusterIP None <none> 3306/TCP 7s

NAME TYPE DATA AGE
secret/default-token-9lcdv kubernetes.io/service-account-token 3 17m
secret/mysql-secret Opaque 1 7s

$ kubectl exec -it mysql-56466cd95-25jr2 bash
root@mysql-56466cd95-25jr2:/# env | grep MYSQL_ROOT_PASSWORD
MYSQL_ROOT_PASSWORD=passwordsec
root@mysql-56466cd95-25jr2:/# exit
exit
$
```

 このように機密情報は Secret を利用して管理します。しかし、マニフェストを確認してわかるように、データベースのパスワードは base64 でエンコードされているのみであるので、このままレポジトリでこのマニフェストを管理するわけにはいきません。そこで、**kubesec** などを使い、値を暗号化してからレポジトリに挙げるなどの対応が必要となってきます。

12. デプロイしたリソースを削除します。

```sh
$ kubectl delete -f sql-secret-deployment.yml
secret "mysql-secret" deleted
service "mysql" deleted
deployment.apps "mysql" deleted
```

 以上で本Labが完了となります。
