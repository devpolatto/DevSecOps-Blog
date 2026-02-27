---
title: Estouro de memória no cgroup
description: "O que é um Memory cgroup out of memory (estouro de memória no cgroup), como ele ocorre e como lidar com ele em um ambiente Kubernetes."
tags:
  - Kubernetes
enableToc: true
---

# Resumo do problema

Erro capturado:

```shell
# Memory cgroup out of memory: Killed process 2757828 (java)
total-vm:2285684kB
anon-rss:96504kB
file-rss:19332kB
oom_score_adj:998
```

👉 Em Kubernetes, isso normalmente resulta em:

- OOMKilled
- Pod reiniciando
- CrashLoopBackOff (em alguns cenários)

# Analise do problema

O erro "Memory cgroup out of memory: Killed process 2757828 (java)" indica que um processo Java foi encerrado pelo sistema operacional devido a um estouro de memória no cgroup. O cgroup é uma funcionalidade do Linux que limita e isola o uso de recursos, como CPU e memória, para grupos de processos. Quando um processo excede o limite de memória definido para o cgroup, ele é forçado a ser encerrado para evitar que o sistema fique instável.

O Linux usa cgroups para import limites de recurso. No Kubernetes, cada container roda dentro de um cgroup com o limite definido em:

```yaml
resources:
  limits:
    memory: "512Mi"
```

Quando esse limite é ultrapassado:

- O kernel mata o processo mais "caro", ou seja, o processo que mais consome memória, nesse caso, o processo Java.
- O sistema registra o evento de OOM (Out of Memory) e o processo é encerrado para liberar memória.

Trecho importante:

```shell
total-vm:2285684kB
anon-rss:96504kB
```

- `total-vm`: Total de memória virtual usada pelo processo.
- `anon-rss`: Memória residente anônima, ou seja, a memória que o processo realmente está usando.
- O cgroup considera mais do que apenas heap visivel, incluindo memória nativa, bibliotecas, etc.
- JVM usa memória fora do heap:
     - Metaspace
     - Threads (stack)
     - Direct ByteBuffers
     - JIT / Code Cache
     - GC structures

Resultado: o container estourou o limite, mesmo que o heap “parecesse ok”.

```shell
oom_score_adj:998
```

- O `oom_score_adj` é um valor que o kernel usa para determinar a prioridade de matar processos em caso de OOM. Um valor alto (próximo de 1000) indica que o processo tem alta prioridade para ser morto.
- Containers geralmente rodam com oom_score_adj alto para garantir que eles sejam os primeiros a serem mortos em caso de OOM, protegendo o host e outros processos críticos.

## Como investigar no Kubernetes

1. Verifique os eventos do pod:

```shell
kubectl describe pod <pod-name> -n <namespace>
```

**Investigação interrompida**