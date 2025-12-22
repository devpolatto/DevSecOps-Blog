---
title: Obter o tamanho do diretório
tags:
  - Linux
  - Disk
enableToc: true
---
```bash
TARGET_DIR="/var"
USAGE_MB=$(du -s "$TARGET_DIR" | awk '{print $1}' | numfmt --to=si --from-unit=1024 --suffix=MB) && \
echo $USAGE_MB
```