---
title: Daily commands
tags:
  - Kubernets
enableToc: true
---

# Recupera todos os recursos

```shell
kubectl get all -n marketplace
```

⚠️ Importante: kubectl get all não mostra tudo (ex: ConfigMaps, Secrets, Ingress).

Listar absolutamente tudo no namespace

```shell
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get -n <namespace> --ignore-not-found
```

Listar absolutamente tudo que tiver a label app=*

```shell
kubectl get $(
  kubectl api-resources --namespaced -o name \
  | paste -sd "," -
) -n admin -l app=admin-api-hub-app
```

# Get Pod

```shell
pod=$(kubectl get pod -n marketplace | grep dqvsn | grep -v NAME | awk '{print $1}')

pod=$(kubectl get pod -n nginx-system | grep ingress-nginx | grep -v NAME | awk '{print $1}') && kubectl logs -f $pod -n ngin-system
```

# Exec

```shell
pod=$(kubectl get pod -n marketplace | grep dqvsn | grep -v NAME | awk '{print $1}')

k exec -i -t -n marketing $pod -- sh -c "clear; (bash || ash || sh)"

# Especific container

k exec -i -t -n marketing $pod -c nginx-ctn -- sh -c "clear; (bash || ash || sh)
```

# Port forwarding

```bash
pod=$(kubectl get pod -n kafka | grep kafka-ui | grep -v NAME | awk '{print $1}') && \
kubectl port-forward $pod 8080:8080 -n kafka
```