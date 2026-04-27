---
title: Docker - Daily commands
description: >-
  Comandos úteis do dia a dia
enableToc: true
tags:
  - Docker
  - Commands
aliases:
---
---

## Containers

### Matando todos os containers

```shell
docker kill $(docker ps -q)
```

### Removendo todos os containers

```shell
docker rm $(docker ps -a -q)
```

## Obter mais detalhes do container

```shell
docker inspect redis-demo | jq
```

```shell
docker inspect redis-demo | jq '.[] | {Pid: .State.Pid, Running: .State.Running, Namespaces: .HostConfig.IpcMode}'
```

## Images

### Listando as imagens mais pesadas

```shell
docker images --format "{{.Repository}}:{{.Tag}} {{.Size}}" | sort -rh -k2 | head -n 10
```

### Removendo todas as imagens

```shell
docker rmi $(docker images -q)
```

## Volumes

### Listando os containers mais pesados

```shell
sudo du -ah /var/lib/docker | sort -rh | head -n 10
```