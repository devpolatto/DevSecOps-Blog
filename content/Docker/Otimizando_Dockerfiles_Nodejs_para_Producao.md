---
title: Otimizando Dockerfiles Node.js para Produção
description: |
  Guia completo para criar Dockerfiles otimizados para aplicações Node.js em produção, focando em performance, segurança e eficiência.
enableToc: true
tags:
  - Docker
  - NodeJS
  - Performance
  - Security
  - Multi-Stage_Build
  - Optimization
aliases:
  - Docker
  - Daily commands
---

## Introdução

90% dos Dockerfiles de Node.js em produção estão errados. A consequência é previsível: imagens pesadas, builds lentos e deploys frágeis. Enquanto "funciona" é o ponto de partida, o padrão profissional é **rodar bem, consistente e com o menor footprint possível**.

Este documento consolidada as **7 otimizações essenciais** que aplicam impacto direto em:
- **Tamanho de imagem**: 1.1GB → 80-120MB
- **Tempo de build com cache**: 5 minutos → 30 segundos
- **Segurança**: redução de superfície de ataque
- **Custo**: menos storage no registry, menos banda no pull

---

## 1. Multi-Stage Build

### O Problema
Um Dockerfile padrão copia tudo para a imagem final: `node_modules`, código-fonte, `tsconfig.json`, source maps, arquivos de build e `devDependencies`.

### A Solução
Dividir em dois estágios:
- **Stage 1**: Instala dependências e compila TypeScript
- **Stage 2**: Copia apenas o resultado compilado e `node_modules` de produção

### Impacto
A imagem final tem **50% do tamanho** (ou menos) porque não carrega artefatos de build.

### Exemplo

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: Runtime
FROM node:20-alpine

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

COPY --from=builder /app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## 2. Node Alpine como Base

### O Problema
`node:20` é baseado em Debian e pesa ~1.1GB. Traz ferramentas desnecessárias para runtime.

### A Solução
Usar `node:20-alpine` (baseado em Alpine Linux): ~150MB, mesma runtime.

### Benefícios
| Critério | node:20 | node:20-alpine |
|----------|---------|----------------|
| Tamanho | 1.1GB | 150MB | 
| Tempo de pull | ~2 min | ~15 seg |
| Superfície de ataque | Grande | Mínima |
| Runtime Node | Completo | Completo |

### Comparação

```dockerfile
# ❌ Evitar
FROM node:20

# ✅ Usar
FROM node:20-alpine
```

---

## 3. Ordem de COPY é Crítica para Cache

### O Problema
Se copiar todo o código antes de instalar dependências, o Docker reconstrói layers desnecessárias.

```dockerfile
# ❌ Ruim: code → npm install
COPY . .
RUN npm ci
```

**Resultado**: Qualquer mudança no código invalida o cache de `npm install`.

### A Solução
Copiar `package.json` e `package-lock.json` primeiro, instalar, depois copiar o código.

```dockerfile
# ✅ Bom: deps → code
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

**Resultado**: Se o código muda mas as deps não, Docker usa cache e pula a instalação.

### Impacto Mensurável
| Cenário | Sem cache | Com cache |
|---------|-----------|-----------|
| Primeiro build | 5 min | 5 min |
| Código alterado (deps iguais) | 5 min | 30 seg |

**Ganho: 91% mais rápido em builds iterativos.**

---

## 4. `npm ci` vs `npm install`

### A Diferença
- **`npm install`**: Resolve `package.json`, pode atualizar versões, comportamento não-determinístico
- **`npm ci`** (Clean Install): Instala **exatamente** o que está em `package-lock.json`, sem resolver, sem atualizar

### Por Que Importa
```dockerfile
# ❌ Evitar
RUN npm install

# ✅ Usar
RUN npm ci
```

### Benefícios
1. **Determinístico**: Build será idêntico todo dia
2. **Mais rápido**: Pula resolução de dependências
3. **Seguro**: Versões pinadas garantem compatibilidade
4. **CI/CD friendly**: Falha se `package-lock.json` está desatualizado

---

## 5. Usuário Non-Root

### O Problema
Por padrão, containers rodam como `root`. Se a aplicação for comprometida, o atacante tem acesso total.

### A Solução
```dockerfile
# No final do Dockerfile
USER node
```

### Impacto de Segurança
| Cenário | Com root | Com non-root |
|---------|----------|--------------|
| App comprometida | Acesso total | Acesso limitado |
| Container escape | Completo | Restrito |
| Postura de segurança | Fraca | Robusta |

### Exemplo Completo

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Instalar como root
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Copiar app
COPY --from=builder /app/dist ./dist

# Mudar para user não-root
USER node

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## 6. .dockerignore Configurado

### O Problema
Sem `.dockerignore`, o Docker envia tudo para o daemon antes de começar o build. Build context fica enorme.

**Impacto**: Build que deveria levar 30 segundos leva 3 minutos.

### Exemplo de .dockerignore

```
node_modules
.git
.gitignore
.env
.env.local
coverage
dist
build
.vscode
.idea
*.log
.DS_Store
.next
.npm
.eslintcache
```

### O Que Ganhamos
- Menos dados enviados ao daemon
- Builds mais rápidos
- Cache mais eficiente
- Melhor performance em CI/CD

---

## 7. Health Check no Container

### O Problema
Docker sabe se o processo está rodando, mas **não sabe se a aplicação está realmente respondendo**.

### A Solução
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"
```

Ou mais simples com `curl`:
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

### Benefícios
- Orchestrators (Kubernetes, Docker Swarm) sabem quando restart
- Melhor observabilidade
- Evita containers "zumbis"

---

## Dockerfile Completo Otimizado

```dockerfile
# ============================================
# Stage 1: Builder
# ============================================
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# ============================================
# Stage 2: Runtime
# ============================================
FROM node:20-alpine

WORKDIR /app

# Instalar dependências de produção
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copiar build da stage anterior
COPY --from=builder /app/dist ./dist

# Criar usuário não-root
USER node

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

---

## Resultados Esperados

Com estas otimizações aplicadas simultaneamente:

| Métrica | Antes | Depois | Ganho |
|---------|-------|--------|-------|
| Tamanho da imagem | 1.1GB | 90MB | **92% menor** |
| Primeiro build | 5 min | 5 min | - |
| Build com cache (code change) | 5 min | 30 seg | **91% mais rápido** |
| Tempo de pull | 2 min | 15 seg | **93% mais rápido** |
| Superfície de ataque | Grande | Mínima | Significante |
| Segurança de runtime | Fraca | Robusta | Root → non-root |

---

## Checklist de Validação

- [ ] Multi-stage build implementado
- [ ] Base image é `node:X-alpine` (não `node:X`)
- [ ] `COPY package*.json` antes de `npm ci`
- [ ] Usando `npm ci` (não `npm install`)
- [ ] `.dockerignore` está configurado
- [ ] `USER node` ou outro usuário não-root
- [ ] `HEALTHCHECK` definido
- [ ] `npm cache clean --force` após `npm ci --only=production`
- [ ] Imagem testada localmente com `docker build` e `docker run`
- [ ] Build com cache validado (código alterado, imagem reconstruída em <1 min)

---

## Dicas Extras

### 1. Security Scanning
```bash
docker scan seu-app:latest
docker scout cves seu-app:latest
```

### 2. Verificar Tamanho das Layers
```bash
docker history seu-app:latest
```

### 3. Otimização Avançada: Distroless
Para máxima segurança e tamanho mínimo:

```dockerfile
FROM node:20-alpine AS builder
# ... build ...

FROM gcr.io/distroless/nodejs20-debian12

COPY --from=builder /app/dist /app/dist
COPY --from=builder /app/node_modules /app/node_modules

WORKDIR /app
CMD ["dist/index.js"]
```

**Distroless**: Não tem shell, nem package manager. Apenas o Node.js e suas dependências. ~50MB.

### 4. Build Args para Flexibilidade
```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG BUILD_DATE
ARG VCS_REF

LABEL org.label-schema.build-date=$BUILD_DATE \
      org.label-schema.vcs-ref=$VCS_REF
```

### 5. Logs e Debugging
```dockerfile
# Manter informações de debug sem aumentar a imagem
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/* /var/tmp/*
```

---

## Referências e Recursos

- [Alpine Linux](https://www.alpinelinux.org/)
- [Docker Official Node.js Images](https://hub.docker.com/_/node)
- [Distroless Docker Images](https://github.com/GoogleContainerTools/distroless)
- [Docker .dockerignore](https://docs.docker.com/engine/reference/builder/#dockerignore-file)

---

## Conclusão

**Dockerfile não é detalhe de infraestrutura.** É parte da engenharia do produto. Otimizações adequadas resultam em:
- ✅ Imagens leves e rápidas
- ✅ Builds previsíveis
- ✅ Containers seguros
- ✅ Deploy confiável
- ✅ Economia de custo

Aplicar essas 7 otimizações é o padrão, não exceção.
