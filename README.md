# cloudnative-pg-operator

> PoC for deploying and operating the [CloudNativePG](https://cloudnative-pg.io/) operator

## Installation

```shell
→ # install operator in `cnpg-system` namespace
→ kubectl apply -f \
    https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.20/releases/cnpg-1.20.2.yaml

→ kubectl get deployments \
    -n cnpg-system cnpg-controller-manager

→ kubectl describe deployments \
    -n cnpg-system cnpg-controller-manager
```

## Deployment

### Secrets

We create two secrets, one for the admin and one for the app user, where in both cases the password equals the username.

### Cluster

```shell
→ kubectl apply -f \
    https://raw.githubusercontent.com/m99coder/cloudnative-pg-operator/main/cluster.yaml

→ # all pods are labeled with the cluster name
→ # kubectl get pods -n cnpg -l cnpg.io/cluster=cnpg-pg15-cluster
→ kubectl get pods -n cnpg
NAME                  READY   STATUS    RESTARTS   AGE
cnpg-pg15-cluster-1   1/1     Running   0          7m13s
cnpg-pg15-cluster-2   1/1     Running   0          6m23s
cnpg-pg15-cluster-3   1/1     Running   0          5m29s
```

While spinning up the cluster, one can see different sidecars in action. The first is `initdb` for setting up the initializing the database server. As soon as the primary is ready, the two secondaries run `join` sidecars before coming up on their own.

```shell
→ kubectl get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
cnpg-pg15-cluster-r    ClusterIP   10.100.63.101   <none>        5432/TCP   85m
cnpg-pg15-cluster-ro   ClusterIP   10.100.204.36   <none>        5432/TCP   85m
cnpg-pg15-cluster-rw   ClusterIP   10.100.8.124    <none>        5432/TCP   85m

→ kubectl get pods/cnpg-pg15-cluster-1 -o json | jq '.spec.containers[].ports'
[
  {
    "containerPort": 5432,
    "name": "postgresql",
    "protocol": "TCP"
  },
  {
    "containerPort": 9187,
    "name": "metrics",
    "protocol": "TCP"
  },
  {
    "containerPort": 8000,
    "name": "status",
    "protocol": "TCP"
  }
]

→ kubectl expose pods/cnpg-pg15-cluster-1 \
    --name=cnpg-pg15-cluster-1-metrics \
    --port=9187 \
    -n cnpg

→ kubectl get ep
NAME                          ENDPOINTS                                                    AGE
cnpg-pg15-cluster-1-metrics   172.24.220.162:9187                                          12s
cnpg-pg15-cluster-r           172.24.220.162:5432,172.24.220.178:5432,172.24.247.51:5432   4h22m
cnpg-pg15-cluster-ro          172.24.220.178:5432,172.24.247.51:5432                       4h22m
cnpg-pg15-cluster-rw          172.24.220.162:5432                                          4h22m

→ kubectl port-forward
    -n cnpg \
    svc/cnpg-pg15-cluster-1-metrics 9187

→ open http://localhost:9187/metrics
```

## Monitoring

```shell
→ kubectl get crds -n monitoring

→ # get endpoints for:
→ #   - alert manager (port 9093)
→ #   - prometheus (port 9090)
→ kubectl get endpoints -n monitoring

→ kubectl get podmonitors.monitoring.coreos.com -n cnpg
NAME                AGE
cnpg-pg15-cluster   152m

→ kubectl port-forward \
    -n monitoring \
    svc/prometheus-app 9090
```

Seems like the `PodMonitor` is not properly configured to monitor the cluster pods.

## Cleanup

```shell
→ # operator and cluster
→ kubectl delete clusters.postgresql.cnpg.io cnpg-pg15-cluster
→ kubectl delete namespace cnpg
→ kubectl delete -f \
    https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.20/releases/cnpg-1.20.2.yaml
```

## Resources

- <https://cloudnative-pg.io/>
- <https://cloudnative-pg.io/documentation/1.20/samples/cluster-example-full.yaml>
- <https://prometheus-operator.dev/>
