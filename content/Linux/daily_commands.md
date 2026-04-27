---
title: Linux - Daily Commands
description: >
     Uma coleção de comandos essenciais para o dia a dia no Linux, abrangendo desde manipulação de arquivos até monitoramento de processos e redes.
enableToc: true
tags:
  - Linux
  - Commands
aliases:
---


## Manipulação de arquivos e diretórios

```bash
find /home/polatto/Playground/AzureDevOpsAI/planningAI -not -path '*/.git/*' -not -path '*/node_modules/*' -not -path '*/__pycache__/*' -not -path '*/planningai.egg-info/*' -not -path '*/.venv/*' | sort | sed 's|/home/polatto/Playground/AzureDevOpsAI/planningAI||' | sed 's|^/||'
```