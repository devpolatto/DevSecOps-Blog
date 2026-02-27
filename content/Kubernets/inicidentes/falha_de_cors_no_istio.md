---
title: Falha de CORS ao acessar um serviço exposto pelo Istio
tags:
  - Kubernetes
  - Istio
  - CORS
  - VirtualService
  - AuthorizationPolicy
enableToc: true
---

# Descrição do Problema

CORS Error em endpoint específico (PATCH) do serviço admin-api-hub – **Falha de Preflight OPTIONS bloqueada por AuthorizationPolicy Istio**

Um único endpoint (/api/users/useradminaccount/{id}/useradminroleids) estava retornando erro de CORS apenas no browser (Firefox/Chrome) quando chamado via PATCH.

Todos os outros endpoints (principalmente GETs) funcionavam normalmente.

O navegador identificou uma falha de CORS (Cross-Origin Resource Sharing) ao tentar acessar o endpoint `/api/users/useradminaccount/<id>/useradminroleids` do serviço `admin-api-hub-dpl`. A resposta do servidor retornou um código de status HTTP 401 (Unauthorized) com detalhes de resposta indicando "ext_authz_denied".

# Causa raiz identificada

O preflight (requisição de verificação prévia) OPTIONS (obrigatório para PATCH + Content-Type: application/json + Authorization) era rejeitado pela AuthorizationPolicy / ext_authz do Istio com 401 Unauthorized (response_code_details: ext_authz_denied / response_flags: UAEX).

Como a requisição era negada antes do filtro CORS do VirtualService ser totalmente aplicado, o Envoy não devolvia os headers Access-Control-Allow-*, fazendo o browser rejeitar a chamada como CORS failure.

# Remediação aplicada (funcionou imediatamente):
Adição da linha `allowCredentials: true` no corsPolicy do VirtualService admin-api-hub-vs.

# Investigação

1. **Browser DevTools (Firefox)**

     - Erro no console: típico “Response to preflight request doesn't pass access control check”

     - Request Headers da chamada PATCH:

     ```txt
     PATCH /api/users/useradminaccount/{id}/useradminroleids undefined
     Host: admin-api-hub.admin.domain.net
     User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0
     Accept: application/json, text/plain, */*
     Accept-Language: en-US,en;q=0.9
     Accept-Encoding: gzip, deflate, br, zstd
     Content-Type: application/json
     Authorization: Bearer ************************
     Access-Control-Allow-Origin: *
     Access-Control-Allow-Headers: Authorization
     Access-Control-Allow-Methods: GET, POST, OPTIONS, PUT, PATCH, DELETE
     Content-Length: 127
     Origin: https://agenty.domain.net
     Connection: keep-alive
     Referer: https://agenty.domain.net/
     Sec-Fetch-Dest: empty
     Sec-Fetch-Mode: cors
     Sec-Fetch-Site: same-site
     ```

     ```txt
     OPTIONS /api/users/useradminaccount/{id}/useradminroleids HTTP/2
     Host: admin-api-hub.admin.domain.net
     User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0
     Accept: */*
     Accept-Language: en-US,en;q=0.9
     Accept-Encoding: gzip, deflate, br, zstd
     Access-Control-Request-Method: PATCH
     Access-Control-Request-Headers: access-control-allow-headers,access-control-allow-methods,access-control-allow-origin,authorization,content-type
     Referer: https://agenty.domain.net/
     Origin: https://agenty.domain.net
     Connection: keep-alive
     Sec-Fetch-Dest: empty
     Sec-Fetch-Mode: cors
     Sec-Fetch-Site: same-site
     Priority: u=4
     ```

2. **Logs do Pod**


     2.1. **Logs do container istio-proxy**

     - O preflight OPTIONS era bloqueado com 401 Unauthorized (response_code_details: ext_authz_denied / response_flags: UAEX).
     - Os GETs normais funcionavam (200 OK).
     - Requeste GET rejeitada com 401 Unauthorized (response_code_details: ext_authz_denied). A flag reforça que a requisição foi negada. Isso sugere que a requisição não atendeu os critérios de autorização definidos.

     ```json
     {"route_name":null,"authority":"admin-api-hub.admin.domain.net","request_id":"4eb2389f-79ee-4632-8869-63c3faad120e","upstream_local_address":null,"method":"GET","response_code":401,"protocol":"HTTP/2","start_time":"2026-02-20T13:55:15.080Z","bytes_sent":0,"duration":9,"upstream_cluster":"inbound|50051||","bytes_received":0,"response_flags":"UAEX","downstream_remote_address":"52.123.190.124:0","connection_termination_details":null,"path":"/api/users/useradminaccount/{id}/useradminroleids","user_agent":null,"requested_server_name":"outbound_.50051_._.admin-api-hub-svc.admin.svc.cluster.local","upstream_service_time":null,"downstream_local_address":"10.26.0.250:50051","upstream_host":null,"upstream_transport_failure_reason":null,"x_forwarded_for":"52.123.190.124","response_code_details":"ext_authz_denied"}
     ```

     ```json
     {"path":"/api/users/useradminaccount/<id>/useradminroleids","request_id":"a770f46f-4720-40c2-895a-d0a18035a796","response_code_details":"ext_authz_denied","upstream_transport_failure_reason":null,"duration":4,"upstream_host":null,"route_name":null,"x_forwarded_for":"52.123.190.124","requested_server_name":"outbound_.50051_._.admin-api-hub-svc.admin.svc.cluster.local","response_code":401,"downstream_local_address":"10.26.0.250:50051","method":"GET","authority":"admin-api-hub.admin.domain.net","upstream_local_address":null,"bytes_sent":0,"protocol":"HTTP/2","start_time":"2026-02-20T13:55:14.115Z","user_agent":"Mozilla/5.0 (Windows NT 6.1; WOW64) SkypeUriPreview Preview/0.5 skype-url-preview@microsoft.com","downstream_remote_address":"52.123.190.124:0","upstream_service_time":null,"bytes_received":0,"response_flags":"UAEX","upstream_cluster":"inbound|50051||","connection_termination_details":null}
     ```

     ```json
     {"connection_termination_details":null,"bytes_sent":289,"upstream_transport_failure_reason":null,"upstream_service_time":"37","downstream_local_address":"10.26.0.250:50051","request_id":"d4740cb1-3966-4c41-82be-f68613bd72bf","bytes_received":57,"upstream_host":"10.26.0.250:50051","downstream_remote_address":"170.78.98.38:0","start_time":"2026-02-20T14:27:44.533Z","user_agent":"PostmanRuntime/7.49.1","response_code_details":"via_upstream","duration":87,"authority":"admin-api-hub.admin.domain.net","route_name":"default","upstream_local_address":"127.0.0.6:58433","upstream_cluster":"inbound|50051||","protocol":"HTTP/2","method":"POST","path":"/api/users/useradminaccount/<id>/useradminroleids","x_forwarded_for":"170.78.98.38","response_flags":"-","requested_server_name":"outbound_.50051_._.admin-api-hub-svc.admin.svc.cluster.local","response_code":200}
     ```

3. **Configurações do Deployment, VirtualService, etc.**

     3.1 **Deployment**

     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
     name: admin-api-hub-dpl-6c87d59fbb-qr92r
     namespace: admin
          labels:
          aadpodidbinding: aad-pod-identity-selector
          app: admin-api-hub-app
          auth: enabled
          security.istio.io/tlsMode: istio
               service.istio.io/canonical-name: admin-api-hub-app
     spec:
     volumes: ...
     initContainers:
         - name: istio-init
          image: docker.io/istio/proxyv2:1.15.2
          args:
               - istio-iptables
               - '-p'
               - '15001'
               - '-z'
               - '15006'
               - '-u'
               - '1337'
               - '-m'
               - REDIRECT
               - '-i'
               - '*'
               - '-x'
               - ''
               - '-b'
               - '*'
               - '-d'
               - 15090,15021,15020
               - '--log_output_level=default:info'
     containers:
          - name: admin-api-hub-ctn
          image: devdomain.azurecr.io/admin/admin-api-hub:********
          args:
               - '--port=50051'
               - >-
                    --user-register-service-url=user-register-service-svc.admin.svc.cluster.local
               - >-
                    --rental-subscription-service-url=rental-subscription-service-svc.admin.svc.cluster.local
               - '--cerc-service-url=cerc-service-svc.acquiring.svc.cluster.local'
               - '--banking-service-url=banking-service-svc.acquiring.svc.cluster.local'
               - >-
                    --user-profile-management-service-url=users-service-svc.customers.svc.cluster.local
               - '--signup-service-url=signup-service-svc.customers.svc.cluster.local'
               - '--users-service-url=users-service-svc.customers.svc.cluster.local'
               - >-
                    --file-hub-service-url=file-hub-service-svc.acquiring.svc.cluster.local
               - '--user-entity-host=user-entity-service-svc.customers.svc.cluster.local'
               - '--report-service-url=report-service-svc.customers.svc.cluster.local'
               - >-
                    --terminal-management-entity-host=jacksonville-terminal-management-service-svc.admin.svc.cluster.local
               - >-
                    --price-campaign-service-url=price-campaign-service-svc.pricing.svc.cluster.local
          ports:
          - containerPort: 50051
               protocol: TCP
          - name: istio-proxy
          image: docker.io/istio/proxyv2:1.15.2
          args:
               - proxy
               - sidecar
               - '--domain'
               - $(POD_NAMESPACE).svc.cluster.local
               - '--proxyLogLevel=warning'
               - '--proxyComponentLogLevel=misc:error'
               - '--log_output_level=default:info'
               - '--concurrency'
               - '2'
          ports:
               - name: http-envoy-prom
                    containerPort: 15090
                    protocol: TCP
     ```

     - Os argumentos mostram que ela chama outros serviços internos via DNS do cluster (user-register, rental-subscription, cerc, banking, etc.).

     3.2 **VirtualService**

     ```yaml
     apiVersion: networking.istio.io/v1alpha3
     kind: VirtualService
     metadata:
     annotations: ...
     labels:
          app: admin-api-hub-app
          instance: admin-api-hub
          managed-by: Helm
          part-of: admin
          name: admin-api-hub-vs
          namespace: admin
     spec:
          gateways:
          - istio-system/istio-gateway-default
          hosts:
          - admin-api-hub.admin.domain.net
          http:
          - corsPolicy:
               allowHeaders:
                    - access-control-allow-headers
                    - access-control-allow-methods
                    - access-control-allow-origin
                    - authorization
                    - keep-alive
                    - user-agent
                    - cache-control
                    - content-type
                    - content-transfer-encoding
                    - custom-header-1
                    - x-accept-content-transfer-encoding
                    - x-accept-response-streaming
                    - x-user-agent
                    - x-grpc-web
                    - grpc-timeout
               allowMethods:
                    - DELETE
                    - POST
                    - GET
                    - PUT
                    - OPTIONS
               allowOrigins:
                    - exact: https://agenty.domain.net
               exposeHeaders:
                    - grpc-status
                    - grpc-message
               maxAge: 1728000s
               match:
               - uri:
                    prefix: /
               name: admin-api-hub-http
               route:
               - destination:
                    host: admin-api-hub-svc
                    port:
                    number: 50051
     ```

     - Metodos permitidos: GET, POST, PUT, DELETE, OPTIONS
     - Allow-Origin: https://agenty.domain.net (exato, sem wildcard)
     - Allow-Headers: Authorization, Content-Type, etc.
     - Expose-Headers: grpc-status, grpc-message (obrigatório para gRPC-Web)
     - Metodo OPTIONS (preflight) é tratado automaticamente pelo Envoy do Gateway, que devolve os headers CORS sem encaminhar para o pod. Já os métodos reais (GET/POST/PUT/etc.) são roteados para o Service admin-api-hub-svc:50051.

4. **Entende o fluxo completo (do browser até o pod) e o papel de cada componente (Ingress Gateway, VirtualService, Service, Sidecar Proxy).**

     4.1 **Fluxo geral**

     ```text
     Browser
          ↓ HTTPS (gRPC-Web)
     Ingress Gateway (Envoy)          ← aplica CORS + VirtualService
          ↓
     admin-api-hub-svc:50051          ← Service Kubernetes
          ↓
     Pod (admin-api-hub-dpl-xxx)
          ├── istio-proxy (sidecar)     ← porta 15006 (inbound)
          └── admin-api-hub-ctn         ← porta 50051 (sua app gRPC)
     ```

     4.2 **Passo a Passo Completo**

     1. Cliente (Browser) → Internet

          - O frontend em https://agenty.domain.net faz uma chamada gRPC-Web.
          gRPC-Web = gRPC “disfarçado” de HTTP (normalmente POST + headers especiais: x-grpc-web, grpc-timeout, content-type: application/grpc-web+proto, etc.).

          - O browser aponta para: https://admin-api-hub.admin.domain.net/...

     2. DNS + Load Balancer Externo

          - DNS resolve admin-api-hub.admin.domain.net → IP público do Istio Ingress Gateway.
          - O Azure Load Balancer entrega na porta 443 (HTTPS) do Gateway.

     3. Istio Ingress Gateway (o “Ingress” do Istio)

          - É um Envoy rodando no namespace istio-system (pod istio-ingressgateway-xxx).
          - Ele tem um recurso Gateway (istio-system/istio-gateway-default).
          - Esse Gateway define: porta 443, TLS, hosts permitidos, etc.
          - O Gateway recebe a requisição e procura qual VirtualService casa com o host admin-api-hub.admin.domain.net.

     4. VirtualService entra em ação (sua config)

          - hosts: - admin-api-hub.admin.domain.net
          - corsPolicy → se for OPTIONS (preflight), o Gateway responde imediatamente (sem ir pro pod). Ele devolve:
               - Access-Control-Allow-Origin: https://agenty.domain.net
               - Access-Control-Allow-Methods, Allow-Headers, exposeHeaders: grpc-status, grpc-message, etc.
          
          - Se for requisição real (GET/POST/PUT/etc.), o VirtualService faz route para admin-api-hub-svc:50051.

          ```yaml
          route:
          - destination:
               host: admin-api-hub-svc   # ← Service Kubernetes
               port: 50051
          ```

     5. Kubernetes Service (admin-api-hub-svc)

          - É um Service do tipo ClusterIP.
          - Ele aponta para todos os pods com label app: admin-api-hub-app.
          - O Envoy do Ingress Gateway faz a chamada para o IP do Service → porta 50051.

     6. Istio Sidecar Proxy (dentro do seu Pod) ← Ponto mais importante que muita gente não entende

          - Todo tráfego entrando no pod passa obrigatoriamente pelo container istio-proxy (o sidecar).
          - Como funciona tecnicamente:
               - O istio-init container (que roda no startup) configurou regras de iptables no pod.
               - Todo tráfego que chega na porta 50051 do pod é redirecionado para a porta 15006 do Envoy (inbound listener).

               Então a sequência dentro do pod é

               ```text
               Rede do cluster → iptables redirect → istio-proxy:15006
                              ↓
                         (aplica mTLS, AuthorizationPolicy, telemetry, etc.)
                              ↓
                         localhost:50051 → sua aplicação gRPC
               ```

               - O Envoy do sidecar entrega a requisição localmente para o container admin-api-hub-ctn que está escutando em 0.0.0.0:50051.

     7. Sua aplicação recebe

          - O gRPC server dentro do container recebe a requisição normalmente (como se tivesse vindo direto).
          - Responde → volta pelo mesmo caminho reverso (sidecar → gateway → browser).


# Remediação Aplicada

Alteração realizada no VirtualService admin-api-hub-vs (namespace admin):

```yaml
http:
- corsPolicy:
    allowOrigins:
    - exact: https://agenty.domain.net
    allowMethods:
    - DELETE
    - POST
    - GET
    - PUT
    - OPTIONS
    allowHeaders: [...]          # lista existente mantida
    exposeHeaders:
    - grpc-status
    - grpc-message
    maxAge: 1728000s
    allowCredentials: true       # ← LINHA ADICIONADA
  match:
  - uri:
      prefix: /
  route:
  - destination:
      host: admin-api-hub-svc
      port:
        number: 50051
```

**Resultado**: O endpoint passou a funcionar imediatamente (testado em aba anônima + cache limpo).

## Ações Recomendadas (Preventivas)

1. Criar AuthorizationPolicy para liberar OPTIONS em todos os serviços com auth habilitado:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-preflight-options
  namespace: admin
spec:
  selector:
    matchLabels:
      app: admin-api-hub-app   # ou usar label comum
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["OPTIONS"]
```

2. Padronizar allowCredentials: true em todos os VirtualServices que expõem APIs consumidas por frontend com JWT.

## Lições Aprendidas

- Preflight OPTIONS nunca carrega credenciais (Authorization, cookies, etc.). Qualquer política de auth deve explicitamente liberá-lo.
- Erro de CORS no browser é frequentemente máscara de outro erro (401/403/500) que acontece no preflight.
- Quando um endpoint específico falha e os demais funcionam → quase sempre é diferença entre “com preflight” (POST/PUT/PATCH) vs “sem preflight” (GET simples).
- `allowCredentials: true` é obrigatório sempre que o frontend envia `Authorization: Bearer`.
-  O CORS definido no VirtualService é global para todas as rotas que correspondem ao host, nesse caso, o https://admin-api-hub.admin.domain.net, não é possível ter regras de CORS diferentes para endpoints diferentes dentro do mesmo host. Se for necessário ter políticas de CORS distintas, seria necessário criar VirtualServices separados com hosts diferentes ou usar outras técnicas de roteamento.
- O preflight (ou "requisição de verificação prévia") é um mecanismo de segurança do navegador (Chrome, Firefox, Edge, etc.) que faz parte da especificação CORS (Cross-Origin Resource Sharing).<br/><br/>

     Quando o frontend (ex: https://agenty.domain.net) quer fazer uma requisição para um domínio diferente (ex: https://admin-api-hub.admin.domain.net), o navegador não envia diretamente a requisição "real" (o seu PATCH, POST, etc.). Em vez disso, ele primeiro envia uma requisição automática para "perguntar ao servidor se é permitido fazer isso".
     <br/><br/>
     Essa requisição de "pergunta" é chamada de preflight e sempre usa o método HTTP OPTIONS.
     <br/><br/>
     Quando o navegador dispara um Preflight?
     <br/><br/>
     O navegador considera a requisição "não simples" (non-simple) e dispara o preflight nas seguintes situações:

     | Condição | Exemplo nesse caso | Dispara preflight? |
     |----------|--------------------|--------------------|
     | "Método HTTP não é GET, HEAD ou POST" | "PATCH, PUT, DELETE" | Sim |
     | "Método é POST, mas Content-Type não é um dos ""simples"" (application/x-www-form-urlencoded, multipart/form-data, text/plain)" | "Content-Type: application/json" | Sim |
     | "Headers customizados (além dos básicos como Accept, Accept-Language, Content-Type simples, etc.)" | "Authorization: Bearer ..., x-grpc-web, etc." | Sim |
     | "Requisição inclui credentials (cookies, Authorization, etc.) com withCredentials: true" | "Sim, Bearer token" | Pode influenciar |

     **Como funciona o fluxo do Preflight (passo a passo)**? <br/><br/>

     O código frontend faz:
     
     ```txt
     fetch('https://admin-api-hub.../useradminroleids', { method: 'PATCH', headers: { Authorization: 'Bearer ...', 'Content-Type': 'application/json' }, body: {...} })
     ```

     O navegador não envia isso direto. Ele intercepta e envia primeiro:
     
     ```txt
     OPTIONS /api/users/useradminaccount/.../useradminroleids HTTP/2
     Host: admin-api-hub.admin.domain.net
     Origin: https://agenty.domain.net
     Access-Control-Request-Method: PATCH
     Access-Control-Request-Headers: authorization,content-type,...
     ```

     O servidor (nesse caso, o Istio Ingress Gateway + Envoy) responde ao OPTIONS com:

     - Status 200 (ou 204) se permitir
     - Headers obrigatórios:
          - Access-Control-Allow-Origin: https://agenty.domain.net (ou *)
          - Access-Control-Allow-Methods: PATCH, POST, GET, ...
          - Access-Control-Allow-Headers: authorization, content-type, ...
          - Access-Control-Allow-Credentials: true (se precisar de token/cookies)<br/>
<br/>
     Se a resposta for OK (200 + headers corretos), o navegador envia a requisição real (o PATCH com body e token). <br/><br/>
     Se não for OK (ex: 401, 403, ou faltar algum header CORS), o navegador bloqueia e mostra erro de CORS no console.