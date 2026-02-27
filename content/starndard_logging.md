---
title: Padronização de Logs
tags:
  - Logs
enableToc: true
---

Este documento serve como um guia abrangente para a criação, manutenção e análise de logs em sistemas de software. Logs são essenciais para monitorar o comportamento do sistema, diagnosticar problemas e garantir a segurança.

# Objetivo Principal

Todo log deve ser JSON estruturado (nada de texto livre).

Todo campo importante deve estar no nível raiz ou em objetos bem definidos para facilitar queries, dashboards e alerts no Elastic.

# Padrão Oficial de Logs (Kubernetes)

Todas as linhas de log DEVEM ser um JSON único com no mínimo os seguintes campos no nível raiz:

```json
{
  "@timestamp": "2025-12-01T11:30:14.058Z",     // ISO 8601 com milissegundos e timezone Z (obrigatório)
  "level": "INFO" | "WARN" | "ERROR" | "DEBUG" | "TRACE",  // Sempre presente e MAIÚSCULO (obrigatório)
  "thread": "32080" | "http-nio-8080-exec-1" | etc,        // nome ou id da thread (Opcional)
  "logger": "com.busines.acquiring.capture.ecommerceservice.EcommerceServiceImpl", // nome completo da classe (Java) ou módulo (NodeJS, Python, etc) que gerou o log (obrigatório)
  "service": "acquiring-ecommerce",                // nome curto do serviço/microserviço (Obrigatório se não for hospedado no Kubernetes)
  "environment": "prod" | "hml" | "dev",           // Opcional, mas recomendado
  "trace_id": "787a5a47-604d-4a27-99cc-5e36bf25873b",   // correlation ID / trace_id (OpenTelemetry, MDC, Elastic APM) (Necessário para rastreamento distribuído em que a intrumentação esteja em modo manual)
  "span_id": "3f2d1e9b4c8a7201",                   // opcional, se tiver tracing (OpenTelemetry, MDC, Elastic APM) (Necessário para rastreamento distribuído em que a intrumentação esteja em modo manual)
  "event":{
     "category": "BUSINESS" | "Ler sobre o campo category", // obrigatório e MAIÚSCULO
     "action": "authorize",       // opcional, mas recomendado
     "outcome": "success" | "failure" | "unknown" // opcional, mas recomendado
  },
  "message": "texto legível para humano (curto)", // opcional, mas recomendado, mas nunca com dados estruturados aqui. Este campo é para fins de leitura humana, não para restreabilização de dados.
}
```

Um campo muito importante é o `event.category`. Ele deve ser usado para categorizar o log em um dos seguintes tipos:

Caategria|Quando usar|Exemplos|
|---------|------------------------|-------------------------------|
| DATABASE | Qualquer interação com banco (conexão, query, pool, timeout, deadlock, migration) | HikariPool, Flyway, JDBC |
| CACHE | Redis, Caffeine, Hazelcast — get/set/evict/miss/hit | RedisTemplate, CacheManager |
| MESSAGE_BROKER | Kafka, RabbitMQ, Google Pub/Sub — produce/consume/ack/nack | KafkaTemplate, listener |
| EXTERNAL_API | Chamada para qualquer API externa | RestTemplate, WebClient |
| GRPC | Chamadas gRPC internas ou externas | gRPC client/server |
| HTTP | Chamadas HTTP internas (outbound) ou entrada no nosso serviço | WebClient, Feign, Controller |
| BUSINESS | Regra de negócio, validação de cartão, antifraude, cálculo de taxa, conciliação | | authorize(), capture(), cancel() |
| AUTHENTICATION | Login, JWT validation, 2FA, sessão | Spring Security, OAuth2 |
| AUTHORIZATION | Permissão de franquia, perfil, feature flag | @PreAuthorize, regras de acesso |
| APPLICATION | Startup, shutdown, healthcheck, actuator, métricas de JVM | main(), server started, liveness |
| SECURITY | Eventos de segurança, seja tentativas de ataque, ou até mesmo falhas de seguranca no fluxo da regra de negócio | Tentativa de SQL Injection, XSS, CSRF, etc |
| AUDIT | Quem fez o quê (ex: franquia X alterou config de taxa) — deve ser imutável | Log de auditoria de admin |

# Exemplos de output logs

## Sucesso

## Log de Negócio

```json
{
  "@timestamp": "2025-12-01T11:30:14.058Z",
  "level": "INFO",
  "thread": "http-nio-8080-exec-1",
  "logger": "com.busines.acquiring.capture.ecommerceservice.EcommerceServiceImpl",
  "service": "acquiring-ecommerce",
  "environment": "prod",
  "trace_id": "787a5a47-604d-4a27-99cc-5e36bf25873b",
  "span_id": "3f2d1e9b4c8a7201",
  "event": {
    "category": "BUSINESS",
    "action": "authorize",
    "outcome": "success"
  },
  "message": "Transação autorizada com sucesso para o pedido 12345"
}
```

O Examplo acima mostra um log de sucesso para uma transação de autorização de pagamento. Note que todos os campos obrigatórios estão presentes e o `event.category` está definido como `BUSINESS`. Vale ressaltar que pode haver cenários em que, por exemplo, desejamos rastrear qual foi a bandeira utilizada, valor da transação, meio de pagamento, etc. Esses dados estruturados adicionais devem ser colocados em campos separados, não no campo `message`. Por exemplo:

```json
{
     "@timestamp": "2025-12-01T11:30:14.058Z",
     "level": "INFO",
     "thread": "http-nio-8080-exec-1",
     "logger": "com.busines.acquiring.capture.ecommerceservice.EcommerceServiceImpl",
     "service": "acquiring-ecommerce",
     "environment": "prod",
     "trace_id": "787a5a47-604d-4a27-99cc-5e36bf25873b",
     "span_id": "3f2d1e9b4c8a7201",
     "event": {
          "category": "BUSINESS",
          "action": "authorize",
          "outcome": "success",
     },
     "extra": {
          "order_id": "12345",
          "amount": 150.75,
          "currency": "BRL",
          "payment_method": "CREDIT_CARD",
          "brand": "VISA"
     },
     "message": "Transação autorizada com sucesso"
}
```

Observe que foi inserido um campo novo chamado `extra` para armazenar dados adicionais relacionados à transação, mantendo o campo `message` limpo e legível. A escolha desse campo é apenas recomendação; você pode nomeá-lo conforme a conveniência do seu projeto, desde que mantenha a estrutura clara e organizada.

Vale ressaltar que essa padronização cabe o time de desevolvimento decidir como implementar. Não faz sentido ter duas ou mais aplicações que operam de formas semelhantes, ter suas prorpia estrutura de logs. O ideal é que o time defina uma única forma de logar, e a utilize em todas as aplicações que desenvolverem. Isso facilita a manutenção, o entendimento e a análise dos logs posteriormente.

Quando esses logs forem enviados para o Elastic, será possível criar dashboards, alertas e queries a partir desses campos estruturados, facilitando o monitoramento e a análise do sistema.

O elastisearch consegue mapear automaticamente os campos JSON, então não é necessário criar mappings manuais para esses campos, a menos que haja uma necessidade específica de customização.

Segue exemplo de logs que o elastic não vai conseguir mapear automaticamente:

```json
{
     "@timestamp":"2025-12-01T18:10:49.579Z",
     "thread":2623,
     "level":"SEVERE",
     "message":"CancelRentalSubscription-eeadf6b0-1644-45ce-99b8-39b5fd06036d from request=device_number: "******" store_document_number: "****" franchisee_document_number: "*****" status_for_inventory_update: DEVICE_INVENTORY_ITEM_STATUS_AVAILABLE",
     ...
}
```

O campo message contem dados estruturado, mas não está em JSON. Se passarmos esses campos para uma ferramenta de busca como o Kibana, não será possível filtrar por esses dados, pois o Elastic não vai conseguir mapear esses campos automaticamente. Por exemplo, se eu quiser listar apenas 0000000001 e 0000000002 no campo store_document_number, não vou conseguir fazer isso, pois o Elastic não vai entender que esse campo existe.

## Logs de job

```json
{
  "@timestamp": "2025-12-01T11:31:00.789Z",
  "level": "INFO",
  "thread": "QuartzScheduler-thread-1",
  "logger": "com.busines.acquiring.jobs.DailyReconciliationJob",
  "service": "acquiring-ecommerce",
  "environment": "prod",
  "event": {
    "category": "APPLICATION",
    "action": "daily_reconciliation",
    "outcome": "success"
  },
  "message": "Job de reconciliação diária concluído com sucesso"
}
```

## Erro

### Log de Falha em API Externa

```json
{
  "@timestamp": "2025-12-01T11:32:45.123Z",
  "level": "ERROR",
  "thread": "http-nio-8080-exec-5",
  "logger": "com.busines.acquiring.capture.paymentgateway.PaymentGatewayClient",
  "service": "acquiring-ecommerce",
  "environment": "prod",
  "trace_id": "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6",
  "span_id": "9f8e7d6c5b4a3210",
  "event": {
    "category": "EXTERNAL_API",
    "action": "call_payment_gateway",
    "outcome": "failure"
  },
  "error": { // campo adicional para detalhes do erro
    "type": "HttpTimeoutException",  // tipo ou classe do erro
    "message": "Timeout ao chamar a API do gateway de pagamento", // mensagem de erro
    "stack_trace": "com.busines.acquiring.capture(PaymentGatewayClient.java:45)..." // stack trace completo ou parcial
  },
  "message": "Falha ao chamar a API do gateway de pagamento"
}
```

No exemplo acima, temos um log de erro que captura uma falha ao chamar uma API externa (gateway de pagamento). Note que o campo `error` foi adicionado para fornecer detalhes específicos sobre o erro ocorrido, incluindo o tipo, a mensagem e a stack trace. Isso facilita a análise e o diagnóstico do problema. Cada linguagem de programação pode ter sua própria convenção para representar erros, mas o importante é manter a estrutura clara e consistente.

### Conexao com o banco de dados

```json
{
  "@timestamp": "2025-12-01T11:35:22.456Z",
  "level": "ERROR",
  "thread": "HikariPool-1-Connection-1",
  "logger": "com.busines.acquiring.database.DataSource",
  "service": "acquiring-ecommerce",
  "environment": "prod",
  "trace_id": "z9y8x7w6-v5u4-t3s2-r1q0-p9o8n7m6l5k4",
  "span_id": "0a1b2c3d4e5f6789",
  "event": {
    "category": "DATABASE",
    "action": "connect",
    "outcome": "failure"
  },
  "error": {
    "type": "SQLException",
    "message": "Falha ao conectar ao banco de dados: timeout de conexão",
    "stack_trace": "com.busines.acquiring.database.DataSource.getConnection(DataSource.java:78)..." 
  },
  "message": "Erro ao tentar conectar ao banco de dados"
}
```

O campo `error` é opcional, mas altamente recomendado para logs de erro, pois fornece informações detalhadas que podem ser cruciais para a resolução de problemas.

Vale ressaltar que, dependendo da linguagem e do framework utilizado, a forma de capturar e estruturar esses logs pode variar. O importante é seguir o padrão JSON estruturado e incluir os campos essenciais para garantir a consistência e a facilidade de análise dos logs.

---

# Considerações Finais

A pesar de este documento fornecer um guia abrangente para a criação e manutenção de logs, é importante lembrar que a implementação prática pode variar dependendo das necessidades específicas do projeto e das ferramentas utilizadas. A chave para um sistema de logging eficaz é a consistência e a clareza na estrutura dos logs, permitindo uma análise eficiente e uma rápida resolução de problemas.
