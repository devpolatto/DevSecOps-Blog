---
title: Auto Scalling de um node spot no AKS
tags:
  - Azure
  - AKS
  - kubernetes
  - spot
enableToc: true
---

```bash
#!/bin/bash

RESOURCE_GROUP=""
CLUSTER_NAME=""
NODE_POOL_NAME="monitoring"
CHECK_INTERVAL=300
DESIRED_COUNT=1


echo "Starting node count monitoring for node pool '$NODE_POOL_NAME' in cluster '$CLUSTER_NAME'..."

# Infinite loop to check node count every 5 minutes
while true; do

    NODE_COUNT=$(az aks nodepool show \
     --subscription <subscription-id> \
     --resource-group "$RESOURCE_GROUP" \
     --cluster-name "$CLUSTER_NAME" \
     --name "$NODE_POOL_NAME" \
     --query "count" \
     -o tsv 2>/dev/null)

    if [ $? -ne 0 ]; then
        echo "$(date): Error retrieving node count for '$NODE_POOL_NAME'. Check configuration."
        sleep "$CHECK_INTERVAL"
        continue
    fi

    echo "$(date): Current node count for '$NODE_POOL_NAME' is $NODE_COUNT."

    if [ "$NODE_COUNT" -eq 0 ]; then
        echo "$(date): Node count is 0. Scaling to $DESIRED_COUNT node(s)..."
        az aks nodepool scale \
          --subscription <subscription-id> \
          --resource-group "$RESOURCE_GROUP" \
          --cluster-name "$CLUSTER_NAME" \
          --name "$NODE_POOL_NAME" \
          --node-count "$DESIRED_COUNT" \
          --no-wait

        if [ $? -eq 0 ]; then
            echo "$(date): Scale command issued successfully."
        else
            echo "$(date): Error scaling node pool '$NODE_POOL_NAME'."
        fi
    fi

    sleep "$CHECK_INTERVAL"
done
```