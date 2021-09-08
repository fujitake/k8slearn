# **Lab 02-1 – Lab exercises on Network, Loadbalancing resources in Kubernetes**

## ***Step 0 (Prepare the Kubernetes runtime environment and necessary files)***

1.  Receive the necessary files for Lab from your instructor and extract them to your working directory.  
**The working directory is *materials/lab02-1_svc-ingress* directory.**

## ***Step 1 (Deploy a ReplicateSet)***

1.  Review the contents of the manifest file `simple-replicaset-with-label.yaml` for ReplaceSet to be deployed.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-tokyo
  labels:
    app: nginx
    release: tokyo
spec:
  replicas: 1
  selector:
    matchLabels: # Set the Pod label managed by rs-tokyo ReplicaSet
      app: nginx 
      release: tokyo
  template:
    metadata:
      labels: # Set the Pod label
        app: nginx
        release: tokyo
    spec:
      containers:
      - name: nginx
        image: nginx:1.13 
        env: 
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80 
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-osaka
  labels:
    app: nginx
    release: osaka 
spec:
  replicas: 2
  selector:
    matchLabels: # Set the Pod label managed by rs-osaka ReplicaSet
      app: nginx
      release: osaka 
  template:
    metadata:
      labels: # Set the Pod label
        app: nginx
        release: osaka 
    spec:
      containers:
      - name: nginx
        image: nginx:1.14 
        env: 
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80 
```

2.  Go to the working directory in Terminal and deploy the Pods with the `kubectl` command.

```sh
$ kubectl apply -f simple-replicaset-with-label.yaml
replicaset.apps/rs-tokyo created
replicaset.apps/rs-osaka created
```

 Created nginx containers in the ReplicaSet Pods named "rs-tokyo" and "rs-osaka".

3.  Confirm the deployment status.

```sh
$ kubectl get rs,pod -l app=nginx,release=tokyo
NAME DESIRED CURRENT READY AGE
replicaset.extensions/rs-tokyo 1 1 1 7s

NAME READY STATUS RESTARTS AGE
pod/rs-tokyo-skw9h 1/1 Running 0 7s
$ kubectl get rs,pod -l app=nginx,release=osaka
NAME DESIRED CURRENT READY AGE
replicaset.extensions/rs-osaka 2 2 2 19s

NAME READY STATUS RESTARTS AGE
pod/rs-osaka-pn2s5 1/1 Running 0 19s
pod/rs-osaka-tbh7j 1/1 Running 0 19s
```

 It is possible to confirm that the pods with `release` label with *tokyo* and *osaka* have been deployed respectively.  
 ＊  The `-l` option is the same as the `-selector` option and allows to display resources that match the label name.

4.  Copy the response file to the Pods.  
 **＊ Use the Pod name same as that of the Pod in your environment.**

```
$ kubectl cp tokyo-pod-1/index.html rs-tokyo-skw9h:/usr/share/nginx/html/

$ kubectl cp osaka-pod-1/index.html rs-osaka-pn2s5:/usr/share/nginx/html/

$ kubectl cp osaka-pod-2/index.html rs-osaka-tbh7j:/usr/share/nginx/html/
```

 ＊ `kubectl cp <copy source> <copy destination>`  
 To specify a Pod, use \<Pod name>:\<path in Pod>.

## ***Step 2 (Deploy a ClusterIP Service)***
  Create the Service so that it can access the Pod whose `release` label is *osaka*.  
1.  Review the contents of the manifest file `simple-service.yaml` for the Service to be deployed.

```yaml
apiVersion: v1
kind: Service  # Set the resource type to Service
metadata:
  name: nginx   # Service Name: Used for Name resolution
spec:
  selector:    # Label of the Pod that the Service is targeting
    app: nginx
    release: osaka
  ports:
    - name: http
      port: 80 # Destination port
```

2.  Deploy the Service with the `kubectl` command.

```sh
$ kubectl apply -f simple-service.yaml
service/nginx created
```

3.  Confirm the deployment status.

```sh
$ kubectl get svc nginx
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
nginx ClusterIP 10.101.159.85 <none> 80/TCP 28s

```

  Since the ClusterIP Service was created, Pods in the cluster can access to Pods whose `app` label is *nginx* and whose `release` label is *osaka* via the nginx Service.

4.  Now check the logs of the *echo* container for each pod.    
 **＊ Use the Pod name same as that of the Pod in your environment.**

```sh
$ kubectl logs rs-tokyo-skw9h

$ kubectl logs rs-osaka-pn2s5

$ kubectl logs rs-osaka-tbh7j
```

 Now temporarily deploy a debug container in the cluster to check the traffic, send http requests with the `curl` command several times, and then check the logs again.

5.  Deploy the debug container and send requests from within the container several times.    
 ＊ The requests are not always returned in the order of Pod1 and Pod2.

```sh
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you do not see a command prompt, try pressing enter.
[ root@curl-6bf6db5c4f-qlbsl:/ ]$ curl http://nginx
Hello! Nginx1.14 Osaka Pod 1!
[ root@curl-6bf6db5c4f-qlbsl:/ ]$ curl http://nginx
Hello! Nginx1.14 Osaka Pod 2!
```

 The message "Hello Nginx1.14 Osaka Pod \<Num>!!" message is returned for the request. By creating the service, it is now possible to resolve names using DNS in the cluster. Intra-cluster DNS allows name resolution with    
 **ServiceName.NamespaceName.svc.cluster.local**  
 Also,`svc.cluster.local` can be omitted. To resolve the name of a Service in a different Namespace, with      
 ServiceName.NamespaceName.  
 If the namespace is the same, it can be resolved using only the ServiceName. Therefore, name resolution can be done from within the debug container by making a request with the `curl` command as shown below.    
 ＊ If deploying with a non-default Namespace, the Namespace name should be replaced.

```sh
[ root@curl-6bf6db5c4f-qlbsl:/ ]$ curl http://nginx.default.svc.cluster.local/
Hello! Nginx1.14 Osaka Pod 1!
[ root@curl-6bf6db5c4f-qlbsl:/ ]$ curl http://nginx.default/
Hello! Nginx1.14 Osaka Pod 1!
```

 Log out of the container with the `exit` command or `ctrl+d`. The following command allows log in to the container again.     
 `kubectl exec -it \<pod名> bash`

6.  Check the Pod logs again.  
 **＊ Use the Pod name same as that of the Pod in your environment.**

```sh
$ kubectl logs rs-tokyo-skw9h

$ kubectl logs rs-osaka-pn2s5
10.1.2.59 - - [06/Jul/2020:05:37:07 +0000] "GET /index.html HTTP/1.1" 200 31 "-" "curl/7.35.0" "-"
10.1.2.59 - - [06/Jul/2020:05:37:10 +0000] "GET /index.html HTTP/1.1" 200 31 "-" "curl/7.35.0" "-"
10.1.2.59 - - [06/Jul/2020:05:37:13 +0000] "GET /index.html HTTP/1.1" 200 31 "-" "curl/7.35.0" "-"

$ kubectl logs rs-osaka-tbh7j
10.1.2.59 - - [06/Jul/2020:05:37:18 +0000] "GET /index.html HTTP/1.1" 200 31 "-" "curl/7.35.0" "-"
10.1.2.59 - - [06/Jul/2020:05:37:23 +0000] "GET /index.html HTTP/1.1" 200 31 "-" "curl/7.35.0" "-"
```

The logs show that only Pods with the label *osaka* are accessible via the Service. The relationship between the Service and the Pods identified with labels is shown in the figure below.

![image2](../../imgs/lab2_1/media/image2.png)

## ***Step 3（Deploy a NodePort Service）***
 Allow access to pods with *osaka* as a label from outside the cluster.  
1.  Review the contents of the manifest file `ssimple-service-nodeport.yaml` for the NodePort to be deployed.

```yaml
apiVersion: v1
kind: Service    # Specify the Resource type as Service
metadata:
 name: nginx
spec:
  type: NodePort # Service type: If default (unspecified), ClusterIP will be used
  selector:      # Label of the Pod that the Service is targeting
    app: nginx
    release: osaka
  ports:
    - name: http
      port: 80   # Destination port
```

2.  Deploy the Nodeport with the `kubectl` command.     
    (Either delete the manifest file base or delete the pod by specifying it.)

```sh
$ kubectl apply -f simple-service-nodeport.yaml
service/nginx configured
```

3.  Confirm the deployment status.

```sh
$ kubectl get svc nginx
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
nginx NodePort 10.101.159.85 <none> 80:30713/TCP 175m
```

 **PORT(S) notation**:  
 {Port of the Pod to which the Service is connected}:{Node's port to connect to the service}/protocol  

 This way, in the above case, the Pod can be accessed using http://localhost:30713.

4.  Access the pod from outside the cluster.    
 Access to the following from a local machine with a browser or the `curl` command, and confirm that the response is returned.    
 http://localhost:{Node's port to connect to the service}

5.  Delete the NodePort with the `kubectl delete` command.    
**＊ Make sure to delete it as it will affect Step 4.**

```sh
$ kubectl delete -f simple-service-nodeport.yaml
service "nginx" deleted
```

## ***Step 4（Deplyo an Ingress）***  
 Use Ingress in a local environment to control at the L7 layer level.  
1.  Deploy **nginx_ingress_controller**

```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
namespace/ingress-nginx configured
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission createdclusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
```

If Pod _ingress-nginx-controller_ and Service _ingress-nginx_ are deployed on the Namespace _ingress-nginx_ as following, it is ready to use the Ingress resource.

```sh
$ kubectl get pods,svc -n ingress-nginx
NAME READY STATUS RESTARTS AGE
pod/ingress-nginx-admission-create-52vzf 0/1 Completed 0 5m59s
pod/ingress-nginx-admission-patch-bcrxn 0/1 Completed 0 5m59s
pod/ingress-nginx-controller-86cbd65cf7-gtpvs 1/1 Running 0 6m9s

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
service/ingress-nginx-controller LoadBalancer 10.96.193.5 localhost 80:30668/TCP,443:30025/TCP 6m9s
service/ingress-nginx-controller-admission ClusterIP 10.110.112.252 <none> 443/TCP 6m9s
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
service/default-http-backend ClusterIP 10.110.141.207 <none> 80/TCP 15m
service/ingress-nginx LoadBalancer 10.111.53.58 localhost 80:32640/TCP,443:31347/TCP 15m
```

 At this stage, a virtual IP address has been dispatched to localhost as localhost, so it is possible to confirm that *404* will be returned when accessing http://localhost from a local machine.

2.  Confirm the contents of the manifest file `simple-ingress.yaml`, which defines Ingress, and the manifest file `simple-service-forIngress.yaml`, which defines the service to be used as a routing destination in Ingress.  

`simple-ingress.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress   # Specify the resource type as Ingress
metadata:
  name: nginx-ing
spec:
  rules:        # Routing Rule
  - host: ingress.test.local
    http:
      paths:
      - path: /
        backend: # Forwarding destination
          serviceName: nginx-svc
          servicePort: 80
```

`simple-service-forIngress.yaml`

```yaml
apiVersion: v1
kind: Service  # Specify the resource type as Service
metadata:
  name: nginx-svc   # Service Name: Used for Name Resolution
spec:
  type: ClusterIP
  selector:    # Label of the Pod that the Service is targeting
    app: nginx
  ports:
    - name: http
      port: 80 # Destination port
```

3.  Deploy both Service and Ingress with the `kubectl` command.

```sh
$ kubectl apply -f simple-service-forIngress.yaml
service/nginx-svc created
$ kubectl apply -f simple-ingress.yaml
ingress.extensions/nginx-ing created
```

4.  Confirm the deployment status.

```sh
$ kubectl get ingress
NAME HOSTS ADDRESS PORTS AGE
ingress.extensions/nginx-ing ingress.test.local localhost 80 8s

```

5.  Send the request from a local machine and confirm that the response is returned.

```sh
$ curl -sS http://localhost/ -H "Host: ingress.test.local"
Hello! Nginx1.14 Osaka Pod 2!

$ curl -sS http://localhost/
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.19.0</center>
</body>
</html>
```

It is confirmed that L7 routing based on the Host header is occurring by Ingress.  
The traffic image looks like the following.

![image3](../../imgs/lab2_1/media/image3.png)

Then, do path-based routing with Ingress.

6.  Delete Service and Ingress that were deployed in 3.

```sh
$ kubectl delete -f simple-ingress.yaml
ingress.extensions "nginx-ing" deleted

$ kubectl delete -f simple-service-forIngress.yaml
service "nginx-svc" deleted
```

7.  Confirm the manifest files for the newly deploy Service and Ingress for path-based routing.

`simple-service-forIngress-path-routing.yaml`

```yaml
apiVersion: v1
kind: Service  # Specify the resource type as Service
metadata:
  name: nginx-svc-tokyo   # Service Name: Used for Name Resolution
spec:
  type: ClusterIP
  selector:    # Label of the Pod that the Service is targeting
    app: nginx
    release: tokyo
  ports:
    - name: http
      port: 80 # Destination port
---
apiVersion: v1
kind: Service  # Specify the resource type as Service
metadata:
  name: nginx-svc-osaka   # Service Name: Used for Name Resolution
spec:
  type: ClusterIP
  selector:    # Label of the Pod that the Service is targeting
    app: nginx
    release: osaka
  ports:
    - name: http
      port: 80 # Destination port
```

`simple-ingress-path-routing.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress   # Specify the resource type as Ingress
metadata:
  name: nginx-ing
spec:
  backend:      # Configuration for Default Backend
    serviceName: nginx-svc-default
    servicePort: 80
  rules:        # Routing Rule
  - host: ingress.test.local
    http:
      paths:
      - path: /tokyo
        backend: # Forwarding destination 
          serviceName: nginx-svc-tokyo
          servicePort: 80
      - path: /osaka
        backend: # Forwarding destination 
          serviceName: nginx-svc-osaka
          servicePort: 80
```

`backend-pod.yaml`

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-default
  labels:
    app: nginx
    release: default
spec:
  replicas: 1
  selector:
    matchLabels: # Set the Pod label managed by rs-default ReplicaSet
      app: nginx 
      release: default
  template:
    metadata:
      labels: # Set the Pod label
        app: nginx
        release: default
    spec:
      containers:
      - name: nginx
        image: nginx:1.15 
        env: 
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80 
---
apiVersion: v1
kind: Service  # Specify the resource type as Service
metadata:
  name: nginx-svc-default   # Service Name: Used for Name Resolution
spec:
  type: ClusterIP
  selector:    # Label of the Pod that the Service is targeting
    app: nginx
    release: default
  ports:
    - name: http
      port: 80 # Destination port
```

Same as before, defined the manifest file to do path-based routing of URLs in addition to L7 routing based on the Host header. Since a default backend can be configured, configure `backend-pod.yaml` to have the nginx 1.15 Pod return *404*.    
In this Lab, the routing will be the following.

**・http://localhost/tokyo/**

To nginx-svc-tokyo port 80

**・http://localhost/osaka/**

To nginx-svc-osaka port 80

**・Other than above**

To nginx-svc-default port 80  

Each Service in the routing destination will be forwarded to the Pod with `app=nginx` label and *tokyo* or *osaka* in each `release` label.

8.  Deploy Servise, Ingress, Pod for Backend with the `kubectl` command.

```sh
$ kubectl apply -f simple-service-forIngress-path-routing.yaml
service/nginx-svc-tokyo created
service/nginx-svc-osaka created

$ kubectl apply -f simple-ingress-path-routing.yaml
ingress.extensions/nginx-ing created

$ kubectl get ingress
NAME HOSTS ADDRESS PORTS AGE
nginx-ing ingress.test.local localhost 80 18m

$ kubectl apply -f backend-pod.yaml
replicaset.apps/rs-default created
service/nginx-svc-default created
```

9.  Modify the *nginx* Pod to return a response.    
**＊ Use the Pod name same as that of the Pod in your environment.**

```sh
// Create a corresponding directory for each Pod and copy the index.html file to that directory.
$ kubectl -it exec rs-tokyo-skw9h -- mkdir /usr/share/nginx/html/tokyo
$ kubectl -it exec rs-tokyo-skw9h -- cp /usr/share/nginx/html/index.html /usr/share/nginx/html/tokyo

$ kubectl -it exec rs-osaka-pn2s5 -- mkdir /usr/share/nginx/html/osaka
$ kubectl -it exec rs-osaka-pn2s5 -- cp /usr/share/nginx/html/index.html /usr/share/nginx/html/osaka/

$ kubectl -it exec rs-osaka-tbh7j -- mkdir /usr/share/nginx/html/osaka
$ kubectl -it exec rs-osaka-tbh7j -- cp /usr/share/nginx/html/index.html /usr/share/nginx/html/osaka/
```

10. Send the request from a local machine and confirm that the response is returned.

```sh
$ curl http://localhost/tokyo/ -H "Host: ingress.test.local" -sS
Hello! Nginx1.13 Tokyo Pod 1!

$ curl http://localhost/osaka/ -H "Host: ingress.test.local" -sS
Hello! Nginx1.14 Osaka Pod 1!

$ curl http://localhost/osaka/ -H "Host: ingress.test.local" -sS
Hello! Nginx1.14 Osaka Pod 2!

$ curl http://localhost/winter -H "Host: ingress.test.local" -sS
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.15.12</center>
</body>
</html>
```

It is confirmed that Ingress is executing L7 routing based on the Host header and path. The image of the traffic is the following.

![image4](../../imgs/lab2_1/media/image4.png)

That should complete the Ingress confirmation.

## ***Step 5（Reset a local cluster environment）***

1.  Open docker's Settings > Kubernetes menu and reset it with *"Reset Kubernetes Cluster"*.

![image5](../../imgs/lab2_1/media/image5.png)
