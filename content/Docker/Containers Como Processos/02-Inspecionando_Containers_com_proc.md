---
title: Inspecionando Containers com proc
description: |
  Usando o filesystem virtual /proc para explorar o interior de um container, acessando namespaces, variáveis de ambiente e arquivos do container diretamente do host.
enableToc: true
tags:
  - Docker
  - Process
  - Linux
  - Containerd
  - Namespaces
  - Cgroups
  - proc
  - Filesystem
  - Variables
  - Environments
aliases:
---

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]
**Anterior**: [[01-Fundamentos-Containers_Como_Processos|← Fundamentos]]
**Próximo**: [[03-Manipulando_Containers_como_Processos_Normais|Manipulando Containers →]]

---

## O Virtual Filesystem /proc

Linux expõe `/proc`, um filesystem virtual que fornece visão em tempo real de cada processo. Cada processo tem seu próprio diretório nomeado com seu PID.

```bash
# Listar diretórios de processos
$ ls -d /proc/[0-9]* | head -20
/proc/1      /proc/10   /proc/100  /proc/1001 /proc/1002
/proc/1010   /proc/1011 /proc/1012 /proc/1013 /proc/1014

# Conteúdo específico do PID do container
$ ls -la /proc/2112/
total 0
dr-xr-xr-x  9 systemd systemd  0 Apr 26 14:23 .
-r--r--r--  1 systemd systemd  0 Apr 26 14:23 attr
-r--r--r--  1 systemd systemd  0 Apr 26 14:23 autogroup
-r--r--r--  1 systemd systemd  0 Apr 26 14:23 auxv
--w-------  1 systemd systemd  0 Apr 26 14:23 cgroup
-r--r--r--  1 systemd systemd  0 Apr 26 14:23 cmdline
-rw-r--r--  1 systemd systemd  0 Apr 26 14:23 coredump_filter
lrwxrwxrwx  1 systemd systemd  0 Apr 26 14:23 cwd -> /data
-r--r--r--  1 systemd systemd  0 Apr 26 14:23 environ
-r--r--r--  1 systemd systemd  0 Apr 26 14:23 exe
dr-x------  2 systemd systemd  0 Apr 26 14:23 fd
dr-x------  2 systemd systemd  0 Apr 26 14:23 fdinfo
-rw-r--r--  1 systemd systemd  0 Apr 26 14:23 limits
-r--r--r--  1 systemd systemd  0 Apr 26 14:23 maps
dr-x--x--x  6 systemd systemd  0 Apr 26 14:23 ns
```

---

## Explorando Namespaces

### Visualizar Namespaces do Container

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

### Comparar Namespaces: Host vs Container

```bash
# Comparar namespace PID (isolamento de processos)
$ cat /proc/1/ns/pid
pid:[4026531836]  # Host

$ cat /proc/2112/ns/pid
pid:[4026532878]  # Container - DIFERENTE (isolado)

# Comparar namespace de rede (isolamento de rede)
$ cat /proc/1/ns/net
net:[4026531956]  # Host

$ cat /proc/2112/ns/net
net:[4026532820]  # Container - DIFERENTE (isolado)

# Comparar namespace de mount (isolamento de filesystem)
$ cat /proc/1/ns/mnt
mnt:[4026531840]  # Host

$ cat /proc/2112/ns/mnt
mnt:[4026532745]  # Container - DIFERENTE (isolado)
```

**Interpretação**: Namespaces com números diferentes = **isolamento completo** naquele recurso.

### Inspecionar o que está Isolado

```bash
# Criar tabela de comparação
$ for ns in cgroup ipc mnt net pid uts user; do
    echo "=== $ns ==="
    echo "Host:      $(cat /proc/1/ns/$ns 2>/dev/null)"
    echo "Container: $(cat /proc/2112/ns/$ns 2>/dev/null)"
  done

# Output
=== cgroup ===
Host:      cgroup:[4026531835]
Container: cgroup:[4026531835]   # IGUAL = Compartilhado!

=== ipc ===
Host:      ipc:[4026531839]
Container: ipc:[4026532715]      # DIFERENTE = Isolado

=== mnt ===
Host:      mnt:[4026531840]
Container: mnt:[4026532745]      # DIFERENTE = Isolado

=== net ===
Host:      net:[4026531956]
Container: net:[4026532820]      # DIFERENTE = Isolado

=== pid ===
Host:      pid:[4026531836]
Container: pid:[4026532878]      # DIFERENTE = Isolado

=== uts ===
Host:      uts:[4026531838]
Container: uts:[4026532719]      # DIFERENTE = Isolado
```

---

## Acessar Variáveis de Ambiente

### Ver Variáveis do Processo Container

```bash
# Mostrar environment em formato legível
$ cat /proc/2112/environ | tr '\0' '\n'
HOSTNAME=a7f3c8d9e2f1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOME=/root

# Ou com sort e grep
$ cat /proc/2112/environ | tr '\0' '\n' | sort

# Comparar com host
$ cat /proc/1/environ | tr '\0' '\n' | head -10
```

---

## Explorar o Filesystem Root do Container

### O Symlink Root

```bash
# Ver o filesystem root do container do host
$ sudo ls -l /proc/2112/root
lrwxrwxrwx 1 root root 0 Apr 26 14:24 /proc/2112/root -> /var/lib/docker/overlay2/xyz123/merged
```

O symlink aponta para o overlay filesystem do Docker, que é a raiz do container.

### Explorar Conteúdo

```bash
# Listar conteúdo da raiz do container (do host)
$ sudo ls /proc/2112/root
bin  boot  data  dev  etc  home  lib  media  mnt  opt  proc  root  run  srv  sys  tmp  usr  var

# Navegar no filesystem do container
$ sudo ls /proc/2112/root/etc/
addgroup       hostname  modules       nsswitch.conf  ssl           upstart
adduser        hosts     motd           passwd         sysctl.conf
aliases        inittab   mtab           resolv.conf    sysctl.d
hostname.old   issue     network        rmt            terminfo

# Ver arquivo específico
$ sudo cat /proc/2112/root/etc/hostname
a7f3c8d9e2f1
```

### Manipular Arquivos do Container pelo Host

```bash
# Criar um arquivo do host que aparecerá dentro do container
$ sudo touch /proc/2112/root/host-created-file.txt
$ sudo echo "Criado do host" > /proc/2112/root/host-created-file.txt

# Verificar de dentro do container
$ docker exec redis-demo ls -la /host-created-file.txt
-rw-r--r-- 1 root root 17 Apr 26 14:25 /host-created-file.txt

$ docker exec redis-demo cat /host-created-file.txt
Criado do host

# Modificar arquivo do container pelo host
$ sudo sed -i 's/redis/REDIS/g' /proc/2112/root/etc/hostname
$ docker exec redis-demo cat /etc/hostname
# Mostrará as mudanças
```

---

## File Descriptors Abertos

### Ver Arquivos e Conexões Abertas

```bash
# Listar file descriptors
$ ls -la /proc/2112/fd/ | head -20
total 0
lrwx------ 1 systemd systemd 64 Apr 26 14:23 0 -> /dev/null
lrwx------ 1 systemd systemd 64 Apr 26 14:23 1 -> /dev/null
lrwx------ 1 systemd systemd 64 Apr 26 14:23 2 -> /dev/null
lrwx------ 1 systemd systemd 64 Apr 26 14:23 3 -> socket:[28473920]
lrwx------ 1 systemd systemd 64 Apr 26 14:23 4 -> socket:[28474001]
lrwx------ 1 systemd systemd 64 Apr 26 14:23 5 -> /var/lib/docker/overlay2/xyz123/merged/data/dump.rdb

# Explicar
#   0, 1, 2 = stdin, stdout, stderr
#   3, 4 = sockets (possivelmente conexões de rede)
#   5 = arquivo aberto (dump.rdb do Redis)
```

### Usando lsof (se disponível)

```bash
# Instalação necessária
$ sudo apt install lsof  # Debian/Ubuntu
$ sudo yum install lsof  # RHEL/CentOS

# Ver arquivos abertos pelo container
$ sudo lsof -p 2112 | head -30
COMMAND  PID USER  FD  TYPE    DEVICE SIZE/OFF    NODE NAME
redis-s 2112 redis 0u  CHR      1,3              1027 /dev/null
redis-s 2112 redis 1w  REG      0,36               0  527645 /var/log/redis.log
redis-s 2112 redis 2w  REG      0,36               0  527645 /var/log/redis.log
redis-s 2112 redis 3u  IPv4     52314    0t0      TCP *:6379 (LISTEN)
redis-s 2112 redis 4u  IPv6     52315    0t0      TCP *:6379 (LISTEN)
redis-s 2112 redis 5u  REG    10,3    123456   1048576 /data/dump.rdb

# Ver apenas conexões de rede
$ sudo lsof -p 2112 -i
```

---

## Limites de Recurso

### Ver Limites do Container

```bash
# Inspecionar limites de CPU, memória, etc
$ cat /proc/2112/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size              unlimited            unlimited            bytes
Max data size              unlimited            unlimited            bytes
Max stack size             8388608              unlimited            bytes
Max core file size         0                    unlimited            bytes
Max resident set           unlimited            unlimited            bytes
Max processes              15343                15343                processes
Max open files             1048576              1048576              files
Max locked memory          65536                65536                bytes
Max address space          unlimited            unlimited            bytes
Max file locks             unlimited            unlimited            locks
Max pending signals        15343                15343                signals
Max msgqueue size          819200               819200               bytes
Max nice priority          0                    0
Max realtime priority      0                    0
Max realtime timeout       unlimited            unlimited            us
```

---

## Casos de Uso Práticos

### Forensics: Investigar o que um Container Fez

```bash
# Container parou inesperadamente. O que ele abriu?
$ sudo cat /proc/2112/maps | head -20
# Mostra memory mappings - útil para entender o que rodava

# Ver linha de comando completa
$ cat /proc/2112/cmdline | tr '\0' ' ' && echo
redis-server *:6379

# Ver diretório atual
$ sudo ls -l /proc/2112/cwd
lrwxrwxrwx 1 systemd systemd 0 Apr 26 14:25 /proc/2112/cwd -> /data
```

### Monitorar Mudanças em Tempo Real

```bash
# Watch: monitorar mudanças no filesystem do container
$ watch -n 0.5 "sudo ls -la /proc/2112/fd/ | wc -l"
# Mostra número de file descriptors abertos em tempo real

# Monitorar uso de memória
$ watch -n 1 "cat /proc/2112/status | grep -E 'VmRSS|VmSize'"
```

---

## ⚠️ Implicações de Segurança

**Fato crítico**: Qualquer um com permissão `sudo` no host pode:
- ✅ Ler todos os arquivos do container
- ✅ Modificar filesystem do container
- ✅ Acessar variáveis de ambiente (incluindo secrets)
- ✅ Ver conexões de rede abertas

Esta é uma ferramenta poderosa para **forensics e debugging**, mas representa um **risco de segurança significativo** em ambientes multi-tenant.

---

## Resumo

✅ `/proc/[PID]/ns/` mostra namespaces e isolamento  
✅ `/proc/[PID]/root` é a raiz do filesystem do container  
✅ `/proc/[PID]/environ` mostra variáveis de ambiente  
✅ `/proc/[PID]/fd` mostra arquivos abertos e conexões  
✅ Nenhuma ferramenta Docker é necessária para isso  

---

## Próximos Passos

Agora que você entende como **inspecionar** containers, o próximo passo é aprender como **manipulá-los** — enviando sinais, parando e limitando recursos.

**Próximo**: [[03-Manipulando_Containers_como_Processos_Normais|Manipulando Containers →]]

---

## Referências

- [Linux /proc filesystem](https://man7.org/linux/man-pages/man5/proc.5.html)
- [Namespaces Detailed](https://man7.org/linux/man-pages/man7/namespaces.7.html)

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]
