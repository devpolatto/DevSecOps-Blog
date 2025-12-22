---
title: Container Health
tags:
  - Kubernets
enableToc: true
---
O Kubernetes Liveness Probe faz parte de um conjunto de verificações de integridade que o Kubernetes usa para determinar se um contêiner dentro de um pod está saudável e responsivo. No entanto, é na verdade a Readiness Probe que determina especificamente se um container está “pronto” para se juntar ao balanceador de carga e começar a receber tráfego.

Aqui está um detalhamento de como essas duas sondas funcionam e como elas afetam a prontidão do contêiner e o balanceamento de carga:

# 1. Liveness Probe

 O Liveness Probe verifica se o aplicativo dentro de um contêiner está ativo (ou seja, funcionando). Se essa sonda falhar, o Kubernetes eliminará o contêiner e o reiniciará de acordo com a política de reinicialização. Esta verificação é crucial para aplicações que podem ocasionalmente falhar ou ficar presas, e reiniciá-las pode restaurá-las a um estado saudável.
 
Um Liveness Probe é frequentemente configurada com:

- Solicitações HTTP (por exemplo, verificando um ponto de extremidade HTTP como `/health` para um status 200 OK).
- Verificações de soquete TCP (por exemplo, conexão a uma porta especificada).
- Execução de comando (por exemplo, executar um comando dentro do contêiner e garantir que ele retorne um status de sucesso).

# 2. Readiness Probe

O Readiness Probe verifica especificamente se o contêiner está pronto para lidar com o tráfego de entrada e se juntar ao pool do balanceador de carga. Isso é particularmente importante para aplicativos que podem demorar um pouco para serem inicializados (por exemplo, configurar conexões, carregar grandes conjuntos de dados) e não devem receber tráfego até que estejam totalmente operacionais.

Se um Readiness Probe falhar, o Kubernetes marcará o contêiner como não pronto, o que o impedirá de receber tráfego do serviço. Assim que a sonda for aprovada, o contêiner será marcado como pronto e o Kubernetes começará a rotear o tráfego para ele.

Configurações do **Sonda de Prontidão**
Semelhante à Sonda de Prontidão, um Readiness Probe pode ser configurada usando:

- Sondas de solicitação HTTP: O Kubernetes verifica um ponto de extremidade no aplicativo para um código de resposta específico (por exemplo, 200 OK).
- Sondas de soquete TCP: O Kubernetes tenta estabelecer uma conexão TCP com uma porta especificada.
- Executar sondas: O Kubernetes executa um comando especificado dentro do contêiner e verifica se ele retorna um status de saída zero.

---

Você pode (e muitas vezes deve) usar o Readiness Probe e o Liveness Probe na mesma implantação. Eles servem a propósitos complementares, e a configuração de ambos permite que o Kubernetes tenha uma visão mais detalhada da integridade e da prontidão operacional do seu contêiner.

No exemplo, em que você tem uma API Node.js com um ponto de extremidade `/health` que verifica a conetividade do banco de dados, veja como você pode usar as duas sondas:

- **Liveness Probe**: Isso poderia usar o ponto de extremidade /health para garantir que o contêiner ainda esteja responsivo e possa se comunicar com o banco de dados. Se a sonda falhar (por exemplo, a verificação do banco de dados falha repetidamente ou o endpoint não responde), o Kubernetes assumirá que o contêiner está em um estado quebrado e o reiniciará. Isso é útil para casos em que sua API pode ficar “presa” e exigir um novo início.

- **Readiness Probe**: Isso também pode usar o endpoint /health, mas com parâmetros diferentes (como um tempo limite mais curto ou uma frequência mais alta). A sonda de prontidão concentra-se em saber se o container está atualmente pronto para servir o tráfego. Se falhar (por exemplo, a conetividade do banco de dados está temporariamente inativa), o Kubernetes marcará o pod como “não pronto” e interromperá o roteamento de tráfego para ele até que a verificação seja aprovada novamente.

Exemplo de configuração YAML para ambas as sondas
Veja como você pode configurar as duas sondas no arquivo YAML de um deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
        - name: my-api-container
          image: my-api-image
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 1

```

Explicação dos parâmetros de configuração

- Liveness Probe:
	- initialDelaySeconds: Aguarda 10 segundos após o início do container antes de efetuar a primeira verificação.
	- periodSeconds: Verifica a cada 15 segundos.
	- timeoutSeconds: A sonda falha se o container não responder no espaço de 5 segundos.
	- failureThreshold: Se o ponto final /health falhar 3 vezes consecutivas, o Kubernetes reinicia o container.
- Readiness Probe:
	- initialDelaySeconds: Começa a verificar 5 segundos após o início do container.
	- periodSeconds: Verifica a cada 10 segundos para ver se o container está pronto para o tráfego.
	- timeoutSeconds: A sonda falha se o ponto de extremidade /health não responder dentro de 3 segundos.
	- failureThreshold: Apenas uma falha é necessária para que o Kubernetes marque o contêiner como “não pronto”.