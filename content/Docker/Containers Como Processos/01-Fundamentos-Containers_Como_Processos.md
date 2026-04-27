---
title: "Fundamentos: Containers Como Processos"
description: >
  Uma exploração profunda e prática sobre como containers funcionam no kernel Linux, desmistificando a abstração que o Docker cria.
enableToc: true
tags:
  - Docker
  - Process
  - Linux
  - Containerd
  - Namespaces
  - Cgroups
  - proc
aliases:
---

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]  
**Próximo**: [[02-Inspecionando_Containers_com_proc|Inspecionando Containers →]]

---

## Introdução: Desmistificando a Containerização

A abstração de container criada por ferramentas como Docker, Podman e containerd é construída sobre primitivas que o kernel Linux já fornece há anos. A ilusão de isolamento completo é criada por dois mecanismos fundamentais:

- **Namespaces**: Limitam o que um processo **pode ver** (PIDs, filesystems, rede, usuários)
- **Cgroups**: Limitam o que um processo **pode usar** (CPU, memória, I/O bandwidth)

Juntos, esses mecanismos dão ao processo a ilusão de estar rodando sozinho no sistema, quando na verdade ele é apenas mais um processo sob o mesmo kernel.

**Premissa central**: Containers NÃO são máquinas virtuais, NÃO são objetos especiais do kernel. São processos Linux ordinários, nada mais.

---

## Comprovando: Containers São Realmente Processos

### Setup: Verificar Processos Antes e Depois

```bash
# Listar processos redis-server no host (deve estar vazio)
$ ps -fC redis-server
UID        PID  PPID  C STIME TTY      STAT   TIME CMD

# Nada! Vamos começar um container
$ docker run --name redis-demo -d redis:7-alpine
a7f3c8d9e2f1b4a6c5d8e9f0a1b2c3d4e5f6a7b8

# Agora verifique novamente
$ ps -fC redis-server
UID        PID  PPID  C STIME TTY      STAT   TIME CMD
systemd+  2112  2098  0 14:23 ?        Ssl    0:00 redis-server *:6379
```

**O que aconteceu**: Docker pediu ao kernel para spawnar um novo processo. O kernel assinou um PID (2112 neste caso) como faria para qualquer outro processo. Não há magia, apenas um processo normal rodando sob isolamento.

### Análise da Árvore de Processos

Para entender a relação parent-child:

```bash
$ ps -ef --forest | grep -A 2 redis-server
containerd-s  2098 2086  0 14:23 ?  Ss     0:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id a7f3c8d9e2f1b4a6c5d8e9f0a1b2c3d4e5f6a7b8
redis        2112  2098  0 14:23 ?  Ssl    0:00  \_ redis-server *:6379
```

**Hierarquia de processos**:
```
containerd-shim-runc-v2 (PID 2098) [pai]
    └── redis-server (PID 2112) [filho - o container]
```

A relação parent-child revela a estrutura de tempo de execução:

1. **containerd-shim**: Camada de runtime do Docker. Gerencia o ciclo de vida do container
2. **redis-server**: O processo real da aplicação dentro do namespace isolado

Se você instalasse Redis diretamente no host, o parent seria `systemd`, `bash`, ou outro launcher. A presença de `containerd-shim-runc-v2` é a assinatura de um container.

---

## Confirmação com Docker CLI

### Listar Containers em Execução

```bash
$ docker ps
CONTAINER ID  IMAGE        COMMAND                 CREATED        STATUS       PORTS      NAMES
a7f3c8d9e2f1 redis:7-alp  "redis-server *:6379"  2 minutes ago  Up 2 minutes 6379/tcp   redis-demo
```

### Obter PID do Container do Host

```bash
# Obter PID do container
$ docker inspect -f '{{.State.Pid}}' redis-demo
2112

# Obter mais detalhes do container
$ docker inspect redis-demo | jq '.[] | {Pid: .State.Pid, Running: .State.Running, Namespaces: .HostConfig.IpcMode}'
{
  "Pid": 2112,
  "Running": true,
  "Namespaces": "shareable"
}
```

### Comparação: Process Normal vs Container

```bash
# Processo nativo no host (nginx)
$ ps -ef | grep nginx
www-data  1234  1198  0 08:00 ?  Ss   0:00 nginx: worker process
# Parent: systemd (PID 1 ou outro systemd service)

# Processo dentro de container
$ docker run -d nginx:latest
$ ps -ef | grep nginx
www-data  3456  2345  0 14:23 ?  Ss   0:00 nginx: master process
# Parent: containerd-shim-runc-v2 (2345)
```

**Como diferenciar**: Procure pelo parent. Se for `containerd-shim-runc-v2` ou `docker-runc`, está em um container.

---

## Verificando Namespaces

O passo seguinte é confirmar que esse processo está isolado via namespaces:

```bash
# Ver namespaces do container
$ ls -l /proc/2112/ns/
total 0
lrwxrwxrwx 1 systemd systemd 0 Apr 26 14:23 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 systemd systemd 0 Apr 26 14:23 ipc -> 'ipc:[4026532715]'
lrwxrwxrwx 1 systemd systemd 0 Apr 26 14:23 mnt -> 'mnt:[4026532745]'
lrwxrwxrwx 1 systemd systemd 0 Apr 26 14:23 net -> 'net:[4026532820]'
lrwxrwxrwx 1 systemd systemd 0 Apr 26 14:23 pid -> 'pid:[4026532878]'
lrwxrwxrwx 1 systemd systemd 0 Apr 26 14:23 uts -> 'uts:[4026532719]'
user -> 'user:[4026531837]'
```

**O que os números entre colchetes significam**:
- Namespaces com o **mesmo número** são compartilhados
- Namespaces com **números diferentes** são isolados

### Comparar Namespaces: Host vs Container

```bash
# Ver namespace PID do host (init)
$ ls -l /proc/1/ns/pid
pid:[4026531836]

# Ver namespace PID do container
$ ls -l /proc/2112/ns/pid
pid:[4026532878]  # DIFERENTE! Namespace isolado

# Comprovar isolamento
$ cat /proc/1/ns/pid
pid:[4026531836]

$ cat /proc/2112/ns/pid
pid:[4026532878]  # Números diferentes = isolamento

# Para cada tipo de namespace:
$ diff <(cat /proc/1/ns/* | sort) <(cat /proc/2112/ns/* | sort)
# Mostrará as diferenças
```

---

## O que Você Aprendeu

✅ Containers são processos Linux com PIDs normais  
✅ Cada container é filho de um `containerd-shim`  
✅ Você pode listar containers com `ps` normal  
✅ Namespaces diferentes significam isolamento  
✅ Sem namespaces isolados = sem container  

---

## Próximos Passos

Agora que confirmamos que containers são processos, é hora de explorar **como inspecioná-los**. O próximo artigo mostra como usar o filesystem `/proc` para explorar o interior de um container sem usar nenhum comando Docker.

**Próximo**: [[02-Inspecionando_Containers_com_proc|Inspecionando Containers com /proc →]]

---

## Referências

- [Linux Namespaces Man Pages](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Docker Architecture](https://docs.docker.com/get-started/docker-overview/#docker-architecture)
- [containerd Shim](https://github.com/containerd/containerd/blob/main/runtime/v2/runc/v2/shim/shim.go)

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]
