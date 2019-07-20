# Kubernetes Notes
This is a collection of snippets of ideas I get over time will organize in future

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

- Serve over proxy

``` shell
$ kubectl proxy
```
    - We can issue curl commands over proxy
    ``` shell
    $ curl http://localhost:8001

    ```

- Calling API without proxy
    - Get Token
    ``````

