---
title: Padronização de Index Templates e Controle de Shards no Elasticsearch
tags:
  - Elasticsearch
enableToc: true
---
# 📌 Contexto

Este documento descreve o problema identificado no cluster Elasticsearch, a solução adotada através de **Index Templates**, e as remediações aplicadas para controlar o crescimento de shards e estabilizar a alocação no cluster.

O ambiente em questão utiliza:
- Kubernetes
- Fluentbit/Kafka/Logstash para ingestão de logs
- Elasticsearch como datastore
- Criação dinâmica de índices por aplicação/container

---
# 🚨 Problema Inicial

Cada nova aplicação no cluster Kubernetes que envia logs para o Elasticsearch estava criando **novos índices mensais**, com a seguinte configuração padrão:

- `number_of_shards: 3`
- `number_of_replicas: 1`

Exemplo de índice criado automaticamente:

```json
{
  "index": "accounting-api-hub-ctn-2025.12",
  "number_of_shards": "3",
  "number_of_replicas": "1"
}
```
### Impactos observados

- Cada índice consumia **6 shards** (3 primários + 3 réplicas)
- Crescimento acelerado do número total de shards
- Risco de atingir o limite operacional do cluster (~3000 shards)
- Problemas de alocação de shards e instabilidade do cluster

A causa raiz identificada foi a existência de **index templates específicos por aplicação**, que forçavam a criação de índices com 3 shards.

# Pipeline do Logstash

```json
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
		# Converte o campo `message` (string) em um objeto JSON.
		# Os campos do JSON passam a existir no evento.

    json { source => "message" }

		# Normalização do campo de log
		# Se existir `logprocessed`, ele vira `log`.
		# Caso contrário, o campo `log` original é renomeado para `raw_log`.
		# Isso diferencia **logs já processados** de **logs crus**.
		
    if ([logprocessed]) {
        mutate { rename => ["logprocessed" , "log"] }
    } else {
        mutate { rename => ["log" , "raw_log"] }
    }

		# Definição do timestamp do evento
		# Define o `@timestamp` do evento com base em vários campos possíveis.
		# Prioriza timestamps vindos do próprio log.
		# Usa `docker_time` como fallback.
		# Se nada existir, mantém o `@timestamp` padrão.

    if ([log][timestamp]) {
        date { match => [ "[log][timestamp]", "ISO8601" ] }
    } else if ([log][EventTimeIso8601]) {
        date { match => [ "[log][EventTimeIso8601]", "ISO8601" ] }
    } else if ([log][time]) {
        date { match => [ "[log][time]", "ISO8601" ] }
    } else if ([docker_time]){
        date { match => [ "[docker_time]", "ISO8601" ] }
    } else {
        date { match => [ "[@timestamp]", "ISO8601" ] }
    }

    mutate {
        remove_field => [
            "[log][timestamp]",
            "[log][EventTimeIso8601]",
            "[log][time]",
            "[docker_time]"
        ]
        
        # Metadados de indexação (default)
        # Define **prefixo do índice** padrão.
        # Define **frequência mensal** por padrão.
        # Esses campos ficam em `@metadata` (não são enviados ao Elasticsearch).
        
        add_field => {
            "[@metadata][IndexPrefix]"    => "logstash"
            "[@metadata][IndexFrequency]" => "monthly"
        }
    }

    # Tratamento específico de Kubernetes
    if ([kubernetes]) {
        mutate {
		        # Prefixo do índice baseado no container
		        # Cada container pode gerar seu próprio índice.
            update       => {
                "[@metadata][IndexPrefix]"    => "%{[kubernetes][container_name]}"
            }
        }
					
				# Override via label Kubernetes
				# Se existir essa label no pod, ela **define explicitamente o índice.
				# Dá controle total via manifest do Kubernetes.
				
        if ([kubernetes][labels][acqio.logstash.elastic.co/IndexPrefix] and
            [kubernetes][labels][acqio.logstash.elastic.co/IndexPrefix] != "") {
            mutate {
                update => {
                    "[@metadata][IndexPrefix]" => "%{[kubernetes][labels][acqio.logstash.elastic.co/IndexPrefix]}"
                }
            }
        }

        mutate {
            update => {
                "[kubernetes][container_image]" => "%{[kubernetes][container_hash]}"
            }
            remove_field => [
                "[kubernetes][annotations]",
                "[kubernetes][container_hash]",
                "[kubernetes][docker_id]",
                "[kubernetes][labels][controller-uid]",
                "[kubernetes][labels][job-name]",
                "[kubernetes][labels][pod-template-hash]",
                "[kubernetes][labels][security.istio.io/tlsMode]",
                "[kubernetes][labels][service.istio.io/canonical-name]",
                "[kubernetes][labels][service.istio.io/canonical-revision]",
                "[kubernetes][pod_id]"
            ]
        }
    }

    # Tratamento do conteúdo da mensagem
    # Padroniza tudo para `log.message`.
    if ([log][message] or [log][Message]) {
        if ([log][Message]) {
            mutate {
                copy          => { "[log][Message]" => "[log][message]" }
                remove_field  => [ "[log][Message]" ]
            }
        }
				
				# Evita erro quando `message` é um objeto JSON.
        ruby {
            code => '
                if not event.get("[log][message]").is_a? String
                    event.set("[log][message]", event.get("[log][message]").to_json)
                end
            '
        }
    }

	# Se `log.message` for JSON válido:
		# Ele é parseado para `log.record`
		# O campo `log.message` é removido
	# Diferencia **mensagem textual** de **mensagem estruturada.

    if [log][message] {
       json {
           source => "[log][message]"
           target => "[log][record]"
           skip_on_invalid_json => "true"
       }
   }
	  if [log][record] {
       mutate {
           remove_field => [ "[log][message]" ]
       }
	  }
		
	# Tratamento especial para métricas
	# Logs e métricas ficam **separados em índices diferentes.
		
    if ([log][metric][@name] and [log][metric][@name] != "") {
        mutate {
            update      => {
                "[@metadata][IndexPrefix]"    => "metric-%{[log][metric][@name]}"
                "[@metadata][IndexFrequency]" => "monthly"
            }
        }

        if ([log][metric][@metadata][idx_split]) {
            mutate {
                update       => {
                    "[@metadata][IndexFrequency]" => "%{[log][metric][@metadata][idx_split]}"
                }
                remove_field => [ "[log][metric][@metadata]" ]
            }
        }
        mutate {
            rename => [ "[log][metric]", "[metric]" ]
        }
    }

    # Definição final do nome do índice
    # Exemplo de formatos:
	    # Daily: `app-2026.01.05`
	    # Monthly: `app-2026.01`
	    # Yearly: `app-2026`

	# O nome do índice é **dinâmico**.
	# Baseado em:
		# Container
		# Label Kubernetes
		# Tipo de dado (log ou métrica)
		# Frequência temporal

    mutate { lowercase => [ "[@metadata][IndexPrefix]", "[@metadata][IndexFrequency]" ] }
    mutate { replace => { "[@metadata][IndexDateFormat]" => "%{+YYYY.MM}" } }

    if [@metadata][IndexFrequency] == "daily" {
        mutate { replace => { "[@metadata][IndexDateFormat]" => "%{+YYYY.MM.dd}" } }
    } else if [@metadata][IndexFrequency] == "monthly" {
        mutate { replace => { "[@metadata][IndexDateFormat]" => "%{+YYYY.MM}" } }
    } else if [@metadata][IndexFrequency] == "yearly" {
        mutate { replace => { "[@metadata][IndexDateFormat]" => "%{+YYYY}" } }
    } else {
        mutate { replace => { "[@metadata][IndexDateFormat]" => "%{+xxxx.ww}" } }
    }

    mutate {
        add_field    => { "[@metadata][IndexName]" => "%{[@metadata][IndexPrefix]}-%{[@metadata][IndexDateFormat]}" }
        remove_field => [ "message" ]
    }
}

output {
    elasticsearch {
        hosts => [ "${ELASTICSEARCH_HOST}" ]
        ssl => true
        index => "%{[@metadata][IndexName]}"
        user => "${ELASTICSEARCH_USER}"
        password => "${ELASTICSEARCH_PASSWORD}"
    }
}
```

# Como funcionam os Index Template
## Conceito

**Index Templates** são aplicados **somente no momento da criação do índice**

Eles definem:

- Settings (shards, replicas, ILM, etc.)
- Mappings
- Aliases

Após o índice ser criado:

- Não existe vínculo persistente entre o índice e o template
- As configurações são apenas copiadas para o índice

## Resolução de conflitos

Quando múltiplos templates combinam com o nome de um índice:

- Todos os templates compatíveis são avaliados
- O template com **maior `priority`** é aplicado
- Não há merge de `number_of_shards`

# 🛠️ Solução Implementada

Foi criado um **Index Template global para logs Kubernetes**, com prioridade maior que os templates existentes.

### Template aplicado

```shell
PUT _index_template/logstash-kubernetes-logs-template
{
  "index_patterns": ["*-ctn-20*.*"],
  "priority": 700,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "expire_index_after_30d"
    }
  }
}
```

### Resultado

Novos índices passam a ser criados com:
  
- ✅ 1 shard primário
- ✅ 1 réplica
- Redução imediata de ~66% no consumo de shards por índice
- Nenhum impacto em índices já existentes
- Compatível com políticas de ILM já em uso

# 🔎 Validação e Evidências

### Simulação oficial do Elasticsearch

```shell
POST _index_template/_simulate_index/meu-custom-index-ctn-2026.12
```

A resposta confirma:

- O template aplicado
- A prioridade vencedora
- Os settings finais do índice

> ⚠️ O Elasticsearch **não mantém referência** ao template após a criação do índice.  
> A simulação é a evidência oficial do mecanismo de decisão.

## Validação do índice criado

```shell
PUT meu-custom-index-ctn-2026.12
```

```shell
GET meu-custom-index-ctn-2026.12/_settings
```

```shell
{
  "meu-custom-index-ctn-2026.12": {
    "settings": {
      "index": {
        "lifecycle": {
          "name": "expire_index_after_30d"
        },
        "routing": {
          "allocation": {
            "include": {
              "_tier_preference": "data_content"
            }
          }
        },
        "number_of_shards": "1",
        "provided_name": "meu-custom-index-ctn-2026.12",
        "creation_date": "1767879001169",
        "priority": "100",
        "number_of_replicas": "1",
        "uuid": "J614BOqkQm20EtCVKpn8oQ",
        "version": {
          "created": "8500003"
        }
      }
    }
  }
}
```

# 🧹 Remediações Aplicadas

## Curto prazo

- Criação de template com prioridade maior
- Correção automática do número de shards para novos índices
## Médio prazo (recomendado)

- Revisar e consolidar templates específicos por aplicação
- Evitar criação automática de templates por app/container
- Centralizar templates por tipo de dado (logs, métricas)
## Boas práticas adotadas

- Controle de shards via Index Template
- Uso de ILM para retenção automática
- Separação clara de responsabilidades:
	- Logstash → ingestão
	- Elasticsearch → governança de índices