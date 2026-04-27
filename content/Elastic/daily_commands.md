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

PUT /_security/role/TESTE_POLATTO
{
    "cluster": [
      "manage_index_templates",
      "manage_ml",
      "manage",
      "manage_watcher"
    ],
    "indices": [
      {
        "names": [
          "*"
        ],
        "privileges": [
          "view_index_metadata","manage","read","all"
        ],
        "allow_restricted_indices": false
      }
    ],
    "applications": [
      {
        "application": "kibana-.kibana",
        "privileges": [
          "feature_apm.all",
          "feature_canvas.all",
          "feature_dashboard.all",
          "feature_dev_tools.all",
          "feature_discover.all",
          "feature_filesManagement.all",
          "feature_filesSharedImages.all",
          "feature_graph.all",
          "feature_indexPatterns.all",
          "feature_infrastructure.all",
          "feature_kibana.all",
          "feature_logs.all",
          "feature_maps.all",
          "feature_ml.all",
          "feature_observabilityAIAssistant.all",
          "feature_profiling.all",
          "feature_savedObjectsManagement.all",
          "feature_savedQueryManagement.all",
          "feature_slo.all",
          "feature_uptime.all",
          "feature_visualize.all",
          "dashboard_v2.all",
          "discover_v2.all",
          "canvas.all",
          "visualize_v2.all"
        ],
        "resources": [
          "space:default"
        ]
      }
    ],
    "run_as": [],
    "metadata": {},
    "transient_metadata": {
      "enabled": true
    }
}
```
