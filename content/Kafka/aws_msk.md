---
title: Instruções para usar o Kafka client com AWS MSK
tags:
  - Kafka
  - AWS_MSK
enableToc: true
---

Para usar o Kafka client com o AWS MSK, é necessário baixar os binários do Kafka e configurar as variáveis de ambiente para apontar para o cluster e o arquivo de configuração do cliente.

Para instalar o Kafka CLI, siga os passos abaixo:

1. Recupere a versão do kafka do seu cluster MSK:

     Recupere o ARN do cluster MSK usando o comando `aws kafka list-clusters` e, em seguida, use o comando `aws kafka describe-cluster` para obter a versão do Kafka em execução no cluster. Substitua `<profile_name>` pelo nome do perfil AWS CLI que você está usando.

     Case queira configurar o perfil AWS CLI, acesse [[aws_cli_credentials|Configurando ambiente para usar o AWS CLI com credenciais]]

     ```shell
     CLUSTER_ARN=$(
     aws kafka list-clusters \
     --profile <profile_name> \
     --region us-west-2 \
     --query 'ClusterInfoList[?State==`ACTIVE`].{ARN:ClusterArn}' \
     --output text
     )
     ```

     ```shell
     KAFKA_VERSION=$(aws kafka describe-cluster \
     --profile <profile_name> \
     --region us-west-2 \
     --cluster-arn $CLUSTER_ARN \
     --query 'ClusterInfo.CurrentBrokerSoftwareInfo.KafkaVersion' \
     --output text)
     ```

     ```shell
     echo "Kafka version: $KAFKA_VERSION"
     ```

2. Baixe e extraia o Apache Kafka.

     ```shell
     wget https://archive.apache.org/dist/kafka/$KAFKA_VERSION/kafka_2.13-$KAFKA_VERSION.tgz
     tar -xzf kafka_2.13-$KAFKA_VERSION.tgz
     cd kafka_2.13-$KAFKA_VERSION
     export KAFKA_ROOT=$(pwd)
     ```

3. Configure a autenticação para seu cluster MSK.

     O arquivo client_admin.properties não aceita variáveis de ambiente, então é necessário colocar as informações de autenticação diretamente no arquivo. Substitua os valores de username e password pelas credenciais do seu cluster MSK.

     ```shell
     bootstrap.servers=""
     security.protocol=SASL_SSL
     sasl.mechanism=SCRAM-SHA-512
     sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="<username>" password="<password>";
     ssl.endpoint.identification.algorithm=https
     ```

4. (Opcional) Ajuste o tamanho do heap Java para ferramentas Kafka.

     ```shell
     export KAFKA_HEAP_OPTS="-Xms512M -Xmx512M"
     ```

5. Exporte as variáveis de ambiente para usar os comandos do Kafka CLI.

     ```shell
     export BOOTSTRAP_SERVER="<>:9093"
     export CLIENT_CONFIG="$(pwd)/client_admin.properties"
     export KAFKA_ROOT="$(pwd)/bin"
     export KAFKA_HEAP_OPTS="-Xms512M -Xmx512M"
     export KAFKA_OPTS="-Djava.security.manager=allow"
     ```