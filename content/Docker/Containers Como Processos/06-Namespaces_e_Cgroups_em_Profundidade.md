---
title: Namespaces e Cgroups em Profundidade
description: >
  Uma análise detalhada dos namespaces e cgroups, os mecanismos de isolamento e limitação que tornam a containerização possível.
enableToc: true
tags:
  - Docker
  - Process
  - Linux
  - Containerd
  - Namespaces
  - Cgroups
  - Isolation
aliases:
---

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]  
**Anterior**: [[05-Segurança_e_Isolamento_de_Containers|← Segurança]]

---

## Arquitetura Completa: Docker ao Kernel

```
┌────────────────────────────────────────────┐
│         Docker CLI / API                   │
│   docker run, docker ps, docker inspect    │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│         Docker Daemon (dockerd)            │
│  - Gerencia imagens, volumes, networks     │
│  - Comunica com containerd via gRPC        │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│      containerd (container runtime)        │
│  - Gerencia lifecycle de containers        │
│  - Orquestra OCI runtime (runc)            │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│  containerd-shim-runc-v2 (runtime shim)   │
│  - Camada entre containerd e runc          │
│  - Keeps container alive se daemon morre   │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│        runc (OCI runtime spec)             │
│  - Executa o container                     │
│  - Aplica namespaces e cgroups             │
│  - Faz fork/exec do processo               │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│      Linux Kernel                          │
│  ├─ Namespaces (PID, Network, Mount, etc) │
│  ├─ Cgroups (resource limiting)           │
│  ├─ Syscalls (fork, exec, mmap, etc)      │
│  └─ Security (DAC, MAC, Seccomp)          │
└────────────────────────────────────────────┘
```

---

## Namespaces: Os 7 Tipos

### 1. PID Namespace

Isola a árvore de processos.

```bash
# No host (PID namespace padrão)
$ ps aux
PID  TTY  COMMAND
1    ?    /sbin/init
...
2112 ?    redis-server
2345 ?    nginx
3456 ?    mysql

# Dentro do container (PID namespace isolado)
$ docker exec redis-demo ps aux
PID  TTY  COMMAND
1    ?    redis-server  ← Vê como PID 1 (não 2112)
```

**O que oferece**:
- Cada container vê si mesmo como PID 1
- Não vê processos do host ou outros containers
- Sinais (SIGTERM) apenas dentro do namespace

**Comando para criar**:
```bash
$ unshare -p -f --mount-proc /bin/bash
# Novo namespace PID onde /bin/bash é PID 1
```

---

### 2. Network Namespace

Isola stack de rede: interfaces, IPs, routing.

```bash
# No host
$ ifconfig
eth0: 192.168.1.10
lo: 127.0.0.1

# Dentro do container
$ docker exec redis-demo ifconfig
eth0: 172.17.0.2  ← IP diferente
lo: 127.0.0.1

# Routing table diferente
$ docker exec redis-demo route -n
Kernel IP routing table
Destination  Gateway      Genmask      Iface
172.17.0.0   0.0.0.0      255.255.0.0  eth0  ← Diferente do host
```

**O que oferece**:
- Interface de rede própria (virtual)
- IP address próprio
- Routing table própria
- Firewall rules independentes (iptables)

**Exemplo prático**:
```bash
# Container 1: Redis
$ docker run --name c1 -d redis:7-alpine
# IP: 172.17.0.2

# Container 2: Web App
$ docker run --name c2 -d app:latest
# IP: 172.17.0.3

# c2 pode acessar c1 em 172.17.0.2:6379
# c1 não vê IP do host (192.168.1.10)
```

---

### 3. Mount Namespace

Isola filesystem: `/` é diferente.

```bash
# No host
$ mount
/dev/sda1 on / type ext4
/dev/sda2 on /home type ext4
/proc on /proc type proc
...

# Dentro do container
$ docker exec redis-demo mount
overlay on / type overlay  ← Diferente! overlay filesystem
/proc on /proc type proc
/sys on /sys type sysfs
/dev on /dev type devtmpfs

# Filesystem root
$ cat /etc/hostname  # host
myhost

$ docker exec redis-demo cat /etc/hostname
a7f3c8d9e2f1  # container
```

**O que oferece**:
- Root filesystem isolado (via overlay)
- Mounts próprios (`/proc`, `/sys`, `/dev`)
- Não vê mounts do host (a menos que bindados)

**Estrutura do overlay filesystem**:
```
/var/lib/docker/overlay2/[CONTAINER_ID]/
├── lower-dir  (imagem base)
├── upper-dir  (mudanças do container)
├── work-dir   (staging area)
└── merged     (view unificado — raiz do container)
```

---

### 4. UTS Namespace

Isola hostname e domainname.

```bash
# No host
$ hostname
my-laptop

# Dentro do container
$ docker exec redis-demo hostname
a7f3c8d9e2f1  # Container ID curto

# Isso está em um namespace UTS separado
$ cat /proc/1/ns/uts
uts:[4026531838]  # Host

$ docker inspect -f '{{.State.Pid}}' redis-demo | \
  xargs -I {} cat /proc/{}/ns/uts
uts:[4026532719]  # Container — DIFERENTE
```

**O que oferece**:
- Hostname próprio
- Domainname próprio
- Não afeta hostname do host

---

### 5. IPC Namespace

Isola Inter-Process Communication.

```bash
# No host
$ ipcs -m
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattach    status
0x00000000 0          user       600        65536      0

# Dentro do container
$ docker exec redis-demo ipcs -m
# Diferente! Shared memory próprio

# POSIX message queues
$ ls /dev/mqueue
# Host e container têm diferentes

# Unix sockets
$ docker exec redis-demo ls /var/run/
# Diferentes sockets que o host
```

**O que oferece**:
- Shared memory isolado
- Message queues isoladas
- POSIX semaphores isolados
- Unix domain sockets isolados

---

### 6. User Namespace

Isola UID/GID.

```bash
# No host
$ id
uid=1000(user) gid=1000(user)

# Dentro do container (default)
$ docker exec redis-demo id
uid=0(root) gid=0(root)  ← Vê como root no container

# No host, ainda é UID não-zero
$ ps aux | grep redis
systemd+ 2112  ...  redis-server  ← UID ~100000 (remetido)

# User namespace remapping
$ docker run --userns-remap=default redis:7-alpine
```

**Remapping**:
```
Container       Host
UID 0      →    UID 100000
UID 1      →    UID 100001
...
UID 65535  →    UID 165535
```

**O que oferece**:
- Separação de UID/GID
- Remapping para non-root
- Reduz risco de privilege escalation

---

### 7. Cgroup Namespace

Isola visão de cgroups.

```bash
# No host
$ cat /proc/self/cgroup
12:cpuset:/user.slice
11:cpuset:/system.slice
10:memory:/user.slice

# Dentro do container
$ docker exec redis-demo cat /proc/self/cgroup
12:cpuset:/docker/a7f3c8d9e2f1
11:cpuset:/docker/a7f3c8d9e2f1
10:memory:/docker/a7f3c8d9e2f1

# O container vê sua própria raiz de cgroup
```

**O que oferece**:
- Cada container vê cgroups como raiz `/`
- Não vê hierarquia completa do host
- Previne escapes via cgroup manipulation

---

## Cgroups: Control Groups

### O que são Cgroups?

Mecanismo para limitar e monitorar recursos:

```
Cgroup
├─ Memória (memory)
│  ├─ memory.limit_in_bytes
│  ├─ memory.usage_in_bytes
│  └─ memory.max_usage_in_bytes
│
├─ CPU (cpuset, cpu)
│  ├─ cpuset.cpus
│  ├─ cpuset.mems
│  ├─ cpu.cfs_quota_us
│  └─ cpu.cfs_period_us
│
├─ I/O (blkio)
│  ├─ blkio.weight
│  ├─ blkio.throttle.read_bps_device
│  └─ blkio.throttle.write_bps_device
│
└─ Network (net_cls)
   ├─ net_cls.classid
   └─ Qdisc rules no host
```

### Cgroups v2 (cgroupsV2)

Versão mais recente, unificada:

```bash
# Verificar versão
$ mount -t cgroup2 | head
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)

# Estrutura unificada
$ cat /sys/fs/cgroup/docker.slice/memory.max
1073741824  # 1GB

$ cat /sys/fs/cgroup/docker.slice/memory.current
52428800    # 50MB

$ cat /sys/fs/cgroup/docker.slice/cpu.max
100000 100000  # 1 CPU (100ms out of 100ms)
```

### Limites Práticos

```bash
# Criar container com limites
$ docker run \
  -m 512m \           # 512MB memória
  --memory-swap 1g \  # 512MB RAM + 512MB swap
  --cpus 2 \          # 2 CPUs
  --cpuset-cpus 0,1 \ # CPUs específicas
  -it ubuntu:latest

# Verificar
$ docker inspect <id> | jq '.HostConfig | {Memory, MemorySwap, CpuQuota, CpuPeriod, CpusetCpus}'
{
  "Memory": 536870912,        # 512MB
  "MemorySwap": 1073741824,   # 1GB (512 + 512 swap)
  "CpuQuota": 200000,         # 2 CPUs
  "CpuPeriod": 100000,
  "CpusetCpus": "0,1"
}
```

### Memory Cgroup

```bash
# Limite hard (container é killed se ultrapassar)
/sys/fs/cgroup/memory/docker/[CID]/memory.limit_in_bytes

# Soft limit (aviso, sem kill)
/sys/fs/cgroup/memory/docker/[CID]/memory.soft_limit_in_bytes

# Uso atual
/sys/fs/cgroup/memory/docker/[CID]/memory.usage_in_bytes

# Pico atingido
/sys/fs/cgroup/memory/docker/[CID]/memory.max_usage_in_bytes

# Exemplos:
$ cat /sys/fs/cgroup/docker/*/memory.limit_in_bytes | sort -n
52428800      # 50MB
536870912     # 512MB
1073741824    # 1GB (unlimited se -1)
```

---

## Interação: Namespaces + Cgroups

```
Container Redis
├─ Namespaces (ISOLAMENTO)
│  ├─ PID: Vê si mesmo como PID 1
│  ├─ Network: IP 172.17.0.2
│  ├─ Mount: Root é overlay filesystem
│  ├─ User: Vê como UID 0, mas remetido para 100000
│  ├─ UTS: Hostname a7f3c8d9e2f1
│  ├─ IPC: Message queues próprias
│  └─ Cgroup: Vê cgroups como raiz /
│
└─ Cgroups (LIMITAÇÃO)
   ├─ Memory: Máximo 1GB
   ├─ CPU: Máximo 2 cores
   ├─ I/O: 10MB/s read
   └─ Network: Limitado por iptables no host
```

---

## Criar Namespaces Manualmente (Avançado)

### Experimento: Criar Container Manual com unshare

```bash
#!/bin/bash
# script: create-container-manual.sh

# 1. Criar filesystem isolado (usando chroot)
CONTAINER_ROOT="/tmp/mycontainer"
mkdir -p $CONTAINER_ROOT

# 2. Populá-lo (simplificado, apenas Alpine)
# Em produção, usaria debootstrap ou container image
cd $CONTAINER_ROOT

# 3. Criar namespaces isolados
unshare -p -f -m -n -i -u \
  --mount-proc \
  /bin/bash <<SHELL

# Agora estamos em namespaces isolados!
echo "PID: $$"  # Será 1 (raiz do namespace)

# Montar filesystems
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t tmpfs tmpfs /tmp

# Network isolada (sem configuração)
# Processo é completamente isolado

# Ver que está isolado
ps aux  # Apenas nossos processos
ifconfig  # Nenhuma interface
hostname  # Padrão

SHELL
```

### Usar runc Diretamente (Production-like)

```bash
# Spec OCI em JSON
cat > spec.json <<'EOF'
{
  "ociVersion": "1.0.0",
  "process": {
    "terminal": false,
    "user": {
      "uid": 0,
      "gid": 0
    },
    "args": [
      "redis-server"
    ],
    "env": [
      "PATH=/usr/local/bin:/usr/bin"
    ],
    "cwd": "/"
  },
  "root": {
    "path": "/tmp/rootfs",
    "readonly": false
  },
  "hostname": "redis",
  "mounts": [
    {
      "destination": "/proc",
      "type": "proc"
    },
    {
      "destination": "/sys",
      "type": "sysfs"
    }
  ],
  "hooks": {},
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "network" },
      { "type": "ipc" },
      { "type": "uts" },
      { "type": "mount" }
    ],
    "devices": [
      {
        "path": "/dev/null",
        "type": "c",
        "major": 1,
        "minor": 3
      }
    ],
    "resources": {
      "memory": {
        "limit": 1073741824
      },
      "cpu": {
        "quota": 200000,
        "period": 100000
      }
    }
  }
}
EOF

# Executar
$ runc run mycontainer
```

---

## Monitoramento em Profundidade

### Namespace Monitoring

```bash
# Ver todos os namespaces
$ for ns in cgroup ipc mnt net pid uts user; do
    echo "=== $ns namespaces ==="
    find /proc -name $ns | sort | uniq -c | sort -rn
  done

# Output
=== pid namespaces ===
  1 /proc/1/ns/pid
  1 /proc/2112/ns/pid (container)
  1 /proc/3456/ns/pid (container)

# Ver hierarquia
$ ls -la /proc/*/ns/ | grep docker
```

### Cgroup Monitoring

```bash
# Ver hierarchia de cgroups
$ systemd-cgls
Control group /:
└─docker-slice
  ├─1234.scope (redis)
  ├─5678.scope (nginx)
  └─9012.scope (mysql)

# Uso por container
$ for cid in $(docker ps -q); do
    echo "Container: $(docker inspect -f '{{.Name}}' $cid)"
    echo "  Memory: $(cat /sys/fs/cgroup/memory/docker/$cid/memory.usage_in_bytes | numfmt --to=iec)"
    echo "  CPU: $(cat /sys/fs/cgroup/cpu/docker/$cid/cpuacct.usage | numfmt --to=iec)"
  done
```

---

## Resumo de Isolamento

```
Namespace    Isola                        Caso de Uso
──────────────────────────────────────────────────────
PID          Processos                    Cada container vê-se como PID 1
Network      Rede (interfaces, IPs)       Cada container tem IP próprio
Mount        Filesystem                   Cada container tem root diferente
UTS          Hostname, domainname         Identificação do container
IPC          Shared memory, message q.    Isolação de IPC
User         UID/GID                      Reduzir escalation risk
Cgroup       Visão de cgroups             Esconder limites do container

Cgroup       Controla                     Caso de Uso
──────────────────────────────────────────────────────
Memory       RAM máxima                   Evitar OOM killer
CPU          Cores e tempo                Isolação de performance
Blkio        I/O de disco                 Isolação de Storage
Network      Bandwidth (iptables)         QoS de rede
```

---

## Conclusão

Namespaces e Cgroups são os pilares da containerização. Compreendê-los é essencial para:

- ✅ Debugar containers
- ✅ Otimizar performance
- ✅ Implementar segurança
- ✅ Criar custom runtimes
- ✅ Entender limites e trade-offs

---

## Próximos Passos

Você completou a série! Você agora entende:

1. ✅ Containers são processos Linux
2. ✅ Como inspecionálos
3. ✅ Como manipulá-los
4. ✅ Como debugá-los
5. ✅ Implicações de segurança
6. ✅ Como funcionam internamente

---

## Referências Finais

- [Linux Namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Cgroups v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec)
- [runc Source Code](https://github.com/opencontainers/runc)
- [containerd Architecture](https://containerd.io/docs/)
- [Docker Architecture](https://docs.docker.com/get-started/docker-overview/)

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]

---

## 🎉 Parabéns!

Você terminou a série **"Containers São Apenas Processos Linux"**. Agora você tem conhecimento profundo e prático sobre como containers realmente funcionam!
