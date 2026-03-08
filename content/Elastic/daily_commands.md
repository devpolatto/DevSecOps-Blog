---
title: Comandos do dia a dia
tags:
  - Elasticsearch
  - Kibana
enableToc: true
---

# Referências

- [Run API requests](https://www.elastic.co/guide/en/kibana/8.11/console-kibana.html)
- [Script query](https://www.elastic.co/guide/en/elasticsearch/reference/8.11/query-dsl-script-query.html)

# Kibana Security

# Roles

**Inspeção de Roles**

```shell
GET /_security/role/<role_name>
```

**Criação de Roles**

```shell
POST /_security/role/<role_name>
{
  "cluster": ["all"],
  "indices": [
    {
      "names": ["*"],
      "privileges": ["read"]
    }
  ]
}
```

**Criar ou atualizar uma role**

```shell
PUT /_security/role/<role_name>
{
  "cluster": ["all"],
  "indices": [
    {
      "names": ["*"],
      "privileges": ["read"]
    }
  ]
}
```
