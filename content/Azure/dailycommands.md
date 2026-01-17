---
title: Daily commands
tags:
  - Azure
enableToc: true
---
# Azure Query
## Verifica discos não vinculados a uma VM

```sql
resources | where type =~ 'microsoft.compute/disks' and managedBy == ''| extend diskState = tostring(properties.diskState)| where managedBy == '' and diskState != 'ActiveSAS'or diskState == 'Unattached' and diskState != 'ActiveSAS' and tags !contains 'ASR-ReplicaDisk' and tags !contains 'asrseeddisk'| project subscriptionId,id
```