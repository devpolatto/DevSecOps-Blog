---
title: Shards x Replicas
tags:
  - Elasticsearch
enableToc: true
---
No Elasticsearch, shards e réplicas são conceitos relacionados, mas distintos, que trabalham juntos para gerenciar a distribuição de dados, o desempenho e a tolerância a falhas em um cluster. Como você está trabalhando com um cluster Elastic Cloud de 3 nós e monitorando o crescimento de shards, entender esses termos é fundamental para otimizar sua configuração. Aqui está uma explicação clara da diferença, adaptada ao seu contexto:

# Shards

Definição: Shards são partes individuais de um índice que contêm um subconjunto de seus dados. O Elasticsearch divide os dados de um índice em shards para distribuí-los entre os nós de um cluster, permitindo processamento paralelo e escalabilidade.

Tipos:

- **Shards Primários**: As cópias principais dos dados. Cada documento em um índice pertence a exatamente um shard primário, determinado por um hash do ID do documento.
- Finalidade: Os shards permitem que o Elasticsearch:
	- Distribua dados entre os nós (por exemplo, no seu cluster de 3 nós, os shards são distribuídos para balancear a carga).
	- Paralelize operações como indexação e pesquisa (cada fragmento pode ser processado independentemente).

Na sua configuração:

- Seu modelo de índice define `"number_of_shards": 1`, o que significa que cada índice (por exemplo, shards-monitoring-2025.36) possui 1 fragmento primário.
- Se você tivesse definido "number_of_shards": 2, cada índice teria 2 fragmentos primários, dividindo os dados em dois subconjuntos (por exemplo, documentos A-M em um fragmento, N-Z em outro, aproximadamente).
- Como você cria um novo índice semanalmente, cada semana adiciona 1 fragmento primário ao seu cluster (por exemplo, após 8 semanas, você teria cerca de 8 fragmentos primários antes que o ILM exclua os índices mais antigos em 60 dias).

# Réplicas

Definição: Réplicas são cópias de shards primários. Cada shard primário pode ter zero ou mais shards de réplica, que são cópias idênticas dos dados do primário.

Objetivo:

- Alta Disponibilidade (HA): Réplicas garantem que os dados permaneçam acessíveis caso um nó que hospeda um shard primário falhe. O Elasticsearch coloca réplicas em nós diferentes de seus primários.
- Escalabilidade de Leitura: Réplicas podem lidar com consultas de pesquisa, distribuindo a carga de leitura entre os nós.
- Sobrecarga de Gravação: Atualizações em um shard primário (por exemplo, indexação, exclusão) devem ser replicadas para todas as suas réplicas, aumentando os custos de gravação.

Em Sua Configuração:

- Seu modelo define "number_of_replicas": 1, o que significa que cada shard primário tem 1 shard de réplica. Para cada índice, você tem:
	- 1 shard primário + 1 shard de réplica = 2 shards totais por índice.
	- Exemplo: Para shards-monitoring-2025.36, o shard primário pode estar no Nó 1 e sua réplica no Nó 2, garantindo alta disponibilidade no seu cluster de 3 nós.
- As réplicas dobram o armazenamento (já que os dados de cada shard são duplicados) e adicionam sobrecarga de gravação, mas melhoram a tolerância a falhas e o desempenho das consultas.