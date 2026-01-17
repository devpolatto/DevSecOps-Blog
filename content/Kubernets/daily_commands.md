---
title: Daily commands
tags:
  - Kubernets
enableToc: true
---
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