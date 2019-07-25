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

## Access Management
### Overview
1. Stages of Access
    1. Authentication: Logs in a user
    1. Authorization: Authorizes the API requests added by the logged-in user
    1. Admission Control: Software modules that can modify or reject the
       requests based on some additional checks, like a preset Quota

### Authentication
1. Users (Kubernetes doesn't have a user object, but can use usernames for
   access control
    1. **Types 1: Normal Users** - managed via independent services like User/Client
       Certificates, a file listing usernames/passwords, Google accounts, etc

    1. **Type 2: Service Accounts** - With service accounts users, in-cluster
       processes communicate with the API server to perform different
       operations. Most of the Service Account users are created automatically
       via the API server, but they can also be created manually. The Service
       Account users are tied to a given Namespace and mount the respective
       credentials to communicate with the API server as Secrets.
    1. Additional Configurations that can be supported
        1. Anonymous Users
        1. User impersonation

1. Authentication modules
    1. Client Certificates
    1. Static Token File
    1. Bootstrap tokens (in beta status**
    1. Static Password File
    1. Service Account Token
    1. OpenID Connect TOkens
    1. Webhook Token Authentication
    1. Authenticating Proxy

### Authorization
1. Once we're authenticated we can send API requests to perform different
   operations. The API requests need to be authorized by Kubernetes

1. **Authorization Modules**
    1. **Node Authorizer**
        1. This is a special purpose authorization mode which specifically
        authorizes API requests made by kubelets.
        1. Authorizes Kubelets
            1. Read operations for services, endpoints, nodes,
            1. Write operations for nodes, pods, events

    1. **Attribute-Based Access Control (ABAC) Authorizer**
        1. With ABAC, Kubernetes grants access to API requests combining
           policies with attributes
        1. To enable the ABAC authorizer start the API server with
           `--authorization-mode=ABAC` option, specify the the authorization
           policy with `--authorization-policy-file=PolicyFile.json`

    1. **Role-Based Access Control (RBAC) Authorizer**
        1. With RBAC, Kubernetes regulate the access to resouces based on the
           roles of individual users.
        1. Restrict access via `create`, `get`, `update`, `patch`. These
           operations are referred to as verbs.
        1. In RBAC we can create two kinds of roles.
            1. Role: Use Role to grant access to resources within a specific Namespace
            1. ClusterRole: Use ClusterRole to grant permissions cluster wide
        1. In RBAC, there are two kinds of RoleBindings:
            1. RoleBinding: This allows us to bind users to the same namespace
               as Role.
            1. ClusterRoleBinding: This allows us to grant access to resources
               at a cluster-level and to all Namespaces
        1. Enable RBAC via `--authorization-mode=RBAC`

    1. **Webhook Authorizer**
        1. This is used to offer authorization decisions to third-party
           services, which would return true for successful authorization, and
           false for failure.
        1. In order to enable the Webhook authorizer, we need to start the API
           server with `--authorization-webhook-config-file=SOME_FILENAME`

### Admission Control
1. Allows for more granular access to control policies.
    1. Allows for Privileged containers
    1. Checking on resource quota

1. We can force policies using different admission controllers. These come after
   the API requests are authenticated and authorized
    1. ResourceQuota,
    1. DefaultStorageClass,
    1. AlwaysPullImages

1. To use Admission controls, start the Kubernetes API server with
   `--enable-adission-plugins` this takes in a comma-delimited, ordered list of
   controller names. Example
   ```
    --enable-admission-plugins=NamespaceLifecycle,ResourceQuota,PodSecurityPolicy,DefaultStorageClass
   ```

### Example
1. To enable authorization on minikube start with additional options

``` shell
$ minikube start \
--extra-config=controller-manager.ClusterSigningCertFile="var/lib/localkube/certs/ca.crt" \
--extra-config=controller-manager.ClusterSigningKeyFile="/var/lib/localkube/certs/ca.key" \
--extra-config=apiserver.Authorization.Mode=RBAC

```


## Services
- Services are used to group Pods to provide common access points from the
  external world to the containerized application.

### Connecting Users to Pods
- To access the kubernetes app, a user/client needs to connect Pods.
- **Problem:** Pods are ephemeral they can die anytime. IP address allocated to it
  cannot be static. Suppose if a pod dies and respawns, the IP address would
  have changed. A client won't know which new IP address to connect to.jj
- Services are created to overcome this problem

### Services
- We group pod into services using `labels` and `selectors`
    - labels are key value pair associated to pods as such `app:front`
    - Select the labels using `select app==front`
    - We can assign a name to the logical grouping such as `ap==front` into a
      service `frontend-svc`

- Services can expose single Pods, ReplicaSets, Deployments, DaemonSets, and StatefulSets

### Service Object Example

``` yaml
kind: Service
apiVersion: v1
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
```
- Specify the selector under spec
- To expose the app you can expose ports under the spec. In this case, we're
  listening on port 80 and forwarding to the targetPort 5000.

### kube-proxy
- All worker nodes run a daemon called kube-proxy.
- kube-proxy watches the API server on the master node for the addition and
  removal of Services and endpoints.
- Using `iptables` kube-proxy is able to capture the traffic for its ClusterIP
  and forward it to one of the Service endpoints.
    - Any node can receive the external traffic and then route it internally in
      the cluster based on the `iptables` rules.
    - If the Service is removed, kube-proxy removes the corresponding `iptables`
      rules on all nodes as well.

### Service Discovery
We need a way to discover services at runtime. Kubernetes supports two methods
for discovering Services.

1. Environment Variables
    - As soon as the Pod starts on any worker node, the kubelet daemon running
      on that node adds a set of environment variables in the Pod for all active
      Services.
    - For example, if we have an active Service called redis-master, which
      exposes port 6379, and its ClusterIP is 172.17.0.6, then, on a newly created
      Pod, we can see the following environment variables:
      ```
        REDIS_MASTER_SERVICE_HOST=172.17.0.6
        REDIS_MASTER_SERVICE_PORT=6379
        REDIS_MASTER_PORT=tcp://172.17.0.6:6379
        REDIS_MASTER_PORT_6379_TCP=tcp://172.17.0.6:6379
        REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
        REDIS_MASTER_PORT_6379_TCP_PORT=6379
        REDIS_MASTER_PORT_6379_TCP_ADDR=172.17.0.6
      ```
    - **Take note of the ordering of Serivces, as Pods will not have environment
      variables set for Services which are created after the Pods are created**

2. DNS
    - Kubernetes has an add-on for DNS
    - It creates a DNS record for each Service and its format is `my-svc.my-namespace.svc.cluster.local`
    - Services within the same Namespace find other Services just by their name.
    - If we add a Service `redis-master` in `my-ns` namespace, all pods in the
      same Namespace lookup the service just by its name, `redis-master`.
    - Pods from other namespaces lookup the same Service by adding the
      respective Namespace as a suffix, such as `redis-master.my-ns`
    - **This is the most common and highly recommended solution**

### ServiceType
- While defining a Service, we can also choose its access scope. We can decide
  whether the service:
    - is only accessible within the cluster
    - is accessible from within the cluster and the external world.
    - Maps to an entity which resides either inside or outside the cluster.

- Access Scope is decided by ServiceType, which can be configured when creating
  the Service

- **ClusterIP** this is the default ServiceType. A Service receives a Virtual Ip
  address, known as its CluterIP. This virtual IP address is ued for
  communicating with the Service and is accessible only within the cluster

- **NodePort** is a high-port, dynamically picked from the default range
  `30000-32767` mapped to the respective Service from all the worker nodes.
    - **A NodePort ServiceType is useful when we want to make our Servies accessible
       to the outside world**
    - The end-user connects to any worker node on the specified high-port, which
      proxies the request internally to the ClusterIP of the Service, then the
      request is forwarded to the application running inside the cluster.
    - To access multiple applications from the external world, administrators
      can configure a reverse proxy - an ingress, and define rules that target
      Services within the cluster.

- **LoadBalancer**
    - NodePort and ClusterIP are automatically created, and the external load
      balancer will route to them
    - The Service is exposed at a static port on each worker node
    - The Service is exposed externally using the underlying cloud provider's
      load balancer feature
    - **NOTE** if the cloud service provider underlying infrastructure doesn't
      support the automatic creation of Load Balancers and supports Kubernetes,
      the LoadBalancer IP address field is not populated, and the Service will
      work the same way as a NodePort type Service.

- **ExternalIP**: A Service can be mapped to an ExternalIP address if it can
  route to one or more of the worker nodes. Traffic that is ingressed into the
  cluster with the ExternalIP (as destination IP) on the Service port, gets
  routed to one of the Service endpoints. This type of service requires an
  external cloud provider such as Google Cloud Platform or AWS.
    - **NOTE** ExternalIPs are not managed by Kubernetes. The cluster
      administrator has to configure the routing which maps the ExternalIP
      address to one of the nodes.

- **ExternalName**: This is a special ServiceType, that has no Selectors and
  does not define any endpoints. When accessed within the cluster, it returns a `CNAME`
  record of an externally configured Service.
    - This is primary used to make externally configured Services like
        `my-database.example.com` available to eapplications inside the cluster.
    - If the externally defined Service resides witin the same Namespace, using
      just the name `my-database` would make it available to other applications
      and Services within that same Namespace.


## Deployment
### Deployment via Dashboard
1. In this example we will deploy an nginx webserver.

1. Start Minikube

``` shell
// NOTE you can specify the cluster via
// $ minikube start -p clustername
// If done in this way note that other commands must include the -p clustername.
// eg. $ minkube status -p clustername

$ minikube start

$ minikube status
host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100

$ minkube dashboard
```

1. You should be able to view the dashboard at the link below. Note that the
   port number may not be the same. If you are starting the dashboard for the
   first time note that you should be directed to the dashboard page

``` shell
http://127.0.0.1:37751/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/
```

1. At the dashboard page click th `+CREATE` tab at the top right. From here you
   can either create an application using a valid YAML/JSON config file or
   manually from the `CREATE AN APP`

1. Click on `CREATE AN APP` you will need to provide the following details
    1. App Name: Set this to `webserver`
    1. Container Image: Set this to `nginx:alpine`
    1. Number of pods aka ReplicaCount: Set this to `3`
    1. Service: Set this to `None` for now, we can create this later
    1. Under `Show Advanced Options` there are label, namespace, env variables,
       etc. By default, the app label value is set to our application name.

1. Check the deployment on CLI

``` shell
// List the Deployments
$ kubectl get deployments OR $ kubectl get deploy
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
webserver   3         3         3            3           2m

// List the replicaSets
$ kubectl get replicasets OR $ kubectl get rs
NAME                   DESIRED   CURRENT   READY     AGE
webserver-6c56d468fc   3         3         3         3m

// List the pods
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
webserver-6c56d468fc-42dt9   1/1       Running   0          4m
webserver-6c56d468fc-7r2c7   1/1       Running   0          4m
webserver-6c56d468fc-l5dmk   1/1       Running   0          4m
```

1. Get more info using labels and selectors using `kubectl describe`

``` shell
$ kubectl describe pod webserver-6c56d468fc-42dt9
```

1. List the pods using their attached labels, note that label2 is empty

``` shell
$ kubectl get pods -L k8s-app,label2
NAME                         READY     STATUS    RESTARTS   AGE       K8S-APP     LABEL2
webserver-6c56d468fc-42dt9   1/1       Running   0          16m       webserver
webserver-6c56d468fc-7r2c7   1/1       Running   0          16m       webserver
webserver-6c56d468fc-l5dmk   1/1       Running   0          16m       webserver
```

1. List the pods with a given label. Note `=` and `==` can be used interchangeably

``` shell
$ kubectl get pods -l k8s-app==webserver OR $ kubectl get po -l k8s-app=webserver
NAME                         READY     STATUS    RESTARTS   AGE
webserver-74d8bd488f-dwbzz   1/1       Running   0          17m
webserver-74d8bd488f-npkzv   1/1       Running   0          17m
webserver-74d8bd488f-wvmpq   1/1       Running   0          17m

// Note if you give a different label selector you shouldn't see anything
$ kubectl get po -l k8s-app=webserver
No resources found
```

### Deployment using the CLI
1. Delete the previous deployment

``` powershell
$ kubectl delete deployments webserver

// check if replicasets and pods are still found
$ kubectl get rs
No resources found

$ kubectl get po
No resources found
```

1. Create a yaml config file with the following deployment details

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
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
        image: nginx:alpine
        ports:
        - containerPort: 80
```

1. Create the deployment using the `-f` option

``` shell
$ kubectl create -f webserver.yaml
deployment.apps/webserver created
```

1. Check replicasets and pods

``` shell
$ kubectl get rs
NAME                   DESIRED   CURRENT   READY   AGE
webserver-5c69f5ccbf   3         3         3       63s

$ kubectl get po
NAME                         READY   STATUS    RESTARTS   AGE
webserver-5c69f5ccbf-jfm9t   1/1     Running   0          72s
webserver-5c69f5ccbf-kg6jk   1/1     Running   0          72s
webserver-5c69f5ccbf-knc4f   1/1     Running   0          72s
```

### Exposing an Application
1. We will need to define a service to expose the pods. Create
   `webserver-svc.yaml` file with the following contents

``` shell
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    run: web-service
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
```

1. Create th webserver-svc

``` shell
$ kubectl create -f webserver-svc.yaml
service/web-service created
```

1. At this point you can explicitly expose the previously created Deployment
   using the `expose` command. **NOTE: It is not necessary to create the
   deployment first then create the service. This can be done in any order**

``` powershell
$ kubectl expose deployment webserver --name=web-service --type=NodePort
service/web-service exposed
```

1. List the services. Notice the cluster ip and port of web-service

``` shell
$ kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP        43h
web-service   NodePort    10.101.143.185   <none>        80:30573/TCP   22m


// Get more details about the service
$ kubectl describe service web-service
Name:                     web-service
Namespace:                default
Labels:                   run=web-service
Annotations:              <none>
Selector:                 app=nginx
Type:                     NodePort
IP:                       10.101.143.185
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30573/TCP
Endpoints:                172.17.0.4:80,172.17.0.6:80,172.17.0.7:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

1. Once the application is running we can get the ip address of minikube VM and
   goto the exposed port found via `kubectl get services`

``` shell
// If using default
$ minikube ip

// If using -p
$ minikube ip -p clustername

192.168.99.120
// Now you can open the browser and access the application on
// 192.168.99.120:30573 for example
// Alternatively, you can run
$ minikube service web-service
```

### Liveness and Readiness Probes
- Liveness and Readiness Probes allows `kubelet` to control the health of the
  application running inside apod.

- Readiness Probe checks if the containers are ready. Allow enough time for the
  Readiness Probe to possibly fail a few times before a pass then check the
  Liveness Probe. **If the Liveness Probe and Readiness Probe overlaps the
  container will never reach the ready state**

- Liveness Probe are used to check if the application is alive. if the
  application is unresponsive either due to deadlock or memory pressure. The
  container is effectively dead. You should restart the container to make the
  application available. Liveness probes help us restart the container
  programmatically should a health check fail.
    - Liveness Probes can be set by defining:
        - Liveness Command
        - Liveness HTTP request
        - TCP Liveness Probe

### Liveness Probes
#### Liveness Command
- In this example we check for the existence of `/tmp/healthy`. We configure a
  livenessProbe to check for this every 5 seconds. Suppose if we do not find
  `/tmp/healthy`, the Pod would get restarted.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

#### Liveness HTTP Request
- In this example, we make kubelet send a `HTTP GET` REQUEST TO THE `/healthz`
  endpoint of the application on port 8080. If we get a failure, the kubelet
  will restart the affected container; otherwise, it would consider the
  application to be alive.

``` yaml
livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

#### TCP Liveness Probe
- In this example, we make kubelet try to open the TCP socket to the container
  running the application. If it succeeds, the application is considered
  healthy, otherwise the kubelet would mark it as unhealthy and restart the
  affected container

``` yaml
livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

### Readiness Probes
- We use readiness probes to check if the application meet certain conditions
  before they can serve traffic.
- Using readiness probes we can ensure:
    - that the depending service is ready
    - some large data set is loaded
    - etc.
-  A pod with containers that do not report ready status will not receive
   traffic from kuberentes services

- Readiness are similar configured to liveness probes. Example

``` yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Kubernetes Volume Management
Volumes are use to persist state. Pods are ephemeral and will die losing all the
temporary data with them. ie. when kubelet restores the state it will be a clean
state. Sometimes we need a way to restore the data even after the pods die.

### Volume Types
- emptyDir
    - An empty volume is created for a pod as soon as it is scheduled on the
      worker node. The volume life is coupled to the Pod. Once the pod is
      terminated, contents in the emptyDir is deleted forever.

- hostPath
    - We share contents from the host with the pod. If the pod is deleted
      contents can still be found on the host.

- gcePersistentDisk
    - Using gcePersistentDisk Volume type, we can mount a google compute engine
       persistent disk into a pod

- awsElasticBlockStore
    - Using this we can mount an AWS EBS Volume to a pod

- azureDisk
    - With azureDisk we can mount a Microsoft Azure Data Disk into a Pod

- azureFile
    - With azureFile we can mount a Microsoft Azure File Volume into a Pod

- cephfs
    - With cephfs, an existing CephFS volume can be mounted into a Pod. When a
      Pod terminates, the volume is unmounte and the contents of the volume are
      preserved

- nfs
    - With nfs, we can mount an NFS share into a Pod

- iscsi
    - With iscsi, we can mount an iSCSI share into a Pod

- secret
    - With the secret Volume Type, we can pass sensitive information, such as
      passwords, to Pods.

- configMap
    - With configMap objects, we can provide configuration data, or shell
      commands and arguments into a Pod

- persistentVolumeClaim
    - We can attach a PersistentVolume to a Pod using a persistentVolumeClaim

### PersistentVolumes
- In the typical IT organization, storage is managed by sysadmin. The end use
  just receives instruction to use the storage but is abstracted from the
  storage management

- Containers would like to follow a similar principle. However, given the large
  variations in volume types this is difficult.

- Kubernetes resolves this by introducing the **PersistentVolume (PV)**
  subsystem.
    - PV provides APIs for user and admin to manage and consume persistent
      storage.
    - PVs are network-attached storage in the cluster which is provisioned by
      the admin
    - To manage the volume, it uses **PersistentVolume API*** resource type
    - To consume the volume it uses the **PersistentVolumeClaim API** and
      consumes it

- PV can be dynamically provisioned based on the StorageClass resource. A
  StorageClass contains pre-defined provisioners and parameters to create a
  PersistentVolume.

- Using PersistentVolumeClaims, a user sends the request for dynamic PV
  creation, which gets wired to the StorageClass resource.

- Refer to
  [docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)
  to see the types of volume types that support managing storage using
  Persistent Volumes

### PersistentVolumeClaims
- A **PersistentVolumeClaim (PVC)** is a request for storage by a user.
- Users request for PersistentVolume resources based on type, access mode, and size.
- There are 3 access modes:
    - ReadWriteOnce (read-write by a single node)
    - ReadOnlyMany (read-only by many nodes)
    - ReadWriteMany (read-write by many nodes)
- Once a suitable PersistentVolume is found, it is bound to a PersistentVolumeClaim
- Once bound, the PersistentVolumeClaim can be used in a Pod
- Should a user finish work, it can release the attached PersistentVolumes. The
  underlying PersistentVolumes can be
    - reclaimed (for an admin to verify or aggregate data)
    - deleted (both data and volume are deleted)
    - recycled (only the data is deleted)
- More info in [docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)

### Container Storage Interface (CSI)
- Container Orchestrators like Kubernetes, Mesos, Docker, Cloud Foundry used to
  have their own methods of managing storage using Volumes.
- For storage vendors, it was challenging to manage different Volume plugins
  for different orchestrators.
- This called for standardization of the Volume interface. This specification is
  found [here](https://github.com/container-storage-interface/spec/blob/master/spec.md**
- CSI went from alpha to stable support from release v1.9 and v1.13.


## ConfigMaps and Secrets
- You will need to pass runtime parameters like
    - configuration details
    - permissions
    - passwords
    - token
- **ConfigMaps API** resources are used in the case when you need to share information between
  applications
- **Secret API** resources act like ConfigMaps but for private information

### ConfigMaps
- ConfigMaps allow us to decouple the configuration details from the container
  image.
- Configuration data are stored as **key-value pairs**
- Information in ConfigMaps are consumed by Pods or any other system components
  and controllers, environment variables, sets of commands and arguments, or
  volumes.
- You can create ConfigMaps from literal values, from configuratio files, from
  one or more files or directories.

#### Create a ConfigMap from Literal Values
1. Create the ConfigMap

``` shell
$ kubectl create configmap my-config \
> --from-literal=key1=value1 \
> --from-literal=key2=value2

configmap/my-config created
```

1. Display the ConfigMap Details for `my-config`

``` shell
// use -o yaml to spit the output in yaml format
$ kubectl get configmaps my-config -o yaml
```

#### Create a ConfigMap from a Configuration File
1. Create a configuration file with the following content and name it `customer1-configmap.yaml`. Make sure you specify
   **apiVersion**, **kind**, **metadata**, **data**

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: customer1
data:
  TEXT1: Customer1_Company
  TEXT2: Welcomes You
  COMPANY: Customer1 Company Technology Pct. Ltd.
```

1. Once you created this file you can run the command

``` shell
$ kubectl create -f customer1-configmap.yaml
configmap/customer1 created
```

#### Create a ConfigMap from a File
1. Create a configuration file `permission-reset.properties` with the following
   data
```
permission=read-only
allowed="true"
resetCount=3
```

1. Run the create configmap with the following command

``` shell
$ kubectl create configmap permission-config --from-file=permission-reset.properties
```

### Use ConfigMaps Inside Pods
- **OPTION 1:** As environment variables
    - Inside a container, we can retrieve the key-value data of an entire
      ConfigMap or the values of specific ConfigMap keys as environment
      variables.
    - Use `full-config-map` to retrieve all the values of the ConfigMap keys
        - Example:
        ``` yaml
        ...
        containers:
        - name: my-app-full-container
            image: myapp
            envFrom:
            - configMapRef:
            name: full-config-map
        ...
        ```
    - Alternatively, you can specify specific key-value pairs from separate
      ConfigMaps in the following manner
        - Example:
        ``` yaml
        ...
          containers:
          - name: myapp-specific-container
            image: myapp
            env:
            - name: SPECIFIC_ENV_VAR1
              valueFrom:
                configMapKeyRef:
                  name: config-map-1
                  key: SPECIFIC_DATA
            - name: SPECIFIC_ENV_VAR2
              valueFrom:
                configMapKeyRef:
                  name: config-map-2
                  key: SPECIFIC_INFO
        ```

- **OPTION 2:** As Volumes
    - We can mount a vol-config-map ConfigMap as a Volume inside a Pod. For each
      key in the ConfigMap, a file gets created in the mount path (where the
      file is named with the key name) and the content of that file becomes the
      respective key's value.
    - Example:
    ``` yaml
    ...
      containers:
      - name: myapp-vol-container
        image: myapp
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: vol-config-map
    ```

### Secrets
- If you store secrets in Deployment YAML file they will be exposed. The secrets
  would be available to anyone who has access to the configuration file
- The Secret object helps us by allowing us to encode sensitive information
  before sharing it.
- Secret object stores information like passwords, token, or keys in
  **key-value** pairs similar to ConfigMaps
- We refer to the Secret object without exposing its content.
- **IMPORTANT** Scret data is stored as plain text inside **etcd**, administrators
  must limit access to the API server and **etcd**.

#### Create a Secret from Literal
- To create a Secret we can use `kubectl create secret**

``` shell
$ kubectl create secret generic my-password --from-literal=password=mysqlpassword
```

- After successfully creating a secret we can analyze it with the `get` and
  `describe` commands. They do not reveal the content of the secret. The type is
  listed as Opaque.

``` shell
$ kubectl get secret my-password
NAME                  TYPE                                  DATA   AGE
my-password           Opaque                                1      7s

$ kubectl describe secret my-password
Name:         my-password
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  13 bytes
```

#### Create a Secret from a Configuration File
- We can create a Secret manually from a YAML config file.
- **USING base64**
    - Encode the string you want to save in base64, note that this is just an
    encoding and not an encryption. Anyone can easily decode this and get your secret!
    ``` shell
    $ echo secretstring | base64
    c2VjcmV0c3RyaW5nCg==

    // Note
    $ echo "c2VjcmV0c3RyaW5nCg==" | base64 --decode
    secretstring
    ```

    - Use it in the configuration file save as `mypass-base64.yaml`
    ``` yaml
    apiVersion: v1
    kind: Secret
    metadata:
        name: my-password
    type: Opaque
    data:
        password: c2VjcmV0c3RyaW5nCg==
    ```
    - Now you can run `kubectl create`
    ``` shell
    $ kubectl create -f mypass-base64.yaml
    secret/my-password created
    ```

    - **FYI** if you see a `Error from server (AlreadyExists): error when creating "mypass-base64.yaml": secrets "my-password" already exists`
      you can either delete the previous `my-password` or rename this secret.
    ``` shell
    $ kc delete secret my-password
    secret "my-password" deleted
    ```

- **USING stringData**
    - Create a a new configuration file, save as `mypass-string.yaml`
    ``` yaml
    apiVersion: v1
    kind: Secret
    metadata:
        name: my-password
    type: Opaque
    stringData:
        password: secretstring
    ```
    - Now you can run `kubectl create`
    ``` shell
    $ kubectl create -f mypass-string.yaml
    secret/my-password created
    ```

#### Create a Secret from a File
- We can create a secret from a file using the `kubectl create secret` command
- First encode the sensitive data into base64 and store it in a file
``` powershell
$ echo secretstring | base64 > password.txt
```
- Run the kubectl create secret command

``` shell
$ kubectl create secret generic my-password --from-file=password.txt
secret/my-password created
```

### Using Secrets inside Pods
- Like ConfigMaps, Secrets can be consumed by containers inside pods as
    1. Environment variables
    1. mounted data volumes
- You can reference secrets in their entirety or specific key-values

- **Using Secrets as Environment Variables**

``` yaml
...
spec:
  containers:
  - image: wordpress:4.7.4-apache
    name: wordpress
    env:
    - name: WORDPRESS_DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-password
          key: password

...
```

- **Using Secrets as Files from a Pod**

``` yaml
...
spec:
  containers:
  - image: wordpress:4.7.4-apache
    name: wordpress
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secret-data"
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-password
```

## Ingress
- Alternative way to expose services to the world. LoadBalancers are expensive.
  We can use ingress to help with this.

- Ingress allows you to access cluster Services by configuring a Layer 7
  HTTP/HTTPS load balancer for Services and provides the following
    - TLS (Transport Layer Security)
    - Name-based virtual hosting
    - Fanout routing
    - Loadbalancing
    - Custom rules

- With Ingress, users don't connect directly to services. Instead, users reach
  the Ingress endpoint and from there is forwarded to the desired service.
    - **Name-based Virtual Hosting** connect to `blue.example.com` or `green.example.com`:
    ``` yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: virtual-host-ingress
      namespace: default
    spec:
      rules:
      - host: blue.example.com
        http:
          paths:
          - backend:
            serviceName: webserver-blue-svc
            servicePort: 80
      - host: green.example.com
        http:
          paths:
          - backend:
            serviceName: webserver-green-svc
            servicePort: 80
    ```
    - **Fanout Ingress Rules** connect to `example.com/blue` or `example.com/green`
    ``` yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: fan-out-ingress
      namespace: default
    spec:
      rules:
      - host: example.com
        http:
          paths:
          - path: /blue
            backend:
              serviceName: webserver-blue-svc
              servicePort: 80
          - path: /green
            backend:
              serviceName: webserver-green-svc
              servicePort: 80
    ```

### Ingress Controller
-  The ingress Controller is an application watching the Master Node's API
   server for changes in the ingress resources and updates the Layer 7 Load
   Balancer accordingly.

- Kubernetes supports different Ingress Controllers, and, if needed we can also
  build our own.

- Commonly used Ingress Controllers are
    - GCE L7 Load Balancer Controller
    - Nginx Ingress Controller
    - Istio
    - Kong
    - Traefik

- Start the Ingress Controller with Minikube. **NOTE: Ingress Controller is
  disabled by default**
``` shell
$ minikube addons enable ingress
```

### Deploy an Ingress Resource
- Once the Ingress Controller is deployed, we can create an Ingress resouce
  using the `kubectl create` command.
  ``` shell
  $ kubectl create -f virtual-host-ingress.yaml
  ```


## Advanced Topics
### Annotations
- Non-identifying key-value data unlike labels
- Mainly used to
    1. Store build/release IDs, PR numbers, git branch
    1. Phone/pager numbers of people responsible
    1. Pointers to logging, monitoring, analytics, audit repositories, debugging tools

- Example we can use annotations when creating a Deployment
``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webserver
  annotations:
    description: Deployment based PoC dates 2 May 2019
```

- Display annotations

``` shell
$ kubectl describe deployment webserver
Name:                webserver
Namespace:           default
CreationTimestamp:   Fri, 03 May 2019 05:10:38 +0530
Labels:              app=webserver
Annotations:         deployment.kubernetes.io/revision=1
                     description=Deployment based PoC dates 2nd May 2019
```

### Jobs and Cronjobs
- A Job creates one or more Pods to perform a task
- The Job object takes the responsibility of Pod failures. It make sure that the
  given task is completed successfully.
- Once the task is complete, all the Pods that have been created will be
  terminated automatically.
- Job Options Include:
    - **parallelism**: Set the number of pods allowed to run in parallel
    - **completions** set the number of expected completions
    - **activeDeadlineSeconds**: Set the duration of the job
    - **backoffLimit**: Set the nubmer of retries before Job is marked as failedjjjjkj
- Starting with Kubernetes 1.4 we can also perform jobs at scheduled times/dates
  with **CronJobs**
- CronJob Options Include:
    - **startingDeadlineSeconds**: to set the deadline to start a Job if
      scheduled time was missed
    - **concurrencyPolicy**: To allow or forbid concurrent Jobs or to replace
      old Jobs with new ones.

### Quota Management
- When there are many users sharing a Kubernetes cluster we need to ensure fair
  usage.
- We can set the following types of quotas per Namespace:
    - **Compute Resource Quota**
    - **Storage Resource Quota**
    - **Object Count Quota**

### Autoscaling
- While it is still fairly easy to manually scale a few Kubernetes objects, this
  may not be a practical solution for a production-ready cluster where hundreds
  or thousands of objects are deployed.

- We will need a dynamic scaling solution which adds or removes objects from the
  cluster based on resource utilization, availability, and requirements.

- Autoscaling is implemented in a Kubernetes Cluster via controllers which
  periodically adjust the number of running objects basedo n single, multiple,
  or custom metrics. There are various types of autoscalers available in
  Kubernetes which can be implemented individually or combined for a more robust
  autoscaling solution
    - **Horizontal Pod Autoscaler (HPA)**
    - **Vertical Pod Autoscaler (VPA)**
    - **Cluster Autoscaler**

### DaemonSets
- DaemonSets are objects that are running on all nodes at all times.
- Sometimes we may need an object to collect monitoring data from all nodes.
  DaemonSets are the solution to this.
- It is a critical controller API resource for multi-node Kubernetes clusters
- The kube-proxy agent running as a Pod on every single node in the cluster is
  managed by a `DaemonSet`
- If a node is added to a cluster, a pod from a given DaemonSet is automatically
  created on it.
    - Although it ensures an automated process, the DaemonSet's Pods are placed
      on nodes by the cluster's default Scheduler
    - If the node dies or it is removed from the cluster, the respective Pods are
      garbage collected.
    - If a DaemonSet is deleted, all Pods it created are deleted as well
- A newer feature of the DaemonSet resource allows for its Pods to be scheduled
  only on specific nodes by configuring nodeSelectors and node affinity rules.
- DaemonSets support rolling updates and rollbacks.

### StatefulSets
- The StatefulSet controller is used for stateful applications which require a
  unique identity, such as name, network identifications, strict ordering, etc.
- Examples inclue MySQL cluster, etcd cluster.
- The StatefulSet controller provides identity and guaranteed ordering of
  deployment and scaling to Pods.
- Similar to Deployments, StatefulSets use ReplicaSets as intermediary Pod
  controllers and support rolling updates and rollbacks.

### Kubernetes Federation
- With Kubernetes Cluster Federation we can manage multiple Kubernetes clusters
  from a single control plane.
- This is still an Alpha feature.
- It is useful when we want to build hybrid solution where we can have clusters
  running inside our private datacenter and another cluster running in the
  public cloud, allowing us to avoid provider lock-in
- You may assign different weights for each cluster in the Federation to
  distribute the load based on custom rules.

### Custom Resources
- Most of the time the provided resources are sufficient.
- Should we need additional functionality we can create Custom Resources. Doing
  so we won't need to modify Kubernetes' Source
- Custom resources are dynamic and can appear and disappera in an already
  running cluster at any time.
- To make a resource declarative, we must create and install a custom
  controller, which can interpret the resource structure and perform the
  required actions.
- Custom controllers can be deployed and managed in an already running cluster
- **Two ways to add custom resource**
    1. **Custom Resource Definitions (CRDs)**: Easiest way as no custom resources
       are required
    1. **API aggregation**: Allows for more fine-grained control

### Helm and Tiller
- To deploy application we use different Kubernetes Manifests such as
  Deployments, Services, Volume Claims, Ingress, etc.
- It can be tiresome to deploy them on by one. We can bundle all those manifest
  after templatizing them into a well-defined format, along with other metadata.
- This bundle is referred to as Chart.
- There Charts can then be served via repositories, such as those we have for
  rpm and deb packages.
- **Helm** is a package manager analogous to `yum` and `apt`
- Helm has two components
    - **Helm** which runs on your user's workstation
    - **Tiller** which runs inside your kubernetes cluster
- View sample charts [here](https://github.com/helm/charts)

### Security Context and Pod Security Policies
- At times we need to control specifi privileges and access control settings for
  Pods and Containers
- **Security Contexts**
    - Allows us to set Discretionary Access Cotnrol for object access
      permissions, privileged running, capabilities, security labels, etc.
    - However, their effect is limited to the individual Pods and Containers
      where such context configuration settings are incorporated in the `spec` section
- **Pod Security Policies**
- Used to apply security settings to multiple Pods and Containers cluster-wide.
- Allows for more fine-grained security settings to control the usage of the:
    - host namespace,
    - host networking and ports,
    - file system groups,
    - usage of volume types,
    - enforce container user and group ID, root privilege escalation

### Network Policies


# References
[Introduction to Kubernetes Course on edX](https://courses.edx.org/courses/course-v1:LinuxFoundationX+LFS158x+2T2019)
