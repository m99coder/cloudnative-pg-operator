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

```shell
→ kubectl create namespace cnpg

→ cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cnpg-pg15-cluster
  namespace: cnpg
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:15.4
  primaryUpdateStrategy: unsupervised
  storage:
    size: 1Gi
  monitoring:
    enablePodMonitor: true
EOF

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

→ # monitoring
→ kubectl delete namespace cnpg-monitoring
→ helm uninstall prometheus-community
```

## Resources

- <https://cloudnative-pg.io/>
- <https://prometheus-operator.dev/>
