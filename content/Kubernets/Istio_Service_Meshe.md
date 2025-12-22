---
title: Istio Service Meshe
tags:
  - Kubernets
  - istio
  - ServiceMeshe
enableToc: true
---
O Istio é uma malha de serviço que aprimora a observabilidade, a segurança e o gerenciamento de tráfego para microsserviços em um cluster Kubernetes. Consegue-o injectando proxies sidecar Envoy em pods de aplicação e gerindo-os através de um plano de controlo.

O Istio é o caminho para o balanceamento de carga, a autenticação de serviço a serviço e o monitoramento - com poucas ou nenhuma alteração no código do serviço.

Ele oferece:

- Comunicação segura serviço-a-serviço em um cluster com criptografia TLS mútua, autenticação e autorização baseadas em identidade forte
- Balanceamento de carga automático para tráfego HTTP, gRPC, WebSocket e TCP
- Controle refinado do comportamento do tráfego com regras de roteamento avançadas, novas tentativas, failovers e injeção de falhas
- Uma camada de política conectável e API de configuração que suporta controles de acesso, limites de taxa e cotas
- Métricas, registros e rastreamentos automáticos para todo o tráfego em um cluster, incluindo entrada e saída de cluster

O Istio foi projetado para extensibilidade e pode lidar com uma gama diversificada de necessidades de implantação. O control plane do Istio é executado no Kubernetes e pode adicionar aplicações implementadas nesse cluster à sua rede, estender a rede a outros clusters ou até ligar VMs ou outros pontos finais executados fora do Kubernetes.

# Componentes base

Segue abaixo os componentes base que um cluster kubernets com istio configurado possui:
## namespace (istio-system)

O namespace istio-system é o espaço de nomes dedicado onde os componentes do plano de control plane (por exemplo, istiod) e os componentes de gateway (por exemplo, istio-ingressgateway) são implementados. Ele isola a infraestrutura do Istio das cargas de trabalho do aplicativo para melhor organização e segurança.

### labels

`istio-injection = “enabled”`: Ativa a injeção automática de sidecar para pods neste namespace (e potencialmente outros se rotulados de forma semelhante). Quando um pod é criado num namespace com esta etiqueta, o injetor sidecar do Istio (gerido pelo istiod) adiciona automaticamente um container proxy Envoy para lidar com o tráfego.

## istio-base

O chart istio-base implementa as Definições de Recursos Personalizados (CRDs) fundamentais necessárias para o Istio. Essas CRDs definem o esquema para os recursos de configuração do Istio, como VirtualService, DestinationRule, Gateway, ServiceEntry e PeerAuthentication. Sem estes CRDs, o plano de controlo do Istio (istiod) não pode interpretar ou aplicar configurações personalizadas.

Atua como a camada de pré-requisito para o Istio, permitindo que o cluster entenda os recursos específicos do Istio. É um componente leve, sem pods em execução, apenas extensões da API do Kubernetes.

## istiod

O istiod (Istio Daemon) é o componente central do plano de controlo do Istio. Combina várias funcionalidades anteriormente divididas em vários componentes (por exemplo, Pilot, Citadel, Galley em versões mais antigas). As suas principais responsabilidades incluem:

- **Gerenciamento de configuração**: Processa CRDs do Istio (por exemplo, VirtualService, DestinationRule) para configurar proxies Envoy para roteamento de tráfego, balanceamento de carga e políticas.
- **Injeção de sidecar**: Gerencia a injeção automática de proxies sidecar do Envoy em pods de aplicativos (com base no label `istio-injection=enabled`).
- **Gerenciamento de certificados**: Emite e gira certificados para TLS mútuo (mTLS) para proteger a comunicação serviço a serviço.
- **Descoberta de serviços**: Descobre serviços no cluster do Kubernetes e fornece essas informações aos proxies do Envoy.

## istio-ingressgateway

O istio-ingressgateway é um conjunto de pods proxy do Envoy que servem de ponto de entrada para o tráfego externo na rede de serviços. Ele lida com o tráfego de entrada (por exemplo, HTTP/HTTPS, TCP) e aplica as regras de roteamento do Istio definidas nos recursos Gateway e VirtualService.

Normalmente, é exposto através de um Serviço Kubernetes do tipo LoadBalancer ou NodePort, associado a um IP público

Ele roteia o tráfego externo para serviços internos com base nas configurações do Istio, oferecendo suporte a recursos como balanceamento de carga, terminação TLS e roteamento baseado em caminho.

---
# istio-injection

o label `istio-injection = “enabled”` é um mecanismo chave no Istio para integrar aplicações na rede de serviços. Desencadeia a injeção automática de sidecar, onde o Istio adiciona um container proxy Envoy aos pods em namespaces etiquetados com `istio-injection = “enabled”`. Esse proxy intercepta e gerencia todo o tráfego de rede de e para o pod, habilitando os recursos do Istio, como roteamento de tráfego, observabilidade e segurança, sem exigir alterações no código do aplicativo.

## O que é que a istio-injection faz?

- **Configuração em nível de namespace**: Quando um namespace tem  label `istio-injection = “enabled”`, qualquer pod criado nesse namespace recebe automaticamente um proxy sidecar Envoy injetado pelo plano de controlo do Istio (istiod).

- **Processo de injeção de sidecar**:
	- O servidor da API do Kubernetes notifica o webhook do Istio (gerido pelo istiod) quando um pod é criado num namespace rotulado.
	- O webhook **modifica a especificação do pod** para incluir um contentor proxy Envoy (e um initContainer chamado **istio-init** para configuração de rede).
	- O proxy do Envoy é configurado para intercptar todo o tráfego de entrada e saída do pod.
- **Estrutura do Pod modificado**: Um pod com o sidecar tem pelo menos dois containers:
	- O contentor da aplicação (a sua aplicação, por exemplo, um serviço Node.js ou Java).
	- O contentor do proxy Envoy (istio-proxy), que trata de todo o tráfego de rede.

## Injeção seletiva

No Istio, a injeção automática de sidecar é activada ao nível do namespace aplicando a etiqueta `istio-injection = “enabled”`, como visto no seu namespace istio-system. Isto faz com que o injetor sidecar do Istio (gerido pelo istiod) adicione automaticamente um contentor proxy Envoy a todos os pods criados nesse namespace. No entanto, a injeção selectiva permite um controle refinado ao sobrepor esta definição ao nível do namespace numa base por pod usando anotações. Isso é útil para cenários onde você quer que certos pods optem por não participar (ou participar) do service mesh, mesmo em um namespace com a injeção ativada.
### Como funciona a injeção selectiva

- **Injeção ao nível do espaço de namespaces**:
	- A label `istio-injection = "enabled"` num namespace (e.g., `kubectl label namespace my-namespace istio-injection=enabled`) diz ao Istio para injetar sidecars Envoy em todos os novos pods nesse namespace.
	- A injeção é realizada por um webhook (parte do istiod) que modifica as especificações do pod durante a criação.
- **Sobreposição ao nível do pod**:
	- Pode sobrepor a definição ao nível do namespace adicionando a anotação `sidecar.istio.io/inject` aos metadados de um pod.
		- Valores possíveis:
			- `sidecar.istio.io/inject: "false"`: Desactiva a injeção sidecar para o pod, mesmo que o namespace tenha `istio-injection = "enabled"`.
			- `sidecar.istio.io/inject: "true"`: Ativa a injeção sidecar para o pod, mesmo que o namespace não tenha `istio-injection = "enabled"`.
# Fluxo de Comunicação no Service Mesh

A aplicação não comunica diretamente com outros pods. Em vez disso, a comunicação é mediada por proxies Envoy. Aqui está como isso funciona:

- **Tráfego de saída**:
	- Sua aplicação (por exemplo, um microservice) envia uma solicitação (por exemplo, HTTP para outro serviço).
	- O pedido é **intercetado** pelo proxy Envoy local do pod (através de regras **iptables** definidas pelo container istio-init).
	- O proxy Envoy aplica as regras de encaminhamento do Istio (por exemplo, de VirtualService ou DestinationRule), trata a encriptação mTLS e encaminha o pedido para o proxy Envoy do pod de destino.
- **Tráfego de entrada**:
	- O proxy Envoy do pod de destino recebe o pedido, desencripta-o (se estiver a utilizar mTLS) e encaminha-o para o contentor da aplicação dentro do mesmo pod.
	- A aplicação processa o pedido e envia uma resposta de volta através do seu proxy Envoy local.
- **Comunicação proxy-to-proxy**:
	- Os proxies Envoy em ambos os pods comunicam entre si e não diretamente com os containers de aplicações. Isso garante que todo o tráfego seja gerenciado pela malha de serviço, permitindo recursos como:
		- **Gerenciamento de tráfego**: Balanceamento de carga, novas tentativas, tempos limite ou interrupção de circuitos.
		- **Segurança**: TLS mútuo (mTLS) para comunicação criptografada.
		- **Observabilidade**: Métricas, logs e traces para monitoramento (por exemplo, via Prometheus ou Jaeger).

![[assets/Istio_Service_Meshe.png]]
## Porquê utilizar um proxy?

- **Abstração**: O proxy Envoy abstrai a lógica de rede complexa, de modo que seu aplicativo não precisa implementar novas tentativas, mTLS ou rastreamento.
- **Consistência**: Todos os serviços na malha usam o mesmo proxy, garantindo um comportamento uniforme.
- **Controle**: o istiod configura os proxies Envoy dinamicamente com base nos CRDs do Istio, permitindo o gerenciamento centralizado do tráfego. 

# istio e Nginx ingress podem usar o mesmo IP válido?

Sim, é tecnicamente possível que o Istio Ingress Gateway e o controlador NGINX Ingress compartilhem o mesmo IP público, mas isso traz desafios e limitações significativas. Como alternativa, o provisionamento de um novo IP público para o Istio Ingress Gateway geralmente é a melhor abordagem para a maioria dos casos de uso devido à simplicidade, ao isolamento e à flexibilidade.

## Compartilhando o mesmo IP válido

Para compartilhar o mesmo IP público, tanto o controlador NGINX Ingress quanto o Istio Ingress Gateway precisariam ser configurados para usar o mesmo IP de front-end do Azure Load Balancer. No Kubernetes, isso é possível atribuindo o mesmo IP público aos seus respectivos serviços do LoadBalancer.

### Como funciona?

- Um balanceador de carga do Azure pode ter várias configurações de IP de front-end, mas um único IP de front-end pode ser compartilhado por vários serviços se eles usarem portas diferentes ou forem cuidadosamente configurados para rotear o tráfego corretamente.
- Você configuraria o serviço LoadBalancer do Istio Ingress Gateway para usar o mesmo IP público que o serviço do controlador NGINX Ingress (por exemplo, 172.179.50.13) definindo o campo loadBalancerIP.
- O tráfego para o IP partilhado é encaminhado para o NGINX ou para o Istio com base nas atribuições de portas ou nas regras do Load Balancer.

## Desafios da partilha do mesmo IP

- **Conflitos de portas**: O NGINX usa 80 e 443, portanto o Istio deve usar portas não padrão (por exemplo, 8080, 8443), o que pode ser inconveniente para os clientes ou exigir configurações adicionais de DNS.
- **Complexidade de encaminhamento**: O Balanceador de Carga do Azure tem de encaminhar o tráfego com base nas portas e tem de garantir que os clientes utilizam as portas corretas para as cargas de trabalho do Istio vs. NGINX.
- **Terminação de TLS**: Ambos os controladores podem tentar terminar o TLS em 443, levando a conflitos, a menos que o Istio utilize uma porta diferente ou encaminhamento avançado (por exemplo, encaminhamento baseado em SNI).
- **Sobrecarga operacional**: Gerir um IP partilhado aumenta a complexidade na depuração, monitorização e dimensionamento.
- **Limitações de DNS**: Usar um único IP com portas diferentes complica a configuração do DNS, pois a maioria dos clientes espera portas padrão (80, 443).