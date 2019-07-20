# Kubernetes Notes
This is a collection of snippets of commands and notes I find over time.


## Basic Commands
Get your hands wet before going down the rabbit hole. Just try to survive at
this stage if you are clueless af.

### minikube
- Start minikube
``` shell
$ minikube start -p <NAME-OF-CLUSTER>
```

- Start Dashboard

``` shell
$ minikube dashboard -p <NAME-OF-CLUSTER>
```

### kubectl
- Kubectl config file found in `~/.kube/config` or run this command

``` shell
$ kubectl config view
```

- Get master info

#### Connecting
- Serve over proxy

``` shell
$ kubectl proxy
```
    - We can issue curl commands over proxy
    ``` shell
    $ curl http://localhost:8001

    ```

- Calling API without proxy
    - **TOKEN BASED**
        - Get Token
        ``` shell
        $ TOKEN=$(kubectl describe secret -n kube-system $(kubectl get secrets -n kube-system | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t' | tr -d " ")
        ```

        - Get the API server endpoint
        ``` shell
        $ APISERVER=$(kubectl config view | grep https | cut -f 2- -d ":" | tr -d " ")
        ```


        - Confirm that the APISERVER stored the same IP as the Kubernetes master IP by issuing the following 2 commands and comparing their outputs:

        ``` shell
        $ echo $APISERVER
        https://192.168.99.100:8443

        $ kubectl cluster-info
        Kubernetes master is running at https://192.168.99.100:8443 ...

        ```

        - Access the API server using curl
        ``` shell
        $ curl $APISERVER --header "Authorization: Bearer $TOKEN" --insecure
        {
        "paths": [
        "/api",
        "/api/v1",
        "/apis",
        "/apis/apps",
        ......
        ......
        "/logs",
        "/metrics",
        "/openapi/v2",
        "/version"
        ]
        }
        ```

    - **CERT BASED**
        - Extract the client certificate, client key, and certificate authority data from the .kube/config file
        - Pass the certs to curl

        ``` shell
        $ curl $APISERVER --cert encoded-cert --key encoded-key --cacert encoded-ca
        ```


## Kubernetes Object Model

- Declare desired state under **spec** section.
- **spec** field is submitted to Kubernetes API server. Server tries to maintain
- Kubernetes system manages the **status** section and records the actual state
  of object. Kubernetes Control Plane tries to match the object's actual state
  with **spec**

- Types of Objects
    - Pods
    - Replica sets
    - Deployments
    - Namespace
    - etc.

- Specifying the yaml file. **Required fields**:
    1. `apiVersion`: Specified the api version endpoint on the server you want to
    connect to. It must match an existing version for the object defined

    1. `kind`: This can be `Deployment`, `Pod`, `Replicaset`, `Namespace`,
       `Service`, etc.

    1. `metadata`: Holds the object's basic information, such as `name`, `labels`, `namespace**, etc.

    1. `spec`: Specify the state for kuberentes to maintaink


**Example**

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.11
        ports:
        - containerPort: 80

```

### Pods
- A pod is the smallest and simplest Kubernetes Object
- Unit of deployment, which represents a single instance of the application
- **A pod is a logical collection of one or more containers** that:
    - Are scheduled together on the same host with the Pod
    - Share the same network namespace
    - Have access to mount the same external storage (volumes**.

- Specifying the yaml file.
    - `apiVersion`: must be `v1`
    - `kind`: specify as `Pod`
    - `metadata`: specify the `name` and `labels` here
    - `spec`: specify the desired state of the Pod aka `PodSpec`


**Example**
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.15.11
    ports:
    - containerPort: 80
```

### Labels
- Key Value Pair attached to Kubernetes objects (Pods, ReplicaSets)
- Labels are used to organize and select a subset of objects
- Many objects can have the same label(s). **Labels don't make objects unique**
- Controllers use Labels to logically group together decoupled objects, rather than using objects' names or IDs.

#### Label Selectors
- **Equality Based**
    - Match using `=` or `==` (both used interchangeably) or `!=`
    - _example:_ `env=dev`

- **Set based**
    - Match using `in` or `notin` for label values
    - Match using `exist` or `does not exist` for label keys
    - _example:_ `env in (dev,qa)`


### ReplicaSet and ReplicationController
- **Both ReplicaSet and ReplicationController control the number of replicas in a pod.**
- **Ensures that the number of pods always equal to spec**
- ReplicationController is deprecated.
- ReplicaSets support both equality- and set-based selectors, whereas ReplicationControllers only support equality-based Selectors. Currently, this is the only difference.


### Deployment
- **Declarative updates to Pods and ReplicaSets**
- The DeploymentController is part of the master node's controller manager, and it ensures that the current state always matches the desired state.
- DeploymentController allows for `rollbacks` and `rollouts`

### Namespaces
- If multiple users and teams use the same Kubernetes cluster we can partition
  the cluster into virtual sub-clusters using Namespaces.
- The names of the resources/objects created inside a Namespace are unique, but
  not across Namespaces in the cluster.
- Kubernetes generally creates four default namespaces:
    - `kube-system` Namespace contains the objects created by the Kubernetes
      system, mostly the control plane agents.
    - `default` Namespace contains the objects and resources created by administrators and developers. By default, we connect to the default Namespace.
    - `kube-public` is a special Namespace, which is unsecured and readable by anyone, used for special purposes such as exposing public (non-sensitive) information about the cluster.
    - `kube-node-lease` is the newest name apce which holds node lease objects used for node heartbeat data.

- Run `$ kubectl get namespaces` to get namespaces


### Example Deployment (rolling update and rollback)

1. Create a deployment
``` shell
$ kubectl create deployment mynginx --image=nginx:1.15-alpine
deployment.extensions "mynginx" created

```

1. Get the status
``` shell
// deploy = deployment, rs = replicaset, po = pods
// -l app=mynginx only lists data that pertains to mynginx
$ kubectl get deploy,rs,po -l app=mynginx
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/mynginx   1         1         1            1           2m

NAME                                       DESIRED   CURRENT   READY     AGE
replicaset.extensions/mynginx-66655d5b94   1         1         1         2m

NAME                           READY     STATUS    RESTARTS   AGE
pod/mynginx-66655d5b94-4qxqn   1/1       Running   0          2m

```

1. Scale the replicas

``` shell
$ kubectl scale deploy mynginx --replicas=3
deployment.extensions "mynginx" scaled

```

1. Get the status notice that we have 3 running pods now
``` shell
$ kubectl get deploy,rs,po -l app=mynginx
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/mynginx   3         3         3            3           5m

NAME                                       DESIRED   CURRENT   READY     AGE
replicaset.extensions/mynginx-66655d5b94   3         3         3         5m

NAME                           READY     STATUS    RESTARTS   AGE
pod/mynginx-66655d5b94-2jh9c   1/1       Running   0          30s
pod/mynginx-66655d5b94-4qxqn   1/1       Running   0          5m
pod/mynginx-66655d5b94-8v5b6   1/1       Running   0          30s
```

1. Describe the deployment

``` shell
$ kubectl describe deployment
Name:                   mynginx                                                                                                                                                                                                        [0/150]
Namespace:              default
CreationTimestamp:      Sat, 20 Jul 2019 17:32:06 +0800
Labels:                 app=mynginx
Annotations:            deployment.kubernetes.io/revision=1
Selector:               app=mynginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  app=mynginx
  Containers:
   nginx:
    Image:        nginx:1.15-alpine
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   mynginx-66655d5b94 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  7m    deployment-controller  Scaled up replica set mynginx-66655d5b94 to 1
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set mynginx-66655d5b94 to 3
```

1. Check history and revisions
``` shell
// Check history
$ kubectl rollout history deploy mynginx
deployments "mynginx"
REVISION  CHANGE-CAUSE
1         <none>

// Get more details regarding revision 1
$ kubectl rollout history deploy mynginx --revision=1
deployments "mynginx" with revision #1
Pod Template:
  Labels:       app=mynginx
        pod-template-hash=66655d5b94
  Containers:
   nginx:
    Image:      nginx:1.15-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

1. Perform a rolling update

``` shell
$ kubectl set image deployment mynginx nginx=nginx:1.16-alpine
deployment.apps "mynginx" image updated

```

1. Check history and revisions again

``` shell
$ kubectl rollout history deploy mynginx
deployments "mynginx"
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

// compare revision 1 and 2, noticed that the label should have changed
$ kubectl rollout history deploy mynginx --revision=1
deployments "mynginx" with revision #1
Pod Template:
  Labels:       app=mynginx
        pod-template-hash=66655d5b94    <--------- Notice this
  Containers:
   nginx:
    Image:      nginx:1.15-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>

$ kubectl rollout history deploy mynginx --revision=2
deployments "mynginx" with revision #2
Pod Template:
  Labels:       app=mynginx
        pod-template-hash=75cd85c66f    <--------- Notice this
  Containers:
   nginx:
    Image:      nginx:1.16-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>

```

1. Check the deployment. Notice that we now have two replica sets

``` shell
$ kubectl get deploy,rs,po -l app=mynginx
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/mynginx   3         3         3            3           17m

NAME                                       DESIRED   CURRENT   READY     AGE
replicaset.extensions/mynginx-66655d5b94   0         0         0         17m <-- original
replicaset.extensions/mynginx-75cd85c66f   3         3         3         3m  <-- new

NAME                           READY     STATUS    RESTARTS   AGE
pod/mynginx-75cd85c66f-6n8pw   1/1       Running   0          3m
pod/mynginx-75cd85c66f-jvs6l   1/1       Running   0          3m
pod/mynginx-75cd85c66f-vkmr8   1/1       Running   0          3m
```

1. If we are unhappy with this revision we can simply rollback to revision 1

``` shell
kubectl rollout undo deployment mynginx --to-revision=1
deployment.apps "mynginx"

```

1. Check the deployment history, we see rev 2 and 3 (orignally 1)

``` shell
$ kubectl rollout history deploy mynginx
deployments "mynginx"
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

// Notice how we reverted back to nginx:1.15-alpine in revision 3

$ kubectl rollout history deploy mynginx --revision=2
deployments "mynginx" with revision #2
Pod Template:
  Labels:       app=mynginx
        pod-template-hash=75cd85c66f
  Containers:
   nginx:
    Image:      nginx:1.16-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>

$ kubectl rollout history deploy mynginx --revision=3
deployments "mynginx" with revision #3
Pod Template:
  Labels:       app=mynginx
        pod-template-hash=66655d5b94
  Containers:
   nginx:
    Image:      nginx:1.15-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>

// Notice that revision 1 will be missing
$ kubectl rollout history deploy mynginx --revision=1
error: unable to find the specified revision
```



# Reference
[Introduction to Kubernetes Course on edX](https://courses.edx.org/courses/course-v1:LinuxFoundationX+LFS158x+2T2019)
