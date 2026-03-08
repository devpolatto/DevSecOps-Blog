---
title: Kafka - Daily commands
tags:
  - Kafka
enableToc: true
---

# Topicos

```shell
$KAFKA_ROOT/kafka-topics.sh \
--bootstrap-server $BOOTSTRAP_SERVER \
--command-config $CLIENT_CONFIG \
--list | describe | create | delete \
--topic <topic_name>
```

```shell
$KAFKA_ROOT/kafka-topics.sh \
--bootstrap-server $BOOTSTRAP_SERVER \
--command-config $CLIENT_CONFIG \
--create \
--topic <topic_name> \
--partitions 3 \
--replication-factor 3 \
--config min.insync.replicas=2 \
--config unclean.leader.election.enable=false \
--config cleanup.policy=delete \
--config compression.type=lz4
```

# Consumer

```shell
$KAFKA_ROOT/kafka-console-consumer.sh \
--bootstrap-server $BOOTSTRAP_SERVER \
--consumer.config $CLIENT_CONFIG \
--topic <topic_name> \
--group <group_name> \
--from-beginning \
--max-messages 5
```

# Producer

Produza uma mensagem JSON para o tópico usando o console producer do Kafka. Substitua `<topic_name>` pelo nome do tópico para o qual deseja enviar a mensagem.

```shell
echo '{"remote_addr":"168.63.129.16","remote_port":"58027","time_iso8601":"2026-02-13T03:08:53+00:00","request":"GET / HTTP/1.1","status": "200","bytes_sent":"147","http_user_agent":"Load Balancer Agent"}' \
  | $KAFKA_ROOT/kafka-console-producer.sh \
--producer.config $CLIENT_CONFIG \
    --bootstrap-server $BOOTSTRAP_SERVER \
    --topic <topic_name>
```