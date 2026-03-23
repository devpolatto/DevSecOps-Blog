---
title: Indexando logs do S3 com Filebeat
description: Aprenda a configurar o Filebeat para recuperar logs do S3 e indexá-los no Elasticsearch.
tags:
  - Elasticsearch
  - Filebeat
  - S3
enableToc: true
---

## Introdução

Neste guia, vamos aprender como configurar o Filebeat para recuperar logs armazenados no Amazon S3 e indexá-los no Elasticsearch. O Filebeat é uma ferramenta leve para coletar e enviar dados para o Elasticsearch, ideal para cenários de ingestão de logs.

## Pré-requisitos

Antes de começar, certifique-se de ter:

- Uma conta na AWS com acesso ao S3.
- O Elasticsearch configurado e em execução.

## Passo a passo completo

1. Crie o arquivo de configuração do Filebeat e adicione a configuração para o módulo S3:

     ```bash
     mkdir -p ~/s3_logs_recovery
     touch ~/s3_logs_recovery/filebeat.yml
     ```

     ```yaml
     filebeat.inputs:
       - type: aws-s3
         id: s3-logs-historico # ID único para rastrear estado (obrigatório)
         bucket_arn: "${AWS_BUCKET_ARN}" # ARN do seu bucket
         # Ou use non_aws_bucket_name se for S3-compatible (ex: MinIO): non_aws_bucket_name: "seu-bucket"

         bucket_list_prefix: "logstash/2026/01/20/03/" # Lê tudo sob /logstash/ (anos/meses/dias/horas)
         # Para testar um dia específico (recomendado para backfill):
         # prefix: "logstash/2026/01/19/"

        include:
          - ".*2026-01-19.*"

     # region: "${AWS_REGION}" # Sua região

     # Autenticação (escolha uma):
     # Opção 1: IAM Role (melhor se Filebeat roda em EC2/ECS/EKS)
     # (não precisa de keys)

     # Opção 2: Chaves explícitas (use env vars para segurança)
     access_key_id: "${AWS_ACCESS_KEY_ID}"
     secret_access_key: "${AWS_SECRET_ACCESS_KEY}"
     session_token: "${AWS_SESSION_TOKEN}" # Se usar credenciais temporárias (recomendado para segurança)
     # credential_profile_name: "${AWS_PROFILE}" # Se usar profiles do AWS CLI (opcional, mas recomendado para segurança)
     # shared_credential_file: /root/.aws # Caminho para o arquivo de credenciais (padrão do AWS CLI)

     # Opção 3: Profile AWS CLI
     # credential_profile_name: "default"

     # Configurações para histórico/one-shot
     number_of_workers: 4 # Quantos workers paralelos (aumente para mais velocidade)
     bucket_list_interval: 300s # Intervalo entre listagens (aumente para menos CPU)
     polling_interval: 60s # Não afeta muito em one-shot

     # Codec e parsing
     # codec: json # Se forem JSON (um evento por arquivo ou por linha? Ajuste)
     # Se forem JSON lines (um por linha):
     codec:
          json:
          keys_under_root: true # Coloca campos no root do evento
     # Ou para texto plano:
     # codec: plain

     # Para evitar reprocessar (estado persistente)
     data_stream:
          dataset: "aws.s3.logs" # Ou customize: "custom.s3.logs"

     # Opcional: ignore arquivos antigos ou recentes
     # exclude_files: ['\.gz$']     # Ex: ignore gzip se não quiser
     # max_file_age: 30d            # Não leia arquivos >30 dias (para histórico)

     # o Filebeat aws-s3 input lê o conteúdo do arquivo S3 linha por linha (ou como um todo, dependendo do codec)
     # e coloca tudo no campo message como uma string raw (JSON stringificado).
     # Isso é o comportamento default quando não há parsing automático configurado para JSON lines
     # ou quando o codec não extrai os campos para o root do evento.
     processors:
     - decode_json_fields:
          fields: ["message"] # Campo que contém o JSON string
          target: "" # "" = coloca no root (nível raiz do evento)
          overwrite_keys: true # Sobrescreve campos existentes se conflito (ex: @timestamp)
          process_array: false # Se não for array
          max_depth: 1 # Profundidade máxima (aumente se JSON nested)
          add_error_key: true # Adiciona "error" se falhar parse

     - drop_event:
          when:
          not.equals: # "not" para DROPAR se NÃO for o valor desejado
               ServiceName: "prod-payments"

     - drop_fields:
          fields:
          - "message"
          - "log.offset"
          - "aws.s3.bucket.arn"
          - "aws.s3.bucket.name"
          - "aws.s3.object.etag"
          - "aws.s3.object.size"
          - "input.type"
          - "agent.ephemeral_id"
          - "agent.id"
          - "ecs.version"
          - "cloud.provider"
          - "cloud.region"
          ignore_missing: true

     # Template obrigatório quando index é customizado
     setup.template.enabled: true
     # setup.template.overwrite: true
     setup.template.name: "<template_name>-logs" # Nome base do template
     setup.template.pattern: "<template_name>-logs-*" # Pattern que cobre seus índices (%{+yyyy.MM.dd})
     # Aqui você define os settings do índice no template
     setup.template.settings:
     index:
     number_of_shards: 1 # 1 shard primário (padrão é 1, mas explicite)
     number_of_replicas: 0 # 0 réplicas → sem cópias, cluster fica green mesmo com 1 nó
     # Opcional: outras otimizações para one-shot
     refresh_interval: 30s # Mais lento que default (1s), mas menos overhead na ingestão
     codec: best_compression # Compressão melhor, economiza storage
     # Se volume muito pequeno (< alguns GB), pode até usar 1 shard total

     # Output para Elasticsearch (ajuste para seu cluster)
     output.elasticsearch:
     hosts: ["${ELASTIC_HOSTS}"]
     username: "${ELASTIC_USER}"
     password: "${ELASTIC_PASSWORD}"
     index: "<template_name>-logs"

     # Para evitar duplicatas (como você fez no Logstash)
     document_id: "%{EventUUID}" # Se o campo existir no JSON

     # Opcional: logging mais verbose para debug
     logging.level: info
     logging.to_files: true
     logging.files:
     path: /var/log/filebeat
     name: filebeat-s3.log

     ```

2. Crie o arquivo `.env` para armazenar suas variáveis de ambiente (opcional, mas recomendado para segurança):

     ```bash
     touch ~/s3_logs_recovery/.env
     ```

     Exemplo de conteúdo do `.env`:

     ```env
     export ELASTIC_HOSTS=""
     export ELASTIC_USER=""
     export ELASTIC_PASSWORD=""
     export LS_JAVA_OPTS="-Xms1g -Xmx2g"
     export AWS_ACCESS_KEY_ID=""
     export AWS_SECRET_ACCESS_KEY=""
     export AWS_SESSION_TOKEN="" # Se usar credenciais temporárias (recomendado para segurança)
     export AWS_PROFILE="" # Se usar profiles do AWS CLI (opcional, mas recomendado para segurança)
     export AWS_BUCKET_ARN=""
     export AWS_BUCKET_NAME=""
     export AWS_BUCKET_PREFIX=""
     export AWS_REGION=""
     ```

3. Crie o docker-compose para rodar o Filebeat.

     ```yml
     services:
       setup:
         image: docker.elastic.co/beats/filebeat:8.11.0
            command: >
               bash -c '
               if [ -z "$ELASTIC_USER" ]; then
                    echo "Set the ELASTIC_USER environment variable in the .env file";
                    exit 1;
               elif [ -z "$ELASTIC_PASSWORD" ]; then
                    echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
                    exit 1;
               elif [ -z "$ELASTIC_HOSTS" ]; then
                    echo "Set the ELASTIC_HOSTS environment variable in the .env file";
                    exit 1;
               else
                    echo "All required environment variables are set. Starting Filebeat...";
                    exit 0;
               fi
               '
          env_file:
             ./.env
       filebeat:
          container_name: filebeat
          user: root # Necessário para acessar o registry e logs
          command: filebeat -e -strict.perms=false # Evita erros de permissão no registry
          image: docker.elastic.co/beats/filebeat:8.11.0
          env_file:
             ./.env
          volumes:
            # - ~/.aws:/root/.aws:ro # Para usar profiles do AWS CLI (opcional, mas recomendado para segurança)
            - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
            - ./registry:/usr/share/filebeat/data/registry
          depends_on:
            - setup
     ```

4. Inicie o Filebeat com Docker Compose:

     ```bash
     docker-compose up
     ```

**Estrutura de Arquivos:**

```bash
s3_logs_recovery/
├── .env
├── docker-compose.yml
├── filebeat.yml
└── registry/ # Pasta local para o registry do Filebeat (estado de leitura)
```