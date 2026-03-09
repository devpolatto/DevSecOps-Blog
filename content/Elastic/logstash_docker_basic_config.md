---
title: Básico de Configuração do Logstash no Docker
tags:
  - Logstash
enableToc: true
---

O Logstash é uma ferramenta de processamento de dados que pode ser usada para coletar, transformar e enviar dados para o Elasticsearch. Configurar o Logstash no Docker é uma maneira eficiente de gerenciar sua infraestrutura de processamento de dados. Aqui está um guia básico para configurar o Logstash no Docker:

# Estrutura de pasta

Primeiro, crie uma estrutura de pasta para organizar seus arquivos de configuração do Logstash:

```shell
logstash-docker
    ├── config
    │   ├── GeoLite2-City.mmdb # Opcional: banco de dados de geolocalização
    │   ├── logstash.yml # Configurações principais do Logstash
    │   └── pipelines.yml # Configurações de pipelines do Logstash
    ├── pipeline
    │   ├── index_mapping.tpl # Opcional: template de mapeamento para índices do Elasticsearch
    │   └── logs.conf # Pipeline do Logstash
    ├── docker-compose.yml
```

# Configuração do Logstash

`logstash-docker/config/logstash.yml`

Configure as opções básicas do Logstash, como host, porta, formato de log e monitoramento:

```yaml
http.host: 0.0.0.0
http.port: 9600
log.format: json
log.level: info
xpack.monitoring.enabled: false
```

`logstash-docker/config/pipelines.yml`

O logstash pode ter múltiplos pipelines. Aqui, definimos um pipeline chamado "main" que aponta para a pasta onde os arquivos de configuração do pipeline estão localizados:

```yaml
- pipeline.id: main
  path.config: /usr/share/logstash/pipeline
```

Para mais informações sobre as opções de configuração, consulte a [documentação oficial do Logstash](https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html).

`logstash-docker/pipeline/logs.conf`

Este é um exemplo básico de configuração de pipeline do Logstash.

```conf
input {
    kafka {
        bootstrap_servers => "${KAFKA_BOOTSTRAP_SERVERS}"
        topics => [ "${KAFKA_TOPIC}" ]
        security_protocol => "SASL_SSL"
        sasl_mechanism => "SCRAM-SHA-512"
        sasl_jaas_config => 'org.apache.kafka.common.security.scram.ScramLoginModule required username="${KAFKA_USER}" password="${KAFKA_USER_PASSWORD}";'
        auto_offset_reset => "earliest"
        consumer_threads => "${KAFKA_CONSUMER_THREADS}"
        group_id => "${GROUP_ID}"
    }
}

filter {
  json { source => "message" }

  mutate {
    remove_field => [
      "event", "agent", "host",
      "docker", "log", "[container][labels]", "@timestamp",
      "@version", "ecs", "input",
      "stream"
    ]
  }

  date {
    match => [ "[data][time_iso8601]", "ISO8601" ]
  }

  mutate {
    add_field => {
      "[@metadata][IndexName]" => "%{[vmss_name]}"
    }
  }

  cidr {
    address => [ "%{[data][remote_addr]}" ]
    network => [ "10.0.2.0/27" ]
    add_field => { "cidr" => "privateIP" }
  }  

  # Opcional: Se o endereço IP não for privado, adicione informações de geolocalização usando o banco de dados GeoLite2. Certifique-se de ter o arquivo GeoLite2-City.mmdb na pasta de configuração do Logstash.
  if "privateIP" not in [cidr] {
    geoip {
      source => "[data][remote_addr]"
      database => "/usr/share/logstash/config/GeoLite2-City.mmdb"
      target => "geoip"
      remove_field => [
        "[geoip][continent_code]", "[geoip][timezone]", "[geoip][postal_code]",
        "[geoip][country_code3]", "[geoip][country_name]", "[geoip][ip]",
        "[geoip][coordinates]", "[geoip][dma_code]"
      ]
    }
  }

  mutate {
    remove_field => [ "cidr" ]
  }
}

output {
  elasticsearch {
    hosts => [ "${ELASTIC_HOSTS}" ]
    index => "%{[@metadata][IndexName]}-%{+YYYY.MM}"
    user => "${ELASTIC_USER}"
    password => "${ELASTIC_PASSWORD}"
  }

  # Utilizado para debug, pode ser removido em produção. Ajuda a visualizar os dados que chegando no Logstash.
  stdout {
    codec => rubydebug {
      metadata => true
    }
  }
}
```

# Configuração do Docker Compose

`logstash-docker/docker-compose.yml`

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
      - ./config:/usr/share/logstash/config
      - ./pipeline/logs.conf:/usr/share/logstash/pipeline/logs.conf
    depends_on:
      - setup
```

# Variáveis de ambiente

`logstash-docker/.env`

```env
export KAFKA_BOOTSTRAP_SERVERS="b-1-public:9196,b-2-public:9196"
export KAFKA_TOPIC_REDSYS=""
export KAFKA_USER=""
export KAFKA_USER_PASSWORD=""
export KAFKA_CONSUMER_THREADS="3"
export GROUP_ID=""
export ELASTIC_HOSTS=""
export ELASTIC_USER=""
export ELASTIC_PASSWORD=""
export LS_JAVA_OPTS="-Xms1g -Xmx2g"
```