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
