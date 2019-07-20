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


# Reference
[Introduction to Kubernetes Course on edX](https://courses.edx.org/courses/course-v1:LinuxFoundationX+LFS158x+2T2019)
