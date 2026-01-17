---
title: Ordem correta dos componentes da Gateway API
tags:
  - Kubernets
  - Ingress
  - NGINX
enableToc: true
---
# 1. GatewayClass

A Gateway API NÃO funciona sozinho. Ele depende de um controlador de gateway para implementar a funcionalidade de rede subjacente. Portanto, o primeiro componente a ser criado é o GatewayClass, que define qual controlador de gateway será usado.

O gateway API é apenas:
- Um conjunto de CRDs (Custom Resource Definitions)
- Um modelo para definir como o tráfego deve ser roteado
- Um contrato entre os administradores de cluster e os usuários do cluster.

Ele não implementa a funcionalidade de rede por conta própria. O controlador de gateway é o componente que realmente lida com o tráfego de rede, aplicando as regras definidas na Gateway API.

Sem controllers, a Gateway API não pode funcionar, pois não há nada para interpretar e aplicar as regras de roteamento definidas nos recursos da Gateway API.

Resultando em:
- `GatewayClass` fica `Accepted=False`
- `Gateway` nunca fica `Ready=True`
- Nenhum tráfego flui

É exatamente o mesmo conceito do Ingress + Ingress Controller

## O papel do controller na Gateway API

O controller é responsável por:

| Função               | Quem faz   |
| -------------------- | ---------- |
| Criar LoadBalancer   | Controller |
| Configurar listeners | Controller |
| Programar rotas      | Controller |
| TLS / Certificados   | Controller |
| Reconciliar mudanças | Controller |


## Quais controllers existem hoje?

- **NGINX Gateway Fabric**

     - Mantido pela NGINX / F5
     - Substituto natural do ingress-nginx
     - Muito bom para migração de Ingress → Gateway API

     ✔️ Prós
     - Simples
     - Estável
     - Excelente documentação
     - Ótima compatibilidade com cert-manager
     - Performance alta

     ❌ Contras
     - Menos recursos L7 avançados que Istio
     - Ainda evoluindo (mas estável)

     👉 Controller name: `gateway.nginx.org/nginx-gateway-controller`

     Exemplo:

     ```yaml
     apiVersion: gateway.networking.k8s.io/v1
     kind: GatewayClass
     metadata:
       name: nginx
     spec:
       controllerName: gateway.nginx.org/nginx-gateway-controller
     ```

- **Envoy Gateway**

     - Mantido pela CNCF / Envoy
     - Baseado em Envoy Proxy

     ✔️ Prós
     - Extremamente poderoso
     - Observabilidade (metrics, tracing)
     - Ideal para arquiteturas avançadas

     ❌ Contras
     - Mais complexo
     - Curva de aprendizado maior

     👉 Controller name: `gateway.envoyproxy.io/gateway-controller`

- **Istio (Gateway API mode)**

     - Service Mesh completo
     - Gateway API é só uma parte

     ✔️ Prós
     - Segurança (mTLS)
     - Traffic shaping
     - Canary / Blue-Green

     ❌ Contras
     - Overkill se você só quer ingress
     - Complexidade operacional
     👉 Controller name: `istio.io/gateway-controller`

- **Azure Application Gateway for Containers (AGC)**

     - Controller gerenciado pela Microsoft
     - Integrado com Azure Application Gateway

     ✔️ Prós
     - Integração nativa com Azure
     - Fácil de usar em clusters AKS
     - Suporte a WAF (Web Application Firewall)
     - Sem NGINX dentro do cluster

     ❌ Contras
     - Custo
     - Dependência do Azure
     - Menos flexível que outras opções

     👉 Controller name: `azure.applicationgateway.kubernetes.io/agc-controller`

- **Outros**

     - Existem outros controllers em desenvolvimento ou menos populares
     - Verifique a compatibilidade e suporte antes de escolher:

     | Controller   | Observação                  |
     | ------------ | --------------------------- |
     | Kong Gateway | Gateway API suporte parcial |
     | Traefik      | Suporte ainda incompleto    |
     | HAProxy      | Em evolução                 |

# 2. Gateway

Documentação oficial: https://gateway-api.sigs.k8s.io/guides/getting-started/simple-gateway/

O segundo componente a ser criado é o Gateway. O Gateway define os pontos de entrada para o tráfego na malha de rede do cluster Kubernetes. Ele especifica como o tráfego deve ser roteado para os serviços dentro do cluster.

Ele é o LoadBalancer, define portas de escuta (listeners), protocolos, TLS, etc. Substitui o antigo Ingress Controller Service.

Exemplo:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: 'false'
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: ''
    service.beta.kubernetes.io/azure-load-balancer-ipv4: 172.179.160.62
  creationTimestamp: '2026-01-16T18:57:55Z'
  generation: 1
  name: nginx-public-gateway
  namespace: nginx-gateway
  resourceVersion: '78532'
  uid: 99a99bba-cb28-477a-b498-fa3802f373d4
  selfLink: >-
    /apis/gateway.networking.k8s.io/v1/namespaces/nginx-gateway/gateways/nginx-public-gateway
status:
  addresses:
    - type: IPAddress
      value: 172.194.144.0
spec:
  gatewayClassName: nginx
  listeners:
    - allowedRoutes:
        namespaces:
          from: Same
      name: http
      port: 80
      protocol: HTTP
    - allowedRoutes:
        namespaces:
          from: Same
      name: https
      port: 443
      protocol: HTTPS
      tls:
        certificateRefs:
          - group: ''
            kind: Secret
            name: agent-tls
        mode: Terminate
```

```shell
kubectl get svc -n nginx-gateway --kubeconfig .kube/test-kubeconfig

NAME                         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
nginx-gateway-fabric         ClusterIP      10.0.130.161   <none>          443/TCP                      63m
nginx-public-gateway-nginx   LoadBalancer   10.0.5.77      172.194.144.0   80:32514/TCP,443:31334/TCP   22m
```

# 3. HTTPRoute / TCPRoute / UDPRoute

Documentação oficial: https://gateway-api.sigs.k8s.io/guides/http-routing/

O terceiro componente a ser criado são as rotas (HTTPRoute, TCPRoute, UDPRoute). Essas rotas definem como o tráfego deve ser roteado para os serviços dentro do cluster Kubernetes com base em regras específicas. Elas são vinculadas ao Gateway e especificam os destinos finais do tráfego.

🧠 Resumo mental
```shell
CRDs
 ↓
Controller (NGINX Gateway Fabric)
 ↓
GatewayClass   ← contrato
 ↓
Gateway        ← load balancer
 ↓
HTTPRoute      ← rotas
 ↓
Service        ← backend
```

Exemplo de HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: acqio-route
  namespace: default
spec:
  parentRefs:
  - name: nginx-public-gateway
    namespace: nginx-gateway
  hostnames:
  - acqio.net
  - www.acqio.net
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: acqio-service
      port: 80
```