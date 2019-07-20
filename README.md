# Kubernetes Notes
This is a collection of snippets of commands I find over time will organize in future

## minikube
- Start minikube
``` shell
$ minikube start -p <NAME-OF-CLUSTER>
```

- Start Dashboard

``` shell
$ minikube dashboard -p <NAME-OF-CLUSTER>
```

## kubectl
- Kubectl config file found in `~/.kube/config` or run this command

``` shell
$ kubectl config view
```

- Get master info

### Connecting
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



