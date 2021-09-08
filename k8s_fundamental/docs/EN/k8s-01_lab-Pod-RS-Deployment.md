# **Lab 01 – Lab exercises on Workload resources in Kubernetes**

The fundamental course Lab assumes the use of Docker for Desktop (Mac or Windows).

## ***Step 1 (Deploy a Pod)***

**The working directory is *materials/lab01_pod-rs-dep* directory.**

1．Review the contents of the manifest file `simple-pod.yaml` of the pod to be deployed.

```yaml
apiVersion: v1
kind: Pod           # Specify the resource type. The schema below spec are depended on the value of kind.
metadata:
  name: simple-pod  # Resource name
  labels:           # Define the label of the Pod
    app: simple
spec:               # Since kind is a pod, define "containers" as the set of containers that make up the pod.
  containers:
  - name: nginx     # Container name
    image: nginx:1.13   # Specify the container image
    env:            # Environment variables used by the container
    - name: BACKEND_HOST
      value: localhost:8080
    ports:          # Specify the port for EXPOSE
    - containerPort: 80 
  - name: alpine
    image: alpine
    command: ["/bin/sh"]
    args: ["-c", "while true; do date; sleep 10;done"]

```

2．Go to the working directory in Terminal and deploy the Pod with the `kubectl` command.

```sh
$ kubectl apply -f simple-pod.yaml
pod/simple-pod created
```

In this Lab, _nginx_ and _alpine_ containers are created in a pod called simple-pod.

3．Check the deployment status.

```sh
$ kubectl get pod
NAME READY STATUS RESTARTS AGE
simple-pod 2/2 Running 0 36m
```

 If the STATUS is "Running", the pod has been successfully deployed.  
 NAME: Pod name  
 READY: The number of launched containers in a Pod  
 STATUS: Pod state  
 AGE: Elapsed time for Pod startup  
 ＊ `kubectl get` command can be used to get basic information about a resource.

## ***Step 2 (Operate a Pod)***

`kubectl` command allows to access to the container in the created pod.

1.  Use the `kubectl exec` command to access the container and confirm the environment variables defined in the manifest. (`-c` option: If there are multiple containers in the pod, specify the container name.)  
 To log off from the container, either `exit` or "`Ctrl + d`" can be used.

```sh
$ kubectl exec -it simple-pod -c nginx -- bash
root@simple-pod:/# echo $BACKEND_HOST
localhost:8080
root@simple-pod:/# exit
```

 `kubectl` command allows to display the standard output of containers in a Pod.

2.  Now, view the logs of the container with the `kubectl logs` command. Also, the `-c` option can be used to specify the container name. With the `-f` option, it allow  output equivalent to the `tail -f` command.  

```sh
$ kubectl logs -f simple-pod -c alpine
Thu Jul 2 21:04:09 UTC 2020
Thu Jul 2 21:04:19 UTC 2020
```

## ***Step 3（Delete a Pod）***

Delete the Pod created earlier. Use the `kubectl delete` command to delete the created resource.  
In addition, the `kubectl delete` command can be used to delete resources based on manifest files.  
In the case of a manifest file based resource deletion, all resources described in the manifest will be deleted.  

1.  Delete the lod with the `kubectl delete` command.  
    (Either delete the manifest file base, or delete it when a pod is specified.)

```sh
＃ To delete with the manifest file base
$ kubectl delete -f simple-pod.yaml
pod "simple-pod" deleted

＃ To delete with the pod specified
$ kubectl delete pod simple-pod
pod "simple-pod" deleted
```

2.  Confirm that the Pod has been deleted.

```sh
$ kubectl get pod
No resources found.
```

## ***Step 4（Deploy a ReplicaSet）***

1.  Confirm the contents of the manifest file `simple-replicaset.yaml` for the pod to be deployed.

```yaml
apiVersion: apps/v1
kind: ReplicaSet  # Manifest for ReplicaSet
metadata:         # Metadata for ReplicaSet
  name: simple-rs 
  labels:
    app: simple 
spec:
  replicas: 2     # The number of Pod replicas 
  selector:
    matchLabels:  # The label of the Pod that the ReplicaSet manages (searches)
      app: simple 
  template:       # Below template is the same as the spec definition in the Pod resources
    metadata:
      labels:     # Define the label of the Pod
        app: simple
    spec:         # Define the containers (same as the definition in "simple-pod.yaml")
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

2.  Go to the working directory in Terminal and deploy the ReplicaSet with the `kubectl` command.

```sh
$ kubectl apply -f simple-replicaset.yaml
replicaset.apps/simple-rs created
```

The Pod to be deployed is the same definition as the Pod deployed in Step 1. The same number of pods as specified in `replicas` will be deployed.

3.  Confirm the deployment status.

```sh
$ kubectl get rs,pod -L app
NAME DESIRED CURRENT READY AGE APP
replicaset.extensions/simple-rs 2 2 2 67s simple

NAME READY STATUS RESTARTS AGE APP
pod/simple-rs-6b987 2/2 Running 0 67s simple
pod/simple-rs-n7dhw 2/2 Running 0 67s simple

```

Confirm that the Pod deployment is complete. The Pod name will be suffixed with a random identifier following "-". The status of the ReplicaSet is as follows.    
DESIRED: The number of pods specified in the manifest file    
CURRENT:  The number of pods currently running   
＊ `kubectl get` command allows you to retrieve information by specifying multiple resources. `-L` option to display the specified label of each Pod.    
Once the pods are deployed, you can perform the same operations as in Step 2.

## ***Step5（Confirm ReplicaSet behavior）***

1.  Deploy a Pod with the same label outside the ReplicaSet.    
Deploy the Pod with the same label using the `simple-pod.yaml` used in step 1. Checking the deployment status, the pod will be stopped as follows.

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

Since the ReplicaSet that is deployed is configured to manage and control two Pods, it mistakenly thinks there are too many Pods and tries to stop one of the three Pods. In this case, the last deployed Pod has been stopped, but be aware that in some cases, existing Pods will be deleted.  

2.  Remove the label from the Pod.  
**＊ Use the Pod name same as that of the Pod in your environment.**

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

When removing the `app` label from `simple-rs-6b987`, there will be only one Pod with the label `app=simple`, and the  ReplicaSet automatically adds one more Pod.   
This way, ReplicaSet manages and controls the number of pods to match the number specified by `replicas`.

## ***Step 6（Delete a ReplicaSet）***

1.  Use the `kubect delete` command to delete the ReplicaSet that is deployed.

```sh
$ kubectl delete -f simple-replicaset.yaml
replicaset.apps "simple-rs" deleted
```

2.  Delete the Pod whose label was removed in Step 5.  
**＊ Use the Pod name same as that of the Pod in your environment.**

```sh
$ kubectl delete pod simple-rs-6b987
pod "simple-rs-6b987" deletede
```

3.  Confirm that the resource has been deleted.

```sh
$ kubectl get rs,pod
No resources found.
```

## ***Step 7（Deploy a Deployment）***

1.  Review the manifest file for the Deployment to be deployed.

```yaml
apiVersion: apps/v1
kind: Deployment # Set the resource type to Deployment
metadata:        # Deployment Metadata
  name: simple-ds
  labels:
    app: simple 
spec:            # Same as the definition in ReplicaSet
  replicas: 2
  selector:
    matchLabels: # The label of the Pod that the ReplicaSet manages (searches)
      app: simple 
  template:      # Below template is the same as the spec definition in the Pod resources
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

2.  Go to the working directory in Terminal and deploy the ReplicaSet with the `kubectl` command.

```sh
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/simple-ds created
```

`--record` option allows to record the `kubectl` commands executed.   
The resources to be deployed are the same as those deployed in Step 4 (Deploy a ReplicaSet).

3.  Confirm the deployment status.

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

Use the `--selector` option to specify a label to retrieve information. In this case, the key `app` specified in the manifest file retrieves information by specifying a label with a value of *simple*.  
Confirm that the Deployment, ReplicaSet, and Pod have been created. The ReplicaSet name is also suffixed with a random identifier following "-".    
Also, check that the Deployment revision was created with the `kubectl rollout history` command.

```sh
$ kubectl rollout history deployment simple-ds
deployment.extensions/simple-ds
REVISION CHANGE-CAUSE
1 kubectl apply --filename=simple-deployment.yaml --record=true

```

REVISION is set to 1 as it is the initial one.

## ***Step 8（Replacement of ReplicaSet by Deployment update）***

Update the Deployment manifest file, and observe the following behavior during the deployment sequence.
* A new ReplicaSet is NOT generated even if only the number of Pods is updated.
* ReplicaSet replacement occurs by updating the container definition.

1.  Change the number of pod replicas in the manifest file.    
Change the `simple-deployment.yaml` file as follows and save it.  
* Change the value of `spec.replicas` from 2 to 3.   
After saving the file, deploy it with the `kubectl` command.

```sh
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/simple-ds configured
```

2.  Check the deployed resources and the Deployment revision.  

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

It can be confirmed that one new Pod has been created. Also, it is confirmed from the suffix that the ReplicaSet and the two existing Pods have not been replaced.  
Also, if a new ReplicaSet had been created, REVISION would been 2, but it is not shown.  
Judging from that, it is confirmed that changing `replicas` will not cause ReplicaSet replacement.

3.  Modify the container definition in the manifest file.    
Change the image of the "echo" container. Modify the `simple-deployment.yaml` file as follows and save it.  
* Change the value of `image` of the nginx container from nginx:1.13 to nginx:1.14.
After saving the file, deploy it with the `kubectl` command.  

```sh
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/simple-ds configured
```

Check the deployment status of the resource and confirm that the Pod and ReplicaSet are being replaced.

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

4.  Confirm the status of the deployed resources.    
When the ReplicaSet replacement is complete, it can be confirmed as follows.

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

By the suffixes of the Pod and ReplicaSet, it is possible to confirm that the replacement of resources has occurred. Also, it is possible to confirm that a new revision has been created with the `kubctl rollout hisroty` command. The container image change caused a revision with a REVISION value of 2 to be created.  

## ***Step 9（Execute Rollback）***

Deployment revisions are recorded, so it is possible to review the contents of a specific revision.  

1.  Using the `kubectl rollout hisroty` command, confirm the contents of the revision.

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

`--revision` option allows to confirm the contents by specifying a specific revision.

2.  Execute the rollback with the `kubectl rollout undo` command.

```sh
$ kubectl rollout undo deployment simple-ds --to-revision 1
deployment.extensions/simple-ds rolled backe
```

 `--to-revision` option allows to execute the rollout by specifying a specific revision. (If not specified, roll out to previous revision.)

3.  Confirm the status of the resource.

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

It can be confirmed that the previous ReplicaSet is being used.  

## ***Step 10（Delete a Deployment）***

1.  Delete the deployed resource with the `kubect delete` command.

```sh
$ kubectl delete -f simple-deployment.yaml
deployment.apps "simple-ds" deleted
```

2.  Confirm that the resource has been deleted.

```sh
$ kubectl get deployment,rs,pod --selector app=simple
No resources found.
```
