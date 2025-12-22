---
title: Kagent
tags:
  - Kubernets
  - kagent
enableToc: true
---
# Overview
O Kagent é um agent IA de código aberto que permite executar agentes de IA autônomos diretamente nos clusters do Kubernetes para automatizar operações, diagnosticar problemas e resolver problemas nativos da nuvem em várias etapas.

O Kagent foi criado especificamente para DevOps e engenheiros de plataforma, permitindo que os agentes de IA sejam executados no cluster e lidem com tarefas como resolução de problemas, manutenção de clusters, alertas do Prometheus, diagnósticos de tráfego do Istio e gestão do Argo Rollout - tudo definido de forma declarativa através dos Recursos Personalizados do Kubernetes (CRDs) 

**Componentes principais**:

- **Agentes**: Cada agente é definido por um prompt do sistema, configuração do modelo e um conjunto de ferramentas permitidas.
- **Tools**: Funções pré-construídas ou personalizadas do tipo MCP (por exemplo, GetResources, DescribeResource, ferramentas de consulta PromQL, instaladores Helm) definidas por meio de CRDs.
- **Framework runtime**: Inclui uma CLI, uma UI da Web e um mecanismo (criado com o [AutoGen](https://www.microsoft.com/en-us/research/project/autogen/)) que gerencia cadeias de planejamento, execução e raciocínio

Ao contrário dos chatbots tradicionais, o kagent utiliza capacidades avançadas de raciocínio e planeamento iterativo para lidar autonomamente com problemas de várias etapas em ambientes nativos da cloud. Ele transforma insights de IA em ações concretas, ajudando as equipes a enfrentar desafios operacionais comuns, como:

- Diagnosticar problemas de conetividade em vários saltos de serviço
- Solucionar problemas de degradação do desempenho de aplicativos
- Automatizar a geração de alertas a partir de métricas do Prometheus
- Depurar configurações de gateway e HTTPRoute
- Gerenciar implementações progressivas com o Argo Rollouts

**Benefícios do mundo real**

- **Rápido Troubleshoot**: Os agentes rastreiam de forma inteligente problemas de conetividade multi-hop, configurações incorrectas do sistema ou degradações de desempenho no Prometheus, Istio, Argo, Helm e muito mais
- **Automatize tarefas de observabilidade**: Use agentes baseados em PromQL para gerar alertas, painéis e solucionar problemas silenciosamente em segundo plano.
- **Gerenciar implantações e pipelines**: Agentes como o agente Argo Rollouts podem automatizar estratégias de conversão e implantação.
- **Escalar conhecimento especializado**: As equipes podem codificar runbooks padrão, lógica de depuração e playbooks operacionais em agentes que podem ser reutilizados por engenheiros ou até mesmo por não especialistas.

# Arquitetura do kagente

O Kagent é composto por vários componentes executados dentro e fora do cluster Kubernetes.


![[Kagent-1.png]]

## Controller

O controlador kagent é um controlador Kubernetes, escrito em Go, que sabe como lidar com CRDs personalizados para criar e gerenciar agentes de IA no cluster.

## App/Engine

O Engine kagent é o componente central do kagent. É uma aplicação Python que é responsável por executar o ciclo de conversação do agente. É construído sobre a estrutura AutoGen. Atualmente, estamos a executar o backend do autogenstudio, mas provavelmente num futuro próximo passaremos a executar o nosso próprio backend à medida que os casos de utilização começarem a divergir.

A equipe do autogen fez um trabalho maravilhoso ao criar uma estrutura flexível, poderosa e, acima de tudo, extensível para construir agentes de IA. Tiramos o máximo proveito disso, utilizando a estrutura e adicionando as nossas próprias equipes, agentes e ferramentas.
# O que é que o kagent install faz?

O comando `kagent install` implementa o control plane do Kagent no seu cluster Kubernetes. Isso inclui:
- Criar o namespace do kagent no seu cluster.
- Instalar os serviços principais do Kagent (servidor de API, dashboard e agentes de tempo de execução).
- Configurar os CRDs (Definições de Recursos Personalizados) que permitem definir agentes e ferramentas de IA como recursos do Kubernetes.
- Configurar segredos, incluindo sua chave de API OpenAI (ou outras chaves de API LLM), que o Kagent usa para alimentar seu raciocínio.
- Essencialmente, o kagent install inicializa toda a plataforma Kagent dentro do Kubernetes, para que sua CLI possa interagir com ela por meio da API e do painel.

O Kagent requer um LLM (como o GPT-4 do OpenAI ou outro modelo suportado) para que seus agentes funcionem.

Durante a instalação, ele verifica se há uma chave (Pode ser variável de ambiente, por exemplo, OPENAI_API_KEY) para armazenar como um segredo do Kubernetes.

# Deploy

O Kagent, como o kubectl, usa o contexto atual do kubeconfig para determinar com qual cluster trabalhar. Então, a maneira mais fácil de dizer ao Kagent qual cluster usar é mudar o contexto antes de executar o kagent install ou qualquer outro comando.

## Como utilizar um cluster específico

**Opção 1**: Usar `kubectl config use-context`

```shell
kubectl config use-context <context-name>
```

Então:

```shell
kagent install
```

O Kagent utilizará o contexto ativo.

**Opção 2**: Utilizar as flags `--kubeconfig` ou `--context`

Se não quiser alterar o seu contexto predefinido, pode especificar um contexto ou um ficheiro kubeconfig diretamente:

```shell
kagent install --context <context-name>
```

ou

```shell
kagent install --kubeconfig ~/.kube/config --context <context-name>
```

---

Instale o kagent no cluster usando a CLI. Primeiro, execute a CLI:

Antes, defina a chave da API OpenAI como uma variável de ambiente:

```shell
export OPENAI_API_KEY="your-api-key-here"
```

Instale o Kagent no seu cluster:

```shell
kagent install
```



# Usando outros modelos de AI

## Deploy com Azure OpenAI Key

Eis como configurar o Kagent para utilizar a sua chave Azure OpenAI em vez do fornecedor OpenAI predefinido:

### Criar segredo do Kubernetes com a sua chave do Azure

Primeiro, armazene a sua chave Azure OpenAI de forma segura em Kubernetes:

```shell
export AZURE_OPENAI_API_KEY="your-azure-openai-key"
kubectl create secret generic kagent-azureopenai \
  -n kagent \
  --from-literal=AZURE_OPENAI_API_KEY=$AZURE_OPENAI_API_KEY
```

Este segredo (kagent-azureopenai) será referenciado pela configuração do seu modelo

### Definir um recurso ModelConfig para o Azure

Em seguida, crie um CRD ModelConfig para que o Kagent saiba que deve usar o Azure em vez do OpenAI. Ajuste os placeholders de acordo:

```yml
apiVersion: kagent.dev/v1alpha1
kind: ModelConfig
metadata:
  name: azureopenai-model-config
  namespace: kagent
spec:
  apiKeySecretRef: kagent-azureopenai
  apiKeySecretKey: AZURE_OPENAI_API_KEY
  model: gpt-35-turbo              # Your deployed Azure model
  provider: AzureOpenAI
  azureOpenAI:
    azureEndpoint: "https://<your-resource-name>.openai.azure.com/"
    apiVersion: "2024-12-01-preview"  # Or your resource version
    azureDeployment: "gpt-35-turbo"    # Deployment name in Azure
    azureAdToken: ""                  # Optional, for MSI/AD auth

```

Aplique-o com:

```shell
kubectl apply -f azure-modelconfig.yaml
```

O Kagent reconhecerá agora o seu modelo baseado no Azure.


