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

> _To be fixed later_

```shell
→ kubectl create namespace cnpg-monitoring

→ helm repo add prometheus-community \
    https://prometheus-community.github.io/helm-charts

→ helm upgrade --install \
    -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/docs/src/samples/monitoring/kube-stack-config.yaml \
    --set namespaceOverride=cnpg-monitoring \
    prometheus-community \
    prometheus-community/kube-prometheus-stack

→ kubectl get pods -l release=prometheus-community \
    -n cnpg-monitoring
NAME                                                      READY   STATUS    RESTARTS   AGE
prometheus-community-kube-operator-5d54b9cc5c-zzzqd       1/1     Running   0          69s
prometheus-community-kube-state-metrics-cd7c95449-plg2q   1/1     Running   0          69s

→ kubectl logs -f prometheus-prometheus-community-kube-prometheus-0 -c prometheus -n cnpg-monitoring
ts=2023-09-27T09:19:44.341Z caller=main.go:487 level=error msg="Error loading config (--config.file=/etc/prometheus/config_out/prometheus.env.yaml)" file=/etc/prometheus/config_out/prometheus.env.yaml err="parsing YAML file /etc/prometheus/config_out/prometheus.env.yaml: found multiple scrape configs with job name \"monitoring/alertmanager/0\""
```

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
