---
title: Segurança e Isolamento de Containers
description: >
  Entenda o modelo de segurança dos containers, o que está isolado, o que não está e os riscos envolvidos. Comparação com VMs e melhores práticas de hardening.
enableToc: true
tags:
  - Docker
  - Process
  - Linux
  - Containerd
  - Namespaces
  - Cgroups
  - Security
  - Isolation
aliases:
---

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]  
**Anterior**: [[04-Debugging_Prático de_Containers|← Debugging]]  
**Próximo**: [[06-Namespaces_e_Cgroups_em_Profundidade|Profundidade →]]

---

## Introdução: Modelo de Segurança de Containers

Containers **não são VMs**. Uma VM tem seu próprio kernel isolado. Containers compartilham o kernel do host. Isso tem implicações profundas de segurança.

---

## O Que Está Isolado

### ✅ Isolado (Visto de Forma Diferente)

| Recurso | O que está isolado |
|---------|-------------------|
| **Filesystem** | Root `/` é diferente (overlay filesystem) |
| **Processos** | PID namespace — vê PID 1 como o próprio init |
| **Rede** | Network namespace — interfaces, IPs, routing próprios |
| **Users** | User namespace — UIDs podem ser remapeados |
| **IPC** | Message queues, shared memory, semaphores |
| **Hostname** | UTS namespace — hostname próprio |
| **Cgroups** | Limits de CPU, memória, I/O |

### ❌ NÃO Está Isolado (Compartilhado)

| Recurso | Por que não está isolado |
|---------|--------------------------|
| **Kernel** | Todos os containers usam o mesmo kernel do host |
| **Syscalls** | Mesma interface de syscall para todos |
| **Clock** | Mesmo relógio do sistema |
| **Alguns dispositivos** | `/dev/null`, `/dev/zero`, etc são compartilhados |
| **Performance** | Um container pode starving CPU dos outros |
| **KSM** | Kernel Samepage Merging compartilha páginas |

---

## Comparação: Containers vs VMs

```
┌─────────────────────────────────┐
│         VirtualMachine          │
├─────────────────────────────────┤
│   Linux Kernel (isolado)        │
│   ├─ Processos próprios         │
│   ├─ Filesystem próprio         │
│   ├─ Rede própria               │
│   └─ Tudo é isolado             │
├─────────────────────────────────┤
│     Hypervisor (vmx/svm)        │
├─────────────────────────────────┤
│      Host Linux Kernel          │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│    Container 1     │ Container 2 │
├────────────────────┼─────────────┤
│ Namespaces isolados│ Namespaces  │
│ Cgroups limitados  │ isolados    │
├────────────────────┴─────────────┤
│    Host Linux Kernel (COMPARTILHADO!)
│    (Qualquer syscall pode afetar todos)
└────────────────────────────────────┘
```

---

## Risco 1: Privilege Escalation

### Container Rodando com CAP_SYS_ADMIN

```bash
# Criar container com privilege escalation (perigoso!)
$ docker run -it --cap-add=SYS_ADMIN ubuntu:latest

# Dentro do container:
$ whoami
root

# Com SYS_ADMIN, um container pode:
# 1. Montar filesystems
$ mount -t tmpfs tmpfs /mnt

# 2. Criar novos devices
$ mknod /dev/foo c 10 200

# 3. Modificar politicas de segurança
$ echo 1 > /proc/sys/kernel/unprivileged_userns_clone

# 4. Escapar do container (em alguns casos)
```

### Container Rodando como root

```bash
# Default: muitos containers rodam como root!
$ docker run -d redis:7-alpine
# Processo roda como UID 0 (root)

# Verificar
$ ps aux | grep redis-server
root      2112  0.1  0.2  174156   51024 ?  Ssl  14:23   0:00 redis-server *:6379

# Se container for comprometido:
# - Atacante tem acesso root ao container
# - Pode modificar qualquer arquivo
# - Pode fazer syscalls que um usuário normal não pode

# Comparar com container seguro (rodando como redis user):
$ docker run -d --user redis:redis redis:7-alpine
```

---

## Risco 2: Container Escape

### Vulnerabilidade do Kernel

```bash
# Um bug no kernel pode permitir escape de container
# Exemplos:
# - CVE-2021-22555 (Netfilter overflow)
# - CVE-2021-4034 (policykit OverlayFS)
# - CVE-2022-0847 (Dirty COW)

# Se vulnerabilidade existe no kernel, TODOS os containers estão em risco
# VMs estariam protegidas (kernel isolado)

# Verificar versão do kernel
$ uname -r
5.15.0-22-generic

# Verificar se há CVEs conhecidas
$ sudo apt install ubuntu-advantage-tools
$ sudo pro security-status
```

### Experimental: User Namespace Remapping

```bash
# Remapear UIDs para reduzir risco
# Se container vê UID 0, map para UID 100000 no host

$ docker run -d --userns-remap=default redis:7-alpine

# Verificar
$ ps aux | grep redis
systemd+  2112  0.1  0.2  174156   51024 ?  Ssl  14:23   0:00 redis-server

# No host, o processo é executado como systemd+ (UID ~100000)
# Não como root (UID 0)
```

---

## Risco 3: Acesso ao /proc do Host

### Dados Sensíveis Expostos

```bash
# Do host, qualquer um com sudo pode ver:
$ sudo cat /proc/2112/environ  # Variáveis de ambiente (pode incluir secrets!)
POSTGRES_PASSWORD=super-secret-123
API_KEY=sk-abc123...

# Ver comandos de startup
$ sudo cat /proc/2112/cmdline | tr '\0' ' '
java -Xmx512m -Dspring.datasource.password=secret ...

# Ver memória do processo
$ sudo cat /proc/2112/mem | strings | grep password
```

**Mitigação**:
- Usar secrets management (HashiCorp Vault, AWS Secrets Manager)
- Não passar secrets como environment variables
- Usar read-only rootfs quando possível

---

## Risco 4: Kernel Exploitation

### Syscalls Sem Filtro

```bash
# Por padrão, container pode fazer qualquer syscall suportada
$ docker run ubuntu:latest strace -e trace=% whoami 2>&1 | wc -l
300+  # Centenas de syscalls diferentes

# Algumas perigosas:
# - ptrace (debugar outros processos)
# - sysrq (comandos emergenciais do kernel)
# - ioctl (interface com devices)

# Mitigação: Seccomp (Secure Computing Mode)
$ docker run --security-opt seccomp=default ubuntu:latest
# Default profile bloqueia ~50 syscalls perigosas
```

### Exemplo: ptrace e Container Escape

```bash
# Sem filtro, container pode:
$ docker run -d app:latest

# Dentro:
$ strace -p 1  # Debugar init do container
$ strace -p $(pgrep nginx)  # Debugar outro processo

# Com Seccomp ativado:
$ docker run --security-opt seccomp=default -d app:latest
# ptrace será bloqueado
```

---

## Risco 5: Resource Exhaustion

### Sem Limites, Um Container Pode Starvar Outros

```bash
# Container SEM limites
$ docker run --name greedy -d ubuntu:latest bash -c "while true; do dd if=/dev/zero of=/tmp/file bs=1M; done"

# Consome toda a RAM e mata outros containers!

# Container COM limites
$ docker run --name nice -m 512m --cpus 1 -d app:latest
# Limitado a 512MB RAM e 1 CPU
```

### Monitorar Limits

```bash
# Ver limites configurados
$ docker inspect greedy | jq '.[] | {Memory, MemorySwap, CpuQuota, CpuPeriod}'

# Ver cgroup limits
$ cat /sys/fs/cgroup/docker/*/memory.limit_in_bytes | sort -n | uniq -c
```

---

## Risco 6: Network Namespace Bypass

### Compartilhar Network Namespace

```bash
# Container A tem acesso à rede de Container B!
$ docker run --name a -d app1:latest
$ docker run --network container:a -d app2:latest

# app2 vê mesma interface de rede, mesmo IP, mesmas portas
# Pode sniff traffic de a ou fazer spoofing
```

---

## Checklist de Segurança

```bash
# [ ] Container roda como usuário não-root?
$ docker inspect <container> | jq '.[] | .Config.User'
# Deve ser algo como "app:app", não vazio

# [ ] Seccomp está ativado?
$ docker inspect <container> | jq '.[] | .HostConfig.SecurityOpt'
# Deve conter "seccomp=default"

# [ ] AppArmor ou SELinux está ativado?
$ docker inspect <container> | jq '.[] | .AppArmorProfile'
# Deve conter um profile, não "unconfined"

# [ ] Capabilities são limitadas?
$ docker inspect <container> | jq '.[] | .HostConfig.CapAdd'
# Deve ser null ou apenas capabilities necessárias

# [ ] Filesystem é read-only?
$ docker inspect <container> | jq '.[] | .HostConfig.ReadonlyRootfs'
# Idealmente true

# [ ] Recursos têm limits?
$ docker inspect <container> | jq '.[] | {Memory, CpuQuota, MemorySwap}'
# Todos devem ter valores, não -1 (unlimited)

# [ ] Network está isolada?
$ docker inspect <container> | jq '.[] | .HostConfig.NetworkMode'
# Deve ser "bridge" ou custom, não "host"
```

---

## Dockerfile Seguro

```dockerfile
# ❌ INSEGURO
FROM ubuntu:latest
RUN apt-get update && apt-get install -y nginx
CMD ["nginx", "-g", "daemon off;"]
# Roda como root!
# Sem AppArmor
# Sem Seccomp
# Sem limites

# ✅ SEGURO
FROM ubuntu:22.04 as builder
RUN apt-get update && apt-get install -y --no-install-recommends nginx=1.18.0-6

FROM ubuntu:22.04
COPY --from=builder /usr/sbin/nginx /usr/sbin/nginx
COPY --from=builder /etc/nginx /etc/nginx

# Criar usuário não-root
RUN groupadd -r nginx && useradd -r -g nginx nginx

# Remover capabilities desnecessárias
RUN setcap -r /usr/sbin/nginx || true

USER nginx
EXPOSE 8080

# Usar read-only root se possível
VOLUME ["/tmp", "/var/run"]

CMD ["nginx", "-g", "daemon off;"]
```

---

## Docker Security Best Practices

### No Dockerfile

```dockerfile
# 1. Multi-stage: reduzir camadas e dependências
# 2. Non-root user
# 3. Read-only filesystem
# 4. Minimal base image (alpine, distroless)
# 5. No secrets em imagem
```

### No Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    image: app:latest
    user: app:app              # Non-root
    read_only: true            # Read-only rootfs
    tmpfs: /tmp                # Writeable apenas /tmp
    memory: 512m               # Memory limit
    cpus: 1.0                  # CPU limit
    security_opt:
      - seccomp:default        # Seccomp profile
      - apparmor=docker-default # AppArmor profile
    cap_drop:
      - ALL                    # Drop todas as capabilities
    cap_add:
      - NET_BIND_SERVICE       # Apenas as necessárias
    networks:
      - app-network            # Network isolada, não host
```

### No docker run

```bash
# Rodar container seguro
$ docker run \
  --user app:app \
  --read-only \
  --tmpfs /tmp \
  -m 512m \
  --cpus 1.0 \
  --security-opt seccomp=default \
  --security-opt apparmor=docker-default \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --network app-network \
  app:latest
```

---

## Scanning de Vulnerabilidades

```bash
# Docker Scout (integrado ao docker CLI)
$ docker scout cves app:latest

# Trivy (open source)
$ trivy image app:latest

# Grype (Anchore)
$ grype app:latest

# Snyk
$ snyk test docker://app:latest
```

---

## Resumo de Riscos e Mitigações

| Risco | Severidade | Mitigação |
|-------|------------|-----------|
| Container como root | 🔴 Alta | Use `--user non-root` |
| Syscalls ilimitadas | 🔴 Alta | Use `--security-opt seccomp=default` |
| Sem resource limits | 🟠 Média | Use `-m`, `--cpus` |
| Secrets em env vars | 🔴 Alta | Use secrets management |
| Privileged mode | 🔴 Crítica | Evitar `--privileged` |
| Host network mode | 🟠 Média | Não use `--network host` |
| Filesystem writable | 🟢 Baixa | Use `--read-only` |

---

## Conclusão

**Containers não são isolamento de segurança completo**. Eles são isolamento de **recurso** com **conveniência**. Para segurança máxima:

1. ✅ Roda como non-root
2. ✅ Usa Seccomp
3. ✅ Limita capabilities
4. ✅ Define resource limits
5. ✅ Mantém kernel atualizado
6. ✅ Scan imagens frequentemente

---

## Próximos Passos

Para uma compreensão **profunda** de como os mecanismos realmente funcionam, continue para o artigo final.

**Próximo**: [[06-Namespaces_e_Cgroups_em_Profundidade|Profundidade →]]

---

## Referências

- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Seccomp Documentation](https://docs.docker.com/engine/security/seccomp/)
- [AppArmor](https://docs.docker.com/engine/security/apparmor/)
- [OWASP Container Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

**Série**: [[00-Containers_Sao_Apenas_Processos_Linux-indice|← Voltar ao Índice]]
