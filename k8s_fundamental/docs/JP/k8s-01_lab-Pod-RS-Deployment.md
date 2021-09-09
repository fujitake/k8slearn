# **Lab 01 – Kubernetes の Workload リソースについての実習**

fundamentalコースのLabは Docker for Desktop(Mac or Windows) を想定しています。

## ***Step 1 (Pod のデプロイ)***

**作業ディレクトリは *materials/ lab01_pod-rs-dep* ディレクトリです。**

1．デプロイするPodのマニフェストファイル `simple-pod.yaml` の内容を確認します。

```yaml
apiVersion: v1
kind: Pod           # リソースの種類を指定する。kindの値でspec配下スキーマが変化
metadata:
  name: simple-pod  # リソースの名称として利用される
  labels:           # Podのlabelを定義
    app: simple
spec:               # kindがPodのため、Podを構成するコンテナ群をcontainersに定義
  containers:
  - name: nginx     # コンテナの名前
    image: nginx:1.13   # コンテナイメージの指定
    env:            # コンテナで利用される環境変数
    - name: BACKEND_HOST
      value: localhost:8080
    ports:          # EXPOSEするポートの指定
    - containerPort: 80 
  - name: alpine
    image: alpine
    command: ["/bin/sh"]
    args: ["-c", "while true; do date; sleep 10;done"]

```

2．ターミナルで作業ディレクトリに移動し、`kubectl`コマンドでPodをデプロイします。

```sh
$ kubectl apply -f simple-pod.yaml
pod/simple-pod created
```

今回は simple-pod というPodの中に _nginx_、_alpine_ というコンテナを作成しています。

3．デプロイ状況を確認します。

```sh
$ kubectl get pod
NAME READY STATUS RESTARTS AGE
simple-pod 2/2 Running 0 36m
```

 STATUSが「Running」となっていれば、Podのデプロイが完了しています。  
 NAME: Pod名  
 READY: Pod内の起動コンテナ数  
 STATUS: Podの状態  
 AGE: Pod起動経過時間  
 ＊ `kubectl get` コマンドでリソースの基本的な情報を取得できます。

## ***Step 2 (Pod の操作)***

作成したPod内のコンテナには`kubectl`コマンドを使ってアクセスすることができます。

1.  `kubectl exec`コマンドを使ってコンテナへアクセスし、マニフェストで定義した環境変数を確認します。(`-c` オプション: Pod内に複数コンテナがある場合はコンテナ名を指定します。)  
 コンテナからログオフするには`exit` もしくは「`Ctrl + d`」でログオフできます。

```sh
$ kubectl exec -it simple-pod -c nginx -- bash
root@simple-pod:/# echo $BACKEND_HOST
localhost:8080
root@simple-pod:/# exit
```

Pod内コンテナの標準出力を`kubectl`コマンドを使って表示することができます。

2.  `kubectl logs`コマンドからコンテナのログを表示してみましょう。こちらも `-c` オプションでコンテナ名を指定できます。`-f` オプションで`tail -f` コマンド相当の出力ができます。

```sh
$ kubectl logs -f simple-pod -c alpine
Thu Jul 2 21:04:09 UTC 2020
Thu Jul 2 21:04:19 UTC 2020
```

## ***Step 3（Pod の削除）***

作成したPodを削除します。作成したリソースの削除には`kubectl delete` コマンドを使用します。  
また、`kubectl delete`コマンドではマニフェストファイルベースでのリソース削除も可能です。  
マニフェストファイルベースでのリソース削除の場合、マニフェストに記述されているリソースがすべて削除されます。

1.  `kubectl delete`コマンドからPodを削除します。  
    (マニフェストファイルベースの削除もしくはPodを指定した場合の削除どちらかを行います。)

```sh
＃ マニフェストファイルベースの削除の場合
$ kubectl delete -f simple-pod.yaml
pod "simple-pod" deleted

＃ Podを指定した削除の場合
$ kubectl delete pod simple-pod
pod "simple-pod" deleted
```

2.  Podが削除されたことを確認します。

```sh
$ kubectl get pod
No resources found.
```

## ***Step 4（ReplicaSet のデプロイ）***

1.  デプロイするPodのマニフェストファイル `simple-replicaset.yaml` の内容を確認します。

```yaml
apiVersion: apps/v1
kind: ReplicaSet  # ReplicaSetのマニフェスト
metadata:         # ReplicaSetのメタデータ
  name: simple-rs 
  labels:
    app: simple 
spec:
  replicas: 2     # Podの複製数 
  selector:
    matchLabels:  # ReplicaSetが管理(検索)するPodのラベル
      app: simple 
  template:       # template以下はPodリソースにおけるspec定義と同じ
    metadata:
      labels:     # Podのlabelを定義
        app: simple
    spec:         #コンテナの定義(simple-pod.yamlの定義と同じ)
      containers:
      - name: nginx 
        image: nginx:1.13 
        env: 
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80 
      - name: alpine
        image: alpine
        command: ["/bin/sh"]
        args: ["-c", "while true; do date; sleep 10;done"]

```

2.  ターミナルで作業ディレクトリに移動し、`kubectl`コマンドで ReplicaSet をデプロイします。

```sh
$ kubectl apply -f simple-replicaset.yaml
replicaset.apps/simple-rs created
```

デプロイするPodは Step1 でデプロイしたPodと同一の定義です。Podを `replicas` で指定された数と同じ数のPodがデプロイされます。

3.  デプロイ状況を確認します。

```sh
$ kubectl get rs,pod -L app
NAME DESIRED CURRENT READY AGE APP
replicaset.extensions/simple-rs 2 2 2 67s simple

NAME READY STATUS RESTARTS AGE APP
pod/simple-rs-6b987 2/2 Running 0 67s simple
pod/simple-rs-n7dhw 2/2 Running 0 67s simple

```

Podのデプロイが完了していることを確認します。Pod名には「-」以降のランダムな識別子がサフィックスに付与されます。ReplicaSet のステータスは下記のとおりです。  
DESIRED: マニフェストファイルで指定されているPod数  
CURRENT: 現在の起動しているPod数  
＊ `kubectl get`コマンドではリソースを複数指定して情報を取得できます。`-L` オプションで各Podの指定したlabelを表示できます。  
デプロイされたPodには Step2 と同様に操作を行うことができます。

## ***Step5（ReplicaSet の動作確認）***

1.  ReplicaSet 外で同じラベルを持つPodを立ててみます。  
Step1で利用した `simple-pod.yaml` から同じラベルをもつPodをデプロイし、デプロイ状況を確認すると、次のようにPodが停止されることが確認できます。

```sh
$ kubectl apply -f simple-pod.yaml
pod/simple-echo created
$ kubectl get rs,pod -L app
NAME DESIRED CURRENT READY AGE APP
replicaset.extensions/simple-rs 2 2 2 2m58s simple

NAME READY STATUS RESTARTS AGE APP
pod/simple-pod 0/2 Terminating 0 3s simple
pod/simple-rs-6b987 2/2 Running 0 2m58s simple
pod/simple-rs-n7dhw 2/2 Running 0 2m58s simple
```

デプロイした ReplicaSet は2つのPodを管理・制御するため、Podを増やしすぎたと誤認し、3つのPodのうち1つの停止を試みます。今回は最後にデプロイしたPodが停止されていますが、場合によっては既存のPodが削除されてしまうので注意が必要です。

2.  Podのラベルを取り除いてみます。  
**＊ Pod名は自身の環境のPod名を指定して下さい。**

```sh
$ kubectl label pod simple-rs-6b987 app-
pod/simple-rs-6b987 labeled
$ kubectl get rs,pod -L app
NAME DESIRED CURRENT READY AGE APP
replicaset.extensions/simple-rs 2 2 1 4m31s simple

NAME READY STATUS RESTARTS AGE APP
pod/simple-rs-6b987 2/2 Running 0 4m31s
pod/simple-rs-cpzdc 0/2 ContainerCreating 0 4s simple
pod/simple-rs-n7dhw 2/2 Running 0 4m31s simple
```

`simple-rs-6b987` の `app` ラベルを取り除くと、`app=simple` のラベルをもつPodは1つになり、ReplicaSet が自動でPodを1つ増やします。  
このように、ReplicaSet は `replicas` で指定された数になるようにPodを管理・制御します。

## ***Step 6（ReplicaSet の削除）***

1.  `kubect delete`コマンドを使ってデプロイした ReplicaSet を削除します。

```sh
$ kubectl delete -f simple-replicaset.yaml
replicaset.apps "simple-rs" deleted
```

2.  Step5でラベルを取り除いたPodを削除します。  
**＊ Pod名は自身の環境のPod名を指定して下さい。**

```sh
$ kubectl delete pod simple-rs-6b987
pod "simple-rs-6b987" deletede
```

3.  リソースが削除されたことを確認します。

```sh
$ kubectl get rs,pod
No resources found.
```

## ***Step 7（Deployment のデプロイ）***

1.  デプロイする Deployment のマニフェストファイルを確認します。

```yaml
apiVersion: apps/v1
kind: Deployment # リソースの種類をDeploymentに
metadata:        # Deploymentのメタデータ
  name: simple-ds
  labels:
    app: simple 
spec:            # ReplicaSetの定義と同じ
  replicas: 2
  selector:
    matchLabels: # ReplicaSetが管理(検索)するPodのラベル
      app: simple 
  template:      # template以下はPodリソースにおけるspec定義と同じ
    metadata:
      labels:
        app: simple
    spec:
      containers:
      - name: nginx 
        image: nginx:1.13 
        env: 
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80 
      - name: echo
        image: alpine
        command: ["/bin/sh"]
        args: ["-c", "while true; do date; sleep 10;done"] 
```

2.  ターミナルで作業ディレクトリに移動し、`kubectl`コマンドで ReplicaSet をデプロイします。

```sh
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/simple-ds created
```

`--record` オプションをつけることで実行した`kubectl`のコマンドを記録できます。  
デプロイされるリソースは Step4(ReplicaSetのデプロイ)でデプロイしたものと同様です。

3.  デプロイ状況を確認します。

```sh
$ kubectl get deployment,rs,pod --selector app=simple
NAME READY UP-TO-DATE AVAILABLE AGE
deployment.extensions/simple-ds 2/2 2 2 12s

NAME DESIRED CURRENT READY AGE
replicaset.extensions/simple-ds-7bcfbdfd5f 2 2 2 12s

NAME READY STATUS RESTARTS AGE
pod/simple-ds-7bcfbdfd5f-kmvs8 2/2 Running 0 12s
pod/simple-ds-7bcfbdfd5f-wff9p 2/2 Running 0 12s
```

`--selector` オプションでラベルを指定して情報を取得します。今回はマニフェストファイルで指定してある、`app`というキーが simple という値が指定されているラベルを指定して情報を取得します。  
Deployment・ReplicaSet・Pod が作成されていることが確認できます。ReplicaSet 名にも「-」以降のランダムな識別子がサフィックスに付与されます。  
また、`kubectl rollout history`コマンドで Deployment のリビジョンが作成されたことを確認します。

```sh
$ kubectl rollout history deployment simple-ds
deployment.extensions/simple-ds
REVISION CHANGE-CAUSE
1 kubectl apply --filename=simple-deployment.yaml --record=true

```

初回反映のため、REVISION が 1となっています。

## ***Step 8（Deployment 更新による ReplicaSet 入れ替え）***

Deployment マニフェストファイルを更新し、デプロイした際に以下の動きを確認します。
* Pod数のみ更新しても新規 ReplicaSet が生成されない
* コンテナ定義を更新することで ReplicaSet の入れ替えが発生

1.  マニフェストファイルのPodレプリカ数を変更します。  
`simple-deployment.yaml` ファイルを下記のように変更し保存します。  
* `spec.replicas` の値を 2から3に変更  
ファイル保存後、`kubectl`コマンドを使ってデプロイします。

```sh
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/simple-ds configured
```

2.  デプロイされたリソースと Deployment のリビジョンを確認します。

```sh
$ kubectl get deployment,rs,pod --selector app=simple
NAME READY UP-TO-DATE AVAILABLE AGE
deployment.extensions/simple-ds 3/3 3 3 3m39s

NAME DESIRED CURRENT READY AGE
replicaset.extensions/simple-ds-7bcfbdfd5f 3 3 3 3m39s

NAME READY STATUS RESTARTS AGE
pod/simple-ds-7bcfbdfd5f-cm9pl 2/2 Running 0 32s
pod/simple-ds-7bcfbdfd5f-kmvs8 2/2 Running 0 3m39s
pod/simple-ds-7bcfbdfd5f-wff9p 2/2 Running 0 3m39s

$ kubectl rollout history deployment simple-ds
deployment.extensions/simple-ds
REVISION CHANGE-CAUSE
1 kubectl apply --filename=simple-deployment.yaml --record=true
```

Podが新たに1つ作成されたことが確認できます。また、サフィックスから ReplicaSet および既存の2つのPodの入れ替えが発生していないことが確認できます。  
また、新たに ReplicaSet が生成されていれば REVISION も 2になりますが、表示されていません。  
これらのことから `replicas` の変更では ReplicaSet の入れ替えが発生しないことを確認できます。

3.  マニフェストファイルのコンテナ定義の変更を行います。  
echo コンテナのイメージを変更します。`simple-deployment.yaml` ファイルを下記のように変更し保存します。
* nginx コンテナの `image` の値を nginx:1.13 から nginx:1.14に変更  
ファイル保存後、`kubectl`コマンドを使ってデプロイします。

```sh
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/simple-ds configured
```

リソースのデプロイ状況を確認すると、Podと ReplicaSet の入れ替えが行われていることを確認することができます。

```sh
$ kubectl get deployment,rs,pod --selector app=simple
NAME READY UP-TO-DATE AVAILABLE AGE
deployment.extensions/simple-ds 3/3 2 3 6m37s

NAME DESIRED CURRENT READY AGE
replicaset.extensions/simple-ds-547f78b574 2 2 1 20s
replicaset.extensions/simple-ds-7bcfbdfd5f 2 2 2 6m37s

NAME READY STATUS RESTARTS AGE
pod/simple-ds-547f78b574-cdcgj 2/2 Running 0 19s
pod/simple-ds-547f78b574-q4j2j 0/2 ContainerCreating 0 3s
pod/simple-ds-7bcfbdfd5f-cm9pl 2/2 Terminating 0 3m30s
pod/simple-ds-7bcfbdfd5f-kmvs8 2/2 Running 0 6m37s
pod/simple-ds-7bcfbdfd5f-wff9p 2/2 Running 0 6m37s
```

4.  デプロイされたリソースの状況を確認します。  
ReplicaSet の入れ替えが完了すると、以下のように確認することができます。

```sh
$ kubectl get deployment,rs,pod --selector app=simple
NAME READY UP-TO-DATE AVAILABLE AGE
deployment.extensions/simple-ds 3/3 3 3 7m38s

NAME DESIRED CURRENT READY AGE
replicaset.extensions/simple-ds-547f78b574 3 3 3 81s
replicaset.extensions/simple-ds-7bcfbdfd5f 0 0 0 7m38s

NAME READY STATUS RESTARTS AGE
pod/simple-ds-547f78b574-bglfc 2/2 Running 0 59s
pod/simple-ds-547f78b574-cdcgj 2/2 Running 0 80s
pod/simple-ds-547f78b574-q4j2j 2/2 Running 0 64s

$ kubectl rollout history deployment simple-ds
deployment.extensions/simple-ds
REVISION CHANGE-CAUSE
1 kubectl apply --filename=simple-deployment.yaml --record=true
2 kubectl apply --filename=simple-deployment.yaml --record=true
```

Podや ReplicaSet のサフィックスからリソースの入れ替えが発生したことが確認できます。また、新たにリビジョンが作成されたことが`kubctl rollout hisroty`コマンドから確認できます。コンテナのイメージ変更により REVISION の値が 2のリビジョンが作成されました。

## ***Step 9（ロールバックの実行）***

Deployment のリビジョンが記録されているため、特定のリビジョンの内容を確認できます。

1.  `kubectl rollout hisroty` コマンドからリビジョンの内容の確認をします。

```sh
$ kubectl rollout history deployment simple-ds --revision=1
deployment.extensions/simple-ds with revision #1
Pod Template:
  Labels: app=simple
        pod-template-hash=7bcfbdfd5f
  Annotations: kubernetes.io/change-cause: kubectl apply --filename=simple-deployment.yaml --record=true
  Containers:
   nginx:
    Image: nginx:1.13
    Port: 80/TCP
    Host Port: 0/TCP
    Environment:
      BACKEND_HOST: localhost:8080
    Mounts:     <none>
  echo:
    Image: alpine
    Port:  <none>
    Host Port: <none>
    Command:
      /bin/sh
    Args:
      -c
      while true; do date; sleep 10;done
    Environment:
    Mounts: <none>
  Volumes:  <none>
```

`--revision` オプションで特定のリビジョンを指定して内容を確認します。

2.  `kubectl rollout undo` コマンドを使ってロールバックの実行をします。

```sh
$ kubectl rollout undo deployment simple-ds --to-revision 1
deployment.extensions/simple-ds rolled backe
```

`--to-revision` オプションで特定のリビジョンを指定してロールアウトを実行できます。(指定しない場合は直前のリビジョンにロールアウト)

3.  リソースの状況を確認します。

```sh
$ kubectl get deployment,rs,pod --selector app=simple
NAME READY UP-TO-DATE AVAILABLE AGE
deployment.extensions/simple-ds 3/3 3 3 10m

NAME DESIRED CURRENT READY AGE
replicaset.extensions/simple-ds-547f78b574 1 1 1 3m59s
replicaset.extensions/simple-ds-7bcfbdfd5f 3 3 2 10m

NAME READY STATUS RESTARTS AGE
pod/simple-ds-547f78b574-bglfc 2/2 Terminating 0 3m37s
pod/simple-ds-547f78b574-cdcgj 2/2 Running 0 3m58s
pod/simple-ds-547f78b574-q4j2j 2/2 Terminating 0 3m42s
pod/simple-ds-7bcfbdfd5f-rsj7j 2/2 Running 0 10s
pod/simple-ds-7bcfbdfd5f-tf4tv 0/2 ContainerCreating 0 5s
pod/simple-ds-7bcfbdfd5f-xfxcn 2/2 Running 0 16s
```

以前の ReplicaSet が使用されていることを確認できます。

## ***Step 10（Deployment の削除）***

1.  `kubect delete` コマンドを使ってデプロイしたリソースを削除します。

```sh
$ kubectl delete -f simple-deployment.yaml
deployment.apps "simple-ds" deleted
```

2.  リソースが削除されたことを確認します。

```sh
$ kubectl get deployment,rs,pod --selector app=simple
No resources found.
```
