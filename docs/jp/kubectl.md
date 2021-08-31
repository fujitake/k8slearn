# 任意のkubectlをダウンロードする方法

versionのところは、適宜書き換えて取得すること

MAC版
```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.14.10/bin/darwin/amd64/kubectl
```

Windows版
```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/windows/amd64/kubectl.exe
```

具体的なversion numberについては、以下を参照すればわかります
https://github.com/kubernetes/kubernetes/tags
