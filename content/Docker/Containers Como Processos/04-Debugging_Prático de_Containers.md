---
title: Debugging Prático de Containers
description: >
  Técnicas avançadas para diagnosticar e resolver problemas em containers usando ferramentas Linux padrão, mesmo quando o Docker daemon falha.
enableToc: true
tags:
  - Docker
  - Process
  - Linux
  - Containerd
  - Namespaces
  - Cgroups
  - Daemon
aliases:
---

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]  
**Anterior**: [[03-Manipulando_Containers_como_Processos_Normais|← Manipulação]]  
**Próximo**: [[05-Segurança_e_Isolamento_de_Containers|Segurança →]]

---

## Introdução

Quando Docker daemon falha ou não responde, você ainda pode investigar containers usando apenas ferramentas Linux padrão. Este artigo cobre casos reais de debugging.

---

## Cenário 1: Docker Daemon Não Responde

### Problema
```bash
$ docker ps
Error: Cannot connect to Docker daemon
```

### Solução: Encontrar Containers via Filesystem

```bash
# Diretório de containers do Docker
$ ls /var/lib/docker/containers/
a7f3c8d9e2f1b4a6c5d8e9f0a1b2c3d4e5f6a7b8
b8g4d9e2f1c5a6d7e8f9g0h1i2j3k4l5m6n7o8p9

# Cada diretório tem um container. Ver arquivo de configuração
$ cat /var/lib/docker/containers/a7f3c8d9e2f1/config.v2.json | jq '.Config.Image, .Config.Cmd, .State.Pid'

# Ou se souber o nome, procurar
$ grep -r "redis-demo" /var/lib/docker/containers/*/config.v2.json | head -1
/var/lib/docker/containers/a7f3c8d9e2f1/config.v2.json

# Extrair informações
$ jq '.State | {Pid, Running, Status}' /var/lib/docker/containers/a7f3c8d9e2f1/config.v2.json
{
  "Pid": 2112,
  "Running": true,
  "Status": "running"
}
```

### Diagnosticar o Docker Daemon

```bash
# Ver se o processo do Docker existe
$ ps aux | grep -E '[d]ockerd|[c]ontainerd'
root      1234  1.5  2.3 1234567 567890 ?  Sl  13:00  2:34 /usr/bin/dockerd
systemd+ 2345  0.8  1.2 890123  456789 ?  Ssl 13:01  1:45 /usr/bin/containerd

# Ver logs do systemd
$ sudo journalctl -u docker -n 100
$ sudo journalctl -u containerd -n 100

# Ver se está respondendo
$ sudo docker ps
# Se não responde, pode estar em deadlock

# Forçar restart (perigoso!)
$ sudo systemctl restart docker
$ sudo systemctl restart containerd
```

---

## Cenário 2: Inspecionar Recursos Sem Docker

### Monitorar Todos os Containers

```bash
# Listar todos os containers em execução (pelo filesystem)
$ for dir in /var/lib/docker/containers/*/; do
    container_id=$(basename "$dir")
    pid=$(jq '.State.Pid' "$dir/config.v2.json" 2>/dev/null)
    status=$(jq -r '.State.Status' "$dir/config.v2.json" 2>/dev/null)
    echo "Container: $container_id | PID: $pid | Status: $status"
  done

# Output
Container: a7f3c8d9e2f1... | PID: 2112 | Status: running
Container: b8g4d9e2f1c5... | PID: 3456 | Status: running
Container: c9h5e3g2d6f7... | PID: 4789 | Status: exited
```

### Diagnosticar Consumo de Recursos

```bash
# Todos os containers rodando e seu uso de memória
$ ps aux | grep -E '\[docker\]' | awk '{print $2, $6}' | while read pid mem; do
    if [ -f "/proc/$pid/cgroup" ]; then
      container=$(grep docker /proc/$pid/cgroup | head -1 | sed 's/.*://' | cut -d'/' -f3)
      echo "PID $pid ($container): $mem KB"
    fi
  done

# Ou mais simples
$ top -b -n1 | grep -E 'redis|nginx|app' | head -20
```

---

## Cenário 3: Container Travado/Não Responde

### Verificar Status Real

```bash
# Docker CLI diz "running", mas não responde
$ docker ps | grep app-service
4ab5c6d7 app:latest "npm start" 30 minutes Up 30 min app-service

# Verificar se o processo está vivo
$ PID=$(docker inspect -f '{{.State.Pid}}' app-service)
$ ps -p $PID
PID TTY      STAT   TIME COMMAND
2234 ?       S      0:12 node app.js

# Se não aparecer, o processo está morto mas Docker não sabe

# Ver status do processo
$ ps -o stat= -p 2234
S    # S = Sleep (normal)
R    # R = Running (normal)
Z    # Z = Zombie (morto! órfão)
T    # T = Stopped (pausado)
D    # D = Uninterruptible Sleep (travado em I/O)
```

### Investigar Travamento

```bash
# Se status é "D" (travado em I/O)
$ ps -ef | grep -E 'D.*app-service'
user      2234  2098  0 10:00 ?  D  0:12 node app.js

# Ver stack trace
$ cat /proc/2234/stack
[<ffffffff81234567>] io_schedule+0x123/0x456
[<ffffffff81345678>] __wait_on_freeing_inode+0x123/0x456

# Strace: ver últimas syscalls
$ sudo strace -p 2234 -e trace=read,write,open
# Mostrará calls bloqueadas

# Diagnóstico: provável I/O bound (disco lento, NFS, etc)
```

### Verificar Conexões de Rede

```bash
# Ver todas as conexões do container
$ sudo netstat -tlnp | grep 2234
tcp  0  0 0.0.0.0:8080  0.0.0.0:*  LISTEN  2234/node

# Ver conexões abertas
$ sudo ss -tlnp | grep 2234
LISTEN 0  128  0.0.0.0:8080  0.0.0.0:*  users:(("node",pid=2234,fd=10))

# Dados pendentes?
$ sudo netstat -tnp | grep 2234
tcp  0  512  192.168.1.10:8080  10.0.0.5:54321  ESTABLISHED 2234/node
# "512" no Recv-Q significa dados aguardando (backlog)

# Dropar conexões (último recurso)
$ sudo ss -K dst 10.0.0.5 dport 54321
```

---

## Cenário 4: Detecção de Vazamento de Memória

### Coletar Dados ao Longo do Tempo

```bash
# Script para monitorar crescimento de memória
#!/bin/bash
CONTAINER="app-service"
PID=$(docker inspect -f '{{.State.Pid}}' $CONTAINER)
OUTPUT="/tmp/memory_growth.log"

echo "Time,MemoryMB,ProcessCount" > $OUTPUT

for i in {1..288}; do  # 24 horas em 5-min intervals
  MEM_KB=$(cat /proc/$PID/status | grep VmRSS | awk '{print $2}')
  MEM_MB=$((MEM_KB / 1024))
  PROC_COUNT=$(ps --ppid $PID | wc -l)
  
  echo "$(date '+%Y-%m-%d %H:%M:%S'),$MEM_MB,$PROC_COUNT" >> $OUTPUT
  sleep 300
done

# Analisar
$ tail -20 /tmp/memory_growth.log
```

### Identificar Causa

```bash
# Se crescimento é lento e linear = vazamento de memória
# Se crescimento é em passos = cache normal (esperado)
# Se crescimento é exponencial = bug crítico

# Ver se há processos filhos crescentes
$ ps -ef --forest | grep -A 20 app-service
# Se muitos processos filhos = fork bomb ou thread leak

# Inspecionar específico
$ cat /proc/2234/status | grep -E 'Threads|VmPeak|VmSize'
Threads:	24          # Número de threads
VmPeak:	  512000 kB   # Pico de VM
VmSize:	  450000 kB   # VM atual

# Se Threads crescendo = thread leak
# Se VmSize > VmPeak = provável vazamento
```

---

## Cenário 5: Rastrear Chamadas de Sistema

### Instalação de Ferramentas

```bash
# Debian/Ubuntu
$ sudo apt install strace ltrace sysstat

# RHEL/CentOS
$ sudo yum install strace ltrace sysstat
```

### Rastrear Syscalls

```bash
# Rastrear todas as syscalls do container
$ sudo strace -p 2234 -f
# Mostrará em tempo real

# Rastrear apenas file operations
$ sudo strace -p 2234 -e trace=file
open("/data/file.db", O_RDONLY) = 3
read(3, "...", 4096) = 4096
close(3) = 0

# Rastrear apenas rede
$ sudo strace -p 2234 -e trace=network
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("db.local")}, 16) = 0

# Com timestamp
$ sudo strace -p 2234 -e trace=read,write -t -T
14:23:45.123456 read(3, "data", 4096) = 100 <0.000123>
14:23:45.123579 write(4, "result", 100) = 100 <0.000456>
```

### Análise de Syscalls

```bash
# Resumo: quantas syscalls de cada tipo
$ sudo strace -p 2234 -c
% time     seconds  usecs/call     calls    errors syscall
-------- ----------- ----------- --------- --------- ----------------
 30.45      0.123456       10  12345              read
 25.30      0.102345       12  8901               write
 15.20      0.061234       50  1234               poll
 ...

# Ver syscalls mais lentas
$ sudo strace -p 2234 -o /tmp/trace.log -T
# Depois analisar
$ grep '<' /tmp/trace.log | sort -t'<' -k2 -rn | head -20
```

---

## Cenário 6: Verificar Arquivo de Log

```bash
# Se container parou e você quer ver o que aconteceu
$ docker logs app-service
# Funciona mesmo se daemon não responde bem

# Se logs não estão disponíveis, verificar diretório
$ ls -la /var/lib/docker/containers/a7f3c8d9e2f1/
config.v2.json
hostname
hosts
a7f3c8d9e2f1-json.log  # ← Log bruto do container

# Ler log bruto
$ cat /var/lib/docker/containers/a7f3c8d9e2f1/*-json.log | jq '.'
{
  "log": "Starting application...\n",
  "stream": "stdout",
  "time": "2024-04-26T14:23:45.000000000Z"
}
```

---

## Script Prático: Diagnóstico Completo

```bash
#!/bin/bash
# diagnostic.sh - Diagnóstico completo de container

CONTAINER_NAME=$1
if [ -z "$CONTAINER_NAME" ]; then
  echo "Usage: $0 <container_name>"
  exit 1
fi

# Obter PID
PID=$(docker inspect -f '{{.State.Pid}}' $CONTAINER_NAME 2>/dev/null)

if [ -z "$PID" ] || ! ps -p $PID > /dev/null 2>&1; then
  # Docker CLI falhou, procurar no filesystem
  echo "[!] Docker CLI não respondeu. Procurando no filesystem..."
  CONFIG=$(grep -l "$CONTAINER_NAME" /var/lib/docker/containers/*/config.v2.json | head -1)
  PID=$(jq '.State.Pid' "$CONFIG")
fi

echo "=== Diagnóstico para $CONTAINER_NAME (PID: $PID) ==="
echo

echo "[1] Status do Processo"
ps -o pid,stat,user,%cpu,%mem,etime,command -p $PID

echo
echo "[2] Uso de Memória"
cat /proc/$PID/status | grep -E "VmPeak|VmSize|VmRSS|VmSwap"

echo
echo "[3] Uso de CPU (últimos 10 segundos)"
pidstat -p $PID 1 10 2>/dev/null || echo "pidstat não disponível"

echo
echo "[4] Conexões de Rede"
netstat -tnp 2>/dev/null | grep $PID || ss -tnp | grep $PID

echo
echo "[5] File Descriptors"
ls -la /proc/$PID/fd/ | wc -l && echo "FDs abertos"

echo
echo "[6] Últimas 5 linhas de log"
docker logs --tail=5 $CONTAINER_NAME 2>/dev/null || \
  tail -5 /var/lib/docker/containers/*/config.v2.json 2>/dev/null | \
  jq -r '.log' | tail -5

echo
echo "[7] Namespaces (isolamento)"
ls -l /proc/$PID/ns/ | awk '{print $NF}' | grep -oP '\d+' | sort | uniq

echo "=== Fim do diagnóstico ==="
```

**Uso**:
```bash
$ chmod +x diagnostic.sh
$ ./diagnostic.sh redis-demo
=== Diagnóstico para redis-demo (PID: 2112) ===
[1] Status do Processo
PID  STAT USER   %CPU %MEM  ETIME COMMAND
2112 Ssl  redis  1.2  0.5  10:32 redis-server *:6379
...
```

---

## Resumo

✅ Encontrar PIDs de containers sem Docker CLI  
✅ Monitorar recursos quando daemon falha  
✅ Diagnosticar travamentos (D state, I/O bloqueado)  
✅ Detectar vazamento de memória  
✅ Rastrear syscalls com strace  
✅ Acessar logs do container diretamente  

---

## Próximos Passos

Agora que você sabe como **debugar**, é hora de entender **segurança** — o que está isolado e o que não está.

**Próximo**: [[05-Segurança_e_Isolamento_de_Containers|Segurança →]]

---

## Referências

- [strace Manual](https://man7.org/linux/man-pages/man1/strace.1.html)
- [netstat/ss Manual](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Linux /proc Manual](https://man7.org/linux/man-pages/man5/proc.5.html)

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]
