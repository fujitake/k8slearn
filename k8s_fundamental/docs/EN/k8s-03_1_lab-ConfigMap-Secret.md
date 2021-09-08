# **Lab 03-1 - Lab exercises on Kubernetes Configuration related resources**

**The working directory is *materials/lab03-1_configmap-secret* directory.**

## ***Step 1 (Provide ConfigMap and Secret as Environment variables)***

1.  Review the contents of the manifest file `sql-env-deployment.yml` of the Pod to be deployed.

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
  clusterIP: None     # Direct access with Pod IP
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
        env:          # Define Environment
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
```

2.  Go to the working directory in Terminal and deploy the Pod with the `kubectl` command.

```sh
$ kubectl apply -f sql-env-deployment.yml
service/mysql created
deployment.apps/mysql created
```

 In this Lab, deploy MySQL Pod. And also deploy the Service in order to access that Pod with mysql from within the cluster.

3.  Confirm the deployment status and the Environment variables of the Containers in the Pod (modify the pod name that follow `kubectl exec -it` to suit your environment).

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

 In this way, by defining environment variables directly in `spec.template.spec.containers.env` in the manifest, it is possible to define any Environment variables for containers.

4.  Delete the deployed resources once.

```sh
$ kubectl delete -f sql-env-deployment.yml
service "mysql" deleted
deployment.apps "mysql" deleted
```

5.  Review the manifest file `ssql-configmap-deployment.yml`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
data:                 # Define the value to be provided in the key-value format
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
  clusterIP: None     # Direct access with Pod IP
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
        envFrom:          # Define Environment variables
        - configMapRef:   # Provide ConfigMap as an Environment variable
            name: mysql-configmap
        ports:
        - containerPort: 3306
          name: mysql
```

6.  Deploy resources.

```sh
$ kubectl apply -f sql-configmap-deployment.yml
configmap/mysql-configmap created
service/mysql created
deployment.apps/mysql created
```

7.  Confirm the deployment status and the Environment variables of the Containers in the Pod.

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

 This way, it is possible to provide configuration information from ConfigMap to Pod as Environment variables.    
 Also, when providing numeric values in ConfigMap, it is needed to enclose them in _double quotation marks_.    
 In Kubernetes, configuration information is used with ConfigMap, and sensitive information such as database passwords are managed with Secret resources.

8.  Delete the deployed resources once.

```sh
$ kubectl delete -f sql-configmap-deployment.yml
configmap "mysql-configmap" deleted
service "mysql" deleted
deployment.apps "mysql" deleted
```

9.  Review the manifest file `sql-secret-deployment.yml`.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque          # type which defines a Secret in a schema-less
data:                 # Define the value to be provided in the key-value format
  MYSQL_ROOT_PASSWORD: cGFzc3dvcmRzZWM=  # Encoded "passwordsec" with base64
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
  clusterIP: None     # Direct access with Pod IP
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
        envFrom:          # Define Environment variables
        - secretRef:      # Provide Secret as an Environment variable
            name: mysql-secret
        ports:
        - containerPort: 3306
          name: mysql
```

10. Deploy resources.

```sh
$ kubectl apply -f sql-secret-deployment.yml
secret/mysql-secret created
service/mysql created
deployment.apps/mysql created
```

11. Confirm the deployment status and the Environment variables of the Containers in the Pod.

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

 In this way, sensitive information is managed with Secret. However, as the manifest shows, the database password is just base64 encoded, so it is not possible to manage this manifest in the repository as it is. So, it is necessary to encrypt the values with **kubesec** or something similar before uploading them to the repository.

12. Delete the deployed resources.

```sh
$ kubectl delete -f sql-secret-deployment.yml
secret "mysql-secret" deleted
service "mysql" deleted
deployment.apps "mysql" deleted
```

 This is the end of this Lab.
