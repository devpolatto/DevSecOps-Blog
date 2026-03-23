---
title: Indexando logs do S3 com Logstash
description: Aprenda a configurar o Logstash para recuperar logs do S3 e indexá-los no Elasticsearch.
tags:
  - Elasticsearch
  - Logstash
  - S3
enableToc: true
---

## Introdução

Neste guia, vamos aprender como configurar o Logstash para recuperar logs armazenados no Amazon S3 e indexá-los no Elasticsearch. O Logstash é uma ferramenta poderosa para coletar, processar e transformar dados antes de enviá-los para o Elasticsearch.

## Pré-requisitos

Antes de começar, certifique-se de ter:

- Uma conta na AWS com acesso ao S3.
- O Logstash instalado em sua máquina (Para roda em docker, leia [[logstash_docker_basic_config|Básico de Configuração do Logstash no Docker]]).
- O Elasticsearch configurado e em execução.

## Passo a passo completo

1. Crie o arquivo de configuração do Logstash.

     ```bash
     mkdir -p ~/s3_logs_recovery
     touch ~/s3_logs_recovery/pipeline.conf
     ```

2. Crie o arquivo `logs.conf` com a seguinte configuração:

     ```conf
     input {
      s3 {
        access_key_id => "${AWS_ACCESS_KEY_ID}"
        secret_access_key => "${AWS_SECRET_ACCESS_KEY}"
        role_arn => "${AWS_ROLE_ARN}"                     # Se usar role assume, priorize isso (mais seguro)
        bucket => "${AWS_BUCKET_NAME}"
        region => "${AWS_REGION}"
        prefix => "${AWS_BUCKET_PREFIX}"                  # Ex: "logstash/2026/01/19/" para um dia específico
        codec => "json_lines"                             # Perfeito se forem arquivos .json com um evento por linha
        temporary_directory => "/tmp/logstash/"
        interval => 30                                    # Tempo entre polls (segundos)
        watch_for_new_files => false                      # false = processa tudo uma vez e para (ideal para backfill histórico)
        delete => false                                   # NÃO deleta nada do S3
        sincedb_path => "/var/lib/logstash/sincedb_s3"    # Persista esse arquivo em volume Docker para evitar reprocessar
        additional_settings => {
            "force_path_style" => false                     # Útil se bucket for path-style (raro hoje)
        }
      }
     }

     filter {
     # Adicione aqui se precisar parsear/extraír campos dos JSONs
     # Exemplo: json { source => "message" target => "parsed" } se não for json_lines
     }

     output {
        elasticsearch {
          hosts => [ "${ELASTIC_HOSTS}" ]                   # Verifique se é "htstp://elasticsearch:9200" ou URL completa
          index => "logs-%{+YYYY.MM.dd}"                     # Nome do índice, pode usar data para rotação
          user => "${ELASTIC_USER}"
          password => "${ELASTIC_PASSWORD}"
          document_id => "%{EventUUID}" # Se seus eventos tiverem um campo único, use aqui para evitar duplicatas
        }

     # Para debug, envie também para o console (opcional)
     stdout {
      codec => rubydebug {
        metadata => true
      }
     }
     }
     ```

     Defina também as configurações de Logstash no diretório `config` (opcional, mas recomendado para organização):

     ```bash
     mkdir -p ~/s3_logs_recovery/config
     touch ~/s3_logs_recovery/config/logstash.yml
     ```

     Exemplo de `logstash.yml`:

     ```yaml
     http.host: 0.0.0.0
     http.port: 9600
     log.format: json
     log.level: info
     xpack.monitoring.enabled: false
     ```

     Exemplo de config de pipeline (opcional, se quiser separar pipelines):

     ```bash
     - pipeline.id: main
       path.config: /usr/share/logstash/pipeline
     ```


3. Crie o arquivo `.env` para armazenar suas variáveis de ambiente sensíveis.

     ```bash
     touch ~/s3_logs_recovery/.env
     ```

     Adicione as seguintes variáveis ao arquivo `.env`:

     ```env
     AWS_ACCESS_KEY_ID=your_access_key_id
     AWS_SECRET_ACCESS_KEY=your_secret_access_key
     AWS_ROLE_ARN=arn:aws:iam::123456789012:role/your_role_name  # Se usar role assume, caso contrário deixe em branco
     AWS_BUCKET_NAME=your_bucket_name
     AWS_REGION=your_bucket_region
     AWS_BUCKET_PREFIX=optional_prefix_in_bucket/  # Ex: "logstash/2026/01/19/"
     ELASTIC_HOSTS=http://elasticsearch:9200  # Ou URL completa do seu Elasticsearch
     ELASTIC_USER=your_elastic_user
     ELASTIC_PASSWORD=your_elastic_password
     LS_JAVA_OPTS="-Xms1g -Xmx2g"
     ```

4. Crie o docker-compose para rodar o Logstash.

     ```bash
     touch ~/s3_logs_recovery/docker-compose.yml
     ```

    ```yaml
    services:
      setup:
        image: docker.elastic.co/logstash/logstash:8.11.0
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
              echo "All required environment variables are set. Starting Logstash...";
              exit 0;
            fi
          '
        env_file:
          - ./.env
      logstash:
        image: docker.elastic.co/logstash/logstash:8.11.0
        ports:
          - "5044:5044"
          - "9600:9600"
        env_file:
          - ./.env
        volumes:
          - ./logs.conf:/usr/share/logstash/pipeline/logs.conf
          - ./config:/usr/share/logstash/config
          - ./sincedb:/var/lib/logstash  # pasta local persistente para sincedb
        depends_on:
          - setup
    ```

5. Rode o Logstash usando o docker-compose.

     ```bash
     docker-compose up
     ```

**Estrutura do projeto:**

```bash
s3_logs_recovery/
├── .env
├── docker-compose.yml
├── logs.conf
├── config/
│   └── logstash.yml
│   └── pipelines.yml
└── sincedb/
```