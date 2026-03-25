---
title: Usando jq para analisar logs NDJSON
tags:
  - JSON
  - jq
enableToc: true
---

# O que é NDJSON?

NDJSON (Newline Delimited JSON) é um formato de arquivo onde cada linha é um objeto JSON separado. É comumente usado para armazenar grandes volumes de dados estruturados, como logs, onde cada linha representa um evento ou registro distinto.

# Usando jq para analisar NDJSON

[jq](https://stedolan.github.io/jq/) é uma ferramenta de linha de comando poderosa para processar e manipular dados JSON. Ele pode ser usado para extrair informações específicas, filtrar registros, transformar dados e muito mais.

## Exemplo de uso

Suponha que você tenha um arquivo de log NDJSON chamado `logs.json` com o seguinte conteúdo:

```json
Chat, eu tenho um arquivo de log com logs no formado json-inline, como posso usar o jq para tornar mais amigavel a visualização? exemplo dos logs:

```json
{"log.level":"info","@timestamp":"2026-03-25T01:35:11.289Z","log.logger":"auditd","log.origin":{"file.name":"auditd/audit_linux.go","file.line":134},"message":"socket_type=multicast will be used.","service.name":"auditbeat","ecs.version":"1.6.0"}
{"log.level":"warn","@timestamp":"2026-03-25T01:35:11.315Z","log.logger":"cfgwarn","log.origin":{"file.name":"host/host.go","file.line":202},"message":"BETA: The system/host dataset is beta","service.name":"auditbeat","ecs.version":"1.6.0"}
{"log.level":"warn","@timestamp":"2026-03-25T01:35:11.320Z","log.logger":"cfgwarn","log.origin":{"file.name":"login/login.go","file.line":93},"message":"BETA: The system/login dataset is beta","service.name":"auditbeat","ecs.version":"1.6.0"}
{"log.level":"warn","@timestamp":"2026-03-25T01:35:11.324Z","log.logger":"cfgwarn","log.origin":{"file.name":"process/process.go","file.line":146},"message":"BETA: The system/process dataset is beta","service.name":"auditbeat","ecs.version":"1.6.0"}
{"log.level":"warn","@timestamp":"2026-03-25T01:35:11.330Z","log.logger":"cfgwarn","log.origin":{"file.name":"user/user.go","file.line":231},"message":"BETA: The system/user dataset is beta","service.name":"auditbeat","ecs.version":"1.6.0"}```
```

🧱 1. Pretty print (formatar bonito)

Pra simplesmente “quebrar” cada linha em JSON formatado:

```bash
jq . logs.json
```

Isso já transforma de uma linha só para algo assim:

```bash
{
  "log.level": "info",
  "@timestamp": "2026-03-25T01:35:11.289Z",
  "log.logger": "auditd",
  "log.origin": {
    "file.name": "auditd/audit_linux.go",
    "file.line": 134
  },
  "message": "socket_type=multicast will be used.",
  "service.name": "auditbeat",
  "ecs.version": "1.6.0"
}
```

🎯 2. Mostrar só campos importantes

Logs ficam muito mais legíveis quando você filtra só o que interessa:

```bash
jq '{time: ."@timestamp", level: ."log.level", msg: .message}' logs.json
```

Isso vai mostrar só o timestamp, nível e mensagem:

```bash
{
  "time": "2026-03-25T01:35:11.289Z",
  "level": "info",
  "msg": "socket_type=multicast will be used."
}
```

📊 3. Formato estilo linha (mais “humano”)

Se quiser algo tipo log tradicional:

```bash
jq -r '"\(.["@timestamp"]) [\(.["log.level"])] \(.message)"' logs.json
```

Isso vai mostrar algo assim:

```2026-03-25T01:35:11.289Z [info] socket_type=multicast will be used.```

🔍 4. Filtrar por nível (ex: só WARN)

```bash
jq 'select(.["log.level"] == "warn")' logs.json
```

Ou combinando com saída bonita:

```bash
jq -r 'select(.["log.level"]=="warn") | "\(.["@timestamp"]) [WARN] \(.message)"' logs.json
```

🧠 5. Ordenar por timestamp

Se quiser garantir ordem (caso o arquivo esteja bagunçado):

```bash
jq -s 'sort_by(."@timestamp")[]' logs.json
```

🔥 6. Destacar origem do log

Seu log tem info útil de arquivo/linha:

```bash
jq -r '"\(.["@timestamp"]) [\(.["log.level"])] \(.["log.origin"].file.name):\(.["log.origin"].file.line) - \(.message)"' logs.json
```

Isso vai mostrar algo tipo:

```2026-03-25T01:35:11.289Z [info] auditd/audit_linux.go:134 - socket_type=multicast will be used.```

💡 Dica bônus (muito útil)

Se você estiver usando tail -f:

```bash
tail -f logs.json | jq -r '"\(.["@timestamp"]) [\(.["log.level"])] \(.message)"'
```