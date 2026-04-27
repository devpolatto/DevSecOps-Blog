---
title: Manipulando Containers Como Processos Normais
description: >
  Aprenda a controlar containers usando sinais Linux padrão, monitorar recursos via cgroups e diagnosticar problemas sem depender do Docker CLI.
enableToc: true
tags:
  - Docker
  - Process
  - Linux
  - Containerd
  - Namespaces
  - Cgroups
aliases:
---

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]  
**Anterior**: [[02-Inspecionando_Containers_com_proc|← Inspeção]]  
**Próximo**: [[04-Debugging_Prático de_Containers|Debugging Prático →]]

---

## Introdução

Porque containers são processos Linux ordinários, você pode controlá-los usando ferramentas Linux padrão. Não precisa sempre do Docker CLI — você pode:

- Enviar sinais (SIGTERM, SIGKILL)
- Monitorar recursos (CPU, memória)
- Verificar file descriptors
- Inspecionar cgroups

---

## Enviando Sinais Diretamente

### Teste 1: SIGTERM (Graceful Shutdown)

```bash
# Verificar containers em execução
$ docker ps
CONTAINER ID  IMAGE        COMMAND                 CREATED       STATUS      PORTS      NAMES
a7f3c8d9e2f1 redis:7-alp  "redis-server *:6379"  5 minutes ago Up 5 min 6379/tcp   redis-demo

# Obter PID do container
$ ps -fC redis-server
UID        PID  PPID  C STIME TTY      STAT   TIME CMD
systemd+  2112  2098  0 14:23 ?        Ssl    0:00 redis-server *:6379

# Enviar SIGTERM (permite shutdown gracioso)
$ sudo kill -SIGTERM 2112
# ou
$ sudo kill -15 2112

# Verificar status no Docker
$ docker ps
# redis-demo não aparecerá mais

# Ver logs do container (mostrará shutdown)
$ docker logs redis-demo
...
oO0OoO0OoO0Oo Redis is shutting down
```

**O que acontece**:
1. `kill -SIGTERM` envia sinal 15 para o processo
2. Redis recebe o sinal e inicia shutdown gracioso
3. Fechar conexões, salvar estado, liberar recursos
4. Processo sai com código 0 (sucesso) ou 143 (SIGTERM)

### Teste 2: SIGKILL (Force Kill)

```bash
# Iniciar novo container
$ docker run --name redis-demo2 -d redis:7-alpine
b8g4d9e2f1c5a6d7e8f9g0h1i2j3k4l5m6n7o8p9

# Obter PID
$ docker inspect -f '{{.State.Pid}}' redis-demo2
3456

# Enviar SIGKILL (kill imediato)
$ sudo kill -9 3456
# ou
$ sudo kill -SIGKILL 3456

# Verificar status imediatamente
$ docker ps | grep redis-demo2
# Aparecerá como "Exited (137)"

# Ver código de saída completo
$ docker inspect redis-demo2 | jq '.[] | .State'
{
  "Status": "exited",
  "ExitCode": 137,
  "Error": "",
  "StartedAt": "2024-04-26T14:28:00.000000000Z",
  "FinishedAt": "2024-04-26T14:28:05.000000000Z"
}
```

**Código de saída 137**:
- 128 + 9 (SIGKILL) = 137
- Indica que o processo foi terminado por SIGKILL

### Tabela: Sinais Comuns

| Sinal | Número | Flag | Efeito | Uso |
|-------|--------|------|--------|-----|
| SIGTERM | 15 | `-SIGTERM` ou `-15` | Shutdown gracioso | Parar containers normalmente |
| SIGKILL | 9 | `-SIGKILL` ou `-9` | Força imediata | Último recurso |
| SIGSTOP | 19 | `-SIGSTOP` ou `-19` | Pausa (não killable) | Freezar container |
| SIGCONT | 18 | `-SIGCONT` ou `-18` | Continua | Descongelar container |
| SIGHUP | 1 | `-SIGHUP` ou `-1` | Reload | Recarregar configuração |

---

## Códigos de Saída

### Interpretação

```bash
# Ver código de saída de um container parado
$ docker inspect <container> | jq '.[] | .State.ExitCode'

# Código 0-127: Saída normal do programa
0   = Sucesso
1   = Erro genérico
2   = Erro de uso
127 = Comando não encontrado

# Código 128+N: Saído por sinal N
137 = 128 + 9   (SIGKILL)
143 = 128 + 15  (SIGTERM)
148 = 128 + 20  (SIGSTOP, se não for pausado)
```

### Exemplo Prático

```bash
# Container terminado com SIGTERM
$ docker run --name test -d redis:7-alpine
# ... deixar rodar um pouco ...
$ sudo kill -15 $(docker inspect -f '{{.State.Pid}}' test)
$ docker inspect test | jq '.[] | .State.ExitCode'
143  # = SIGTERM

# Container terminado com SIGKILL
$ docker run --name test2 -d redis:7-alpine
$ sudo kill -9 $(docker inspect -f '{{.State.Pid}}' test2)
$ docker inspect test2 | jq '.[] | .State.ExitCode'
137  # = SIGKILL
```

---

## Pausar e Resumir Containers

### Usando Sinais de Pause/Continue

```bash
# Pausar container (sem killing)
$ sudo kill -SIGSTOP $(docker inspect -f '{{.State.Pid}}' redis-demo)

# Verificar — container está "paused" do ponto de vista do sistema
$ ps aux | grep redis
# O processo aparecerá com "T" (Stopped)

# Resumir container
$ sudo kill -SIGCONT $(docker inspect -f '{{.State.Pid}}' redis-demo)

# Volta ao normal
```

⚠️ **Aviso**: Usar SIGSTOP diretamente bypassa o Docker, pode deixar o container em estado inconsistente. Use `docker pause` para pausar propriamente.

---

## Inspecionando Cgroups (Control Groups)

### Ver Limites do Container

```bash
# Obter CGroup ID do container
$ docker inspect redis-demo | jq '.[] | .HostConfig | {Memory, MemorySwap, CpuQuota, CpuPeriod, Cpuset}'
{
  "Memory": 1073741824,     # 1GB
  "MemorySwap": -1,          # Unlimited
  "CpuQuota": -1,            # Unlimited
  "CpuPeriod": 100000,       # 100ms
  "Cpuset": null             # Pode usar qualquer CPU
}

# CGroup path
$ cat /proc/2112/cgroup
12:cpuset:/docker/a7f3c8d9e2f1b4a6c5d8e9f0a1b2c3d4e5f6a7b8
11:memory:/docker/a7f3c8d9e2f1b4a6c5d8e9f0a1b2c3d4e5f6a7b8
...
```

### Ler Limites Diretamente

```bash
# Container com limite de 1GB
$ docker run --memory=1g --name limited -d redis:7-alpine

# Obter limites
$ docker inspect limited | jq '.[] | .HostConfig.Memory'
1073741824  # bytes = 1GB

# Ver no cgroup
$ cat /sys/fs/cgroup/docker/*/memory.limit_in_bytes | grep -B5 1073741824
```

### Inspecionar Uso Atual

```bash
# Uso atual de memória
$ cat /sys/fs/cgroup/docker/a7f3c8d9e2f1/memory.usage_in_bytes
52428800  # ~50MB

# Uso máximo atingido
$ cat /sys/fs/cgroup/docker/a7f3c8d9e2f1/memory.max_usage_in_bytes
61865984  # ~59MB

# Limite hard
$ cat /sys/fs/cgroup/docker/a7f3c8d9e2f1/memory.limit_in_bytes
1073741824  # 1GB

# Cálculo de uso
$ echo "scale=2; $(cat /sys/fs/cgroup/docker/a7f3c8d9e2f1/memory.usage_in_bytes) / 1024 / 1024 MB" | bc
50.00 MB
```

---

## Monitorar Recursos em Tempo Real

### CPU Usage

```bash
# Ver CPU do container em tempo real (requer sysstat)
$ sudo pidstat -p 2112 1 5
Linux 5.15.0 (hostname) 26/04/24 14h23m30s
UID  PID %usr %system %guest %CPU CPU minflt majflt
root 2112 1.20  0.40   0.00  1.60  1    450     12
root 2112 1.10  0.35   0.00  1.45  1    460     10

# Explicar:
#   %usr = Tempo em user space (aplicação)
#   %system = Tempo em kernel space (syscalls)
#   %CPU = Total
#   CPU = Qual processador

# Ou usar top
$ top -p 2112
PID USER      PR  NI    VIRT    RES SHR S %CPU %MEM    TIME+ COMMAND
2112 redis    20   0  174.1m   50m 4.2m S  1.5  0.6   0:15.32 redis-server
```

### Memória

```bash
# Ver memória do container
$ cat /proc/2112/status | grep -E "VmPeak|VmSize|VmRSS|VmSwap"
VmPeak:	  178792 kB  # Pico de memória virtual
VmSize:	  174184 kB  # Memória virtual atual
VmRSS:	   51000 kB  # Resident Set (RAM física)
VmSwap:	       0 kB  # Swap usado

# Watch em tempo real
$ watch -n 1 "cat /proc/2112/status | grep -E 'VmRSS|VmSwap'"
Every 1.0s: cat /proc/2112/status | grep -E 'VmRSS|VmSwap'
VmRSS:	   51000 kB
VmSwap:	       0 kB
```

### I/O

```bash
# Ver estatísticas de I/O do container
$ cat /proc/2112/io
rchar: 1024000          # Bytes lidos (total)
wchar: 512000           # Bytes escritos (total)
syscr: 5000             # Syscalls read
syscw: 3000             # Syscalls write
read_bytes: 4096000     # Bytes lidos do disco
write_bytes: 2048000    # Bytes escritos para disco
cancelled_write_bytes: 0

# Monitorar I/O em tempo real
$ watch -n 1 "cat /proc/2112/io | grep -E 'read_bytes|write_bytes'"
```

---

## Casos de Uso Prático

### Cenário 1: Docker Está Travado

```bash
# Docker daemon não responde, mas você precisa parar um container
PID=$(docker inspect -f '{{.State.Pid}}' app-service)
echo "Parando container com PID $PID"
sudo kill -SIGTERM $PID

# Aguardar 10 segundos para shutdown gracioso
sleep 10

# Se ainda estiver rodando, force
if ps -p $PID > /dev/null; then
  sudo kill -9 $PID
fi
```

### Cenário 2: Diagnosticar Vazamento de Memória

```bash
# Container cresce indefinidamente
docker run --memory=512m --name debug -d app:latest

# Monitorar memória a cada 5 segundos
for i in {1..100}; do
  MEM=$(cat /proc/$(docker inspect -f '{{.State.Pid}}' debug)/status | grep VmRSS | awk '{print $2}')
  echo "$(date '+%H:%M:%S') - Memory: $MEM KB"
  sleep 5
done

# Output mostrará crescimento (ou não) ao longo do tempo
```

### Cenário 3: Limitar Recursos em Tempo Real

```bash
# Container já rodando e consumindo muita memória
PID=$(docker inspect -f '{{.State.Pid}}' hungry-app)

# Ver uso atual
$ cat /sys/fs/cgroup/docker/*/memory.usage_in_bytes | head -1
2000000000  # 2GB - Muito!

# Não é possível mudar limite sem docker CLI, mas você pode:
# 1. Graciously shutdown
$ sudo kill -SIGTERM $PID

# 2. Ou restarter com novo limite
$ docker run --memory=1g hungry-app:latest
```

---

## ⚠️ Observações Importantes

### Docker CLI vs Kill Direto

| Aspecto | docker stop | kill -SIGTERM | kill -9 |
|---------|-------------|---------------|---------|
| Graceful? | ✅ Sim | ✅ Sim | ❌ Não |
| Timeout? | ✅ 10s default | ❌ Não | ❌ Imediato |
| Clean | ✅ Limpo | ✅ Limpo | ⚠️ Abrupto |
| Docker aware? | ✅ Sim | ❌ Não | ❌ Não |
| Código de saída | 0 | 143 | 137 |

**Recomendação**: Use `docker stop` em produção. Use `kill` apenas para debugging.

---

## Resumo

✅ Containers podem ser parados com sinais (SIGTERM, SIGKILL)  
✅ Cgroups controla limites de recursos  
✅ `/proc` mostra uso real em tempo real  
✅ Sem Docker CLI, você consegue diagnosticar tudo  
✅ SIGTERM = graceful, SIGKILL = force  

---

## Próximos Passos

Até agora aprendemos a **inspecionar** e **manipular** containers. Agora vamos explorar casos **avançados de debugging** quando o Docker daemon falha completamente.

**Próximo**: [[04-Debugging_Prático de_Containers|Debugging Prático →]]

---

## Referências

- [Linux Signals](https://man7.org/linux/man-pages/man7/signal.7.html)
- [Cgroups Memory Control](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory)
- [Docker Exit Codes](https://docs.docker.com/engine/reference/run/#exit-status)

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]
