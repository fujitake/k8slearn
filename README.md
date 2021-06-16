## Purpose
This repository is for sharing memo of learning Kubernetes.

### Understanding Kubernetes basics

- [Create Kubernetes Cluster with Raspberry Pi](docs/eng/configure_k3s_w_rasppi.md)
- [Create Kubernetes Cluster with Note PC](docs/jp/configure_k3s_w_notepc.md)
- [Set up Database Pods on Kubernetes](docs/jp/setup_db_pods.md)
- Launch web service on Kubernetes Cluster
- Try to do failure test with simple Pods
- Improve availability of the Pods
- Configure HTTPS
- Try again failure test with high availability configuration

### Debugging Kubernetes

- Failure Pod run

### Misc

- How to prepare SSL Cert
- Create SSL Cert with ECC
- Create container images for x86-64 and ARM64 by Docker


## 目的

このリポジトリは、Kubernetesを学ぶためのメモを共有するものです。

### Kubernetesの基礎を実践的に理解する

- [2時間でRaspberry Piを使ったKubernetes Clusterを作る](docs/jp/configure_k3s_w_rasppi.md)
- [ノートPCを使ってKubernetes Clusterを作ってみた](docs/jp/configure_k3s_w_notepc.md)
- [Kubernetes上でデータベースPodを作成してみる](docs/jp/setup_db_pods.md)
- Kubernetes上でWebサービスを起動する
- シンプルなPodで障害試験を試す
- Kubernetes上のWebサービスを可用性の高い構成に変更する
- Kubernetes上のWebサービスをHTTPSにする
- Kubernetes上で可用性が高い構成を使って、障害試験を試してみる

### Kubernetes デバッグ

- Podが起動しない

### その他

- SSL証明書の作り方
- [楕円曲線SSL証明書の作り方](docs/jp/create_ecdsa_sslcert.md)
- Dockerを使って、x86-64とARM64の両方に対応したコンテナイメージを作成する
