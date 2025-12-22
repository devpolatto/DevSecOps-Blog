---
title: Analise de vulnerabildade com Trivy
tags:
  - Trivy
  - Kubernets
  - Docker
  - AzureDevOps
enableToc: true
---

O scanner de imagens é um componente crucial do DevSecOps (Desenvolvimento, Segurança e Operações) porque ajuda a identificar e mitigar vulnerabilidades de segurança em aplicativos em contêiner e suas dependências. O DevSecOps visa integrar práticas de segurança em todo o ciclo de vida de desenvolvimento de software, incluindo os estágios iniciais de desenvolvimento, para garantir que a segurança não seja uma reflexão tardia. O scanner de imagens aborda especificamente preocupações de segurança relacionadas a ambientes em contêineres, como contêineres do Docker e orquestração do Kubernetes.

Aqui estão algumas das principais razões que destacam a importância da analise de imagens no DevSecOps:

> Detecção Precoce de Vulnerabilidades

O scanner de imagens permite a detecção precoce de vulnerabilidades em imagens de contêiner. Ao digitalizar imagens como parte do processo de desenvolvimento, os problemas de segurança podem ser identificados e abordados antes de entrarem em produção, reduzindo o risco de violações de segurança.

> shift left

O scanner de imagens promove a abordagem “shift left” à segurança, o que significa que a segurança está integrada no processo de desenvolvimento desde o início. Os desenvolvedores podem identificar e corrigir vulnerabilidades em seus códigos e dependências no início do ciclo de vida do desenvolvimento, minimizando a necessidade de correções de segurança de última hora antes da implantação.

> Segurança do container

Os contêineres se tornaram uma tecnologia popular para empacotar e implantar aplicativos. No entanto, eles introduzem novos desafios de segurança. O scanner de imagens ajuda a garantir que as imagens do contêiner estejam livres de vulnerabilidades conhecidas, configurações incorretas ou outros problemas de segurança que possam ser explorados por invasores.

> Monitoramento Contínuo

O DevSecOps enfatiza o monitoramento contínuo de aplicativos e infraestrutura. O scanner de imagens não é uma atividade única, mas sim um processo contínuo que é integrado ao pipeline CI/CD (Continuous Integration/Continuous Deployment). Esse monitoramento contínuo ajuda a detectar e resolver novas vulnerabilidades que podem surgir ao longo do tempo.

---

# Trivy

O Trivy é um scanner de vulnerabilidades de código aberto projetado para detetar problemas de segurança em imagens de contêineres, sistemas de arquivos e configurações de infraestrutura como código, como manifestos do Kubernetes ou arquivos do Terraform. É particularmente conhecido pela sua simplicidade, velocidade e capacidades abrangentes de deteção de vulnerabilidades, tornando-o uma escolha popular para equipas de DevOps e pipelines de CI/CD.
## Overview
### Principais benefícios do Trivy para imagens Docker e arquivos Dockerfile

**Deteção abrangente de vulnerabilidades:**
- Examina as imagens do Docker em busca de vulnerabilidades conhecidas nos pacotes de software e SO instalados.
- Inspeciona Dockerfiles em busca de configurações incorretas que possam levar a riscos de segurança.
- Suporta a verificação de modelos de Infraestrutura como Código (IaC), aumentando a segurança das configurações de orquestração de contêineres.

**Facilidade de uso:**
- Requer configuração mínima - o Trivy pode analisar com um único comando.
- Actualiza automaticamente a sua base de dados de vulnerabilidades a partir de fontes públicas, como a NVD (National Vulnerability Database) e avisos de fornecedores.

**Velocidade e eficiência:**
- Varreduras rápidas com baixo uso de recursos, adequadas para uso em pipelines de CI/CD sem atrasos significativos.

**Ampla compatibilidade:**
- Suporta vários sistemas operacionais e gerenciadores de pacotes populares, garantindo ampla cobertura.
- Compatível com as principais ferramentas de CI/CD, como Jenkins, GitLab CI/CD e GitHub Actions.

**Relatórios detalhados:**
- Gera relatórios claros e acionáveis, destacando a gravidade e as correções das vulnerabilidades.
- Integra-se facilmente com ferramentas de monitorização e dashboards.

### Boas práticas de utilização do Trivy

**Automatizar a verificação em pipelines de CI/CD:**
- Integre o Trivy para executar varreduras em vários estágios do pipeline, como após a criação da imagem do Docker ou durante a implantação.

**Use a segurança [Shift-Left](https://itsmnapratica.com.br/shift-left/):**
- Habilite a verificação de modelos de Dockerfile e IaC no início do desenvolvimento para detetar configurações incorretas antes que elas cheguem à produção.

**Definir limites de gravidade:**
- Configure pipelines para falhar em verificações para vulnerabilidades acima de uma determinada gravidade (por exemplo, ALTA ou CRÍTICA).
- Use políticas para personalizar o que é considerado aceitável para sua organização.

**Mantenha o Trivy e os bancos de dados atualizados:**
- Atualize regularmente o Trivy para se beneficiar dos recursos e definições de vulnerabilidade mais recentes. Automatize as atualizações do banco de dados sempre que possível.

**Habilite a filtragem:**
- Use filtros para excluir vulnerabilidades conhecidas e aceitáveis ou falsos positivos para reduzir o ruído nos resultados.

**Combine com outras ferramentas:**
- Embora o Trivy seja abrangente, complemente seus resultados com outras ferramentas (por exemplo, testes dinâmicos de segurança de aplicativos ou ferramentas de proteção em tempo de execução) para uma abordagem de defesa em profundidade.

### Instalando o Trivy

O Trivy está disponível nos canais de distribuição mais comuns. A lista completa de opções de instalação está disponível na página Instalação. Aqui estão alguns exemplos populares:

- `brew install trivy`
- `docker run aquasec/trivy`
- Binário [https://github.com/aquasecurity/trivy/releases/latest/](https://github.com/aquasecurity/trivy/releases/latest/)
- [Outros](https://trivy.dev/v0.57/getting-started/installation/)


### Como executar o trivy

Estrutura do comando:

```bash
trivy <target> [--scanners <scanner1,scanner2>] <subject>
```

Exemplos:

```bash
# Scan Dockerfile for vulnerabilities
trivy config --severity HIGH,CRITICAL --exit-code 1 

# Scan Filesystem for vulnerabilities
trivy fs --scanners vuln,secret,misconfig myproject/

# Scan K8s for vulnerabilities
trivy k8s --report summary cluster
```
---



## Trivy na Pipeline

### Overview

A integração do Trivy nos pipelines do Azure DevOps ajuda a automatizar a verificação de vulnerabilidades durante o processo de CI/CD, garantindo imagens de containers, configurações e dependências seguras. Ao detectar problemas no início do ciclo de vida do desenvolvimento, está a melhorar a postura de segurança das suas aplicações.

**Por que usar o Trivy no Azure DevOps Pipelines?**

- Segurança Shift-Left: Detecte vulnerabilidades e configurações incorretas no início do pipeline de CI/CD.
- Varredura abrangente:
	- Examinar imagens de containers para detectar vulnerabilidades (por exemplo, em imagens Docker).
	- Auditar arquivos de infraestrutura como código (IaC) em busca de configurações incorretas (por exemplo, Terraform, manifestos do Kubernetes).
	- Examinar as dependências do código-fonte em busca de problemas de segurança.
- Integração perfeita: Funciona com pipelines de DevOps do Azure sem configurações complexas.
- Conformidade: Garante a adesão a padrões de segurança como CIS Benchmarks.


### implementar o Trivy no pipeline do Azure DevOps

Para garantir que o Trivy esteja disponível em todo o pipeline, ele deve ser instalado em um diretório persistente e adicionado ao PATH usando os comandos de log do Azure DevOps.

```yml
- task: CmdLine@2
  displayName: "Installing Trivy"
  inputs:
    script: |
      # Define the directory for installation
      export TRIVY_DIR=$(Agent.ToolsDirectory)/trivy # /home/AzDevOps/azure-devops-agent/_work/_tool/trivy
      mkdir -p $TRIVY_DIR
      
      # Install Trivy in the designated directory
      curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b $TRIVY_DIR

      # Set ownership for the Azure DevOps agent user
      sudo chown $(whoami):$(whoami) $TRIVY_DIR -R

      # Add Trivy to the PATH using Azure DevOps logging command
      echo "##vso[task.prependpath]$TRIVY_DIR"
      
      # Verify Trivy installation
      trivy --version

```

Esta task:
- Instala o Trivy no diretório `$(Agent.ToolsDirectory)/trivy`.
- Atualiza o PATH globalmente para o pipeline usando o comando `##vso[task.prependpath]`.
- Garante que a instalação seja acessível pelo agente do pipeline.


```shell title:"Bash output"
Trivy is not installed. Installing now...

/home/AzDevOps/azure-devops-agent/_work/_tool/trivy

aquasecurity/trivy info checking GitHub for latest tag

aquasecurity/trivy info found version: 0.57.1 for v0.57.1/Linux/64bit

aquasecurity/trivy info installed /home/AzDevOps/azure-devops-agent/_work/_tool/trivy/trivy
```

#### Verificar se há erros de configuração no Dockerfile

O Trivy pode verificar os Dockerfiles em busca de configurações incorretas com base nas práticas recomendadas de segurança.

```yml
- task: CmdLine@2
  displayName: "Running Trivy Dockerfile scan..."
  inputs:
    script: |
      # Scan Dockerfile for vulnerabilities
      trivy config --severity HIGH,CRITICAL --exit-code 1 $(WORKDIR)/Dockerfile

```

Esta task:
- Examina o Dockerfile localizado em $(WORKDIR)/Dockerfile.
- Falha o pipeline se forem detectadas vulnerabilidades de gravidade HIGH ou CRITICAL.


#### Verificar se há vulnerabilidades na imagem do Docker

Assim que a imagem Docker for criada, analise-a em busca de vulnerabilidades.

```yml
- task: CmdLine@2
  displayName: "Running Trivy image scan..."
  inputs:
    script: |
      # Scan Docker image for vulnerabilities
      trivy image --severity HIGH,CRITICAL --exit-code 1 $(REMOTE_DOCKER_IMAGE):$(REMOTE_DOCKER_IMAGE_TAG)

```

Esta task:
- Examina a imagem do Docker marcada com `$(REMOTE_DOCKER_IMAGE):$(REMOTE_DOCKER_IMAGE_TAG)`
- Falha o pipeline se forem detectadas vulnerabilidades de gravidade HIGH ou CRITICAL.

#### Implementando aprovação adicional

A pipeline abaixo efectua verificações de segurança utilizando o Trivy numa fase de desenvolvimento e incorpora um processo de aprovação manual se forem encontradas vulnerabilidades específicas.


```yml
stages:
  - stage: Development
    jobs:
      - job: trivy
        displayName: "Releasing on Development"
        timeoutInMinutes: 0
        steps:
          - checkout: self

          - template: /.azuredevops/templates/git/checkout.yml
            parameters:
              HASH_CODE: $(HASH_CODE)
              WORKDIR_RELEASE: $(WORKDIR_RELEASE)
              PRODUCTION: $(PRODUCTION)

          - task: CmdLine@2
            displayName: "Check and Install Trivy"
            inputs:
              script: |
                if ! command -v trivy &> /dev/null; then
                  echo "Trivy is not installed. Installing now..."
                  export TRIVY_DIR=$(Agent.ToolsDirectory)/trivy
                  echo $TRIVY_DIR
                  mkdir -p $TRIVY_DIR

                  curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b $TRIVY_DIR

                  sudo chown $(whoami):$(whoami) $TRIVY_DIR -R

                  echo "##vso[task.prependpath]$TRIVY_DIR"
                else
                  echo "Trivy is already installed. Skipping installation."
                  trivy --version
                fi

          - task: CmdLine@2
            displayName: "Running Trivy Dockerfile scan..."
            name: trivyScan
            inputs:
              script: |
                echo "Trivy is already installed. Skipping installation."
                
                trivy config --severity CRITICAL --exit-code 1 $(WORKDIR)/Dockerfile
		if [ $? -eq 1 ]; then
                   echo "CRITICAL vulnerabilities found in Dockerfile! Failing the pipeline."
                   exit 1
                fi
                
                trivy config --severity HIGH,CRITICAL --exit-code 2 $(WORKDIR)/Dockerfile
                if [ $? -eq 2 ]; then
                  echo "HIGH vulnerabilities found! Marking for manual approval."
                  echo "##vso[task.setvariable variable=trivyIssuesFound;isOutput=true]true"
                fi

      - deployment: PauseForApproval
        displayName: "Manual Approval for HIGH Vulnerabilities"
        environment: "approvalrequired"
        dependsOn: trivy
        condition: eq(dependencies.trivy.outputs['trivyScan.trivyIssuesFound'],'true')
        pool: server
        strategy:
          runOnce:
            deploy:
              steps:
                - task: ManualValidation@0
                  displayName: "Manual Approval for HIGH Vulnerabilities"
                  inputs:
                    notifyUsers: 'name.lastname'
                    instructions: 'Vulnerabilities detected. Review and approve to continue.'

```

O Trivy examina Dockerfile em busca de vulnerabilidades com gravidade HIGH ou CRITICAL, e define uma variável de pipeline (trivyIssuesFound) para acionar um processo de revisão manual se forem encontradas vulnerabilidades HIGH.

Em seguida, a pipeline exige uma validação manual com base no valor da variavel
`trivyIssuesFound`, o que Instrui os utilizadores a avaliarem as vulnerabilidades antes de avançarem mais no pipeline.

```yaml
- deployment: PauseForApproval
        displayName: "Manual Approval for HIGH Vulnerabilities"
        environment: "approvalrequired"
        dependsOn: trivy
        condition: eq(dependencies.trivy.outputs['trivyScan.trivyIssuesFound'],'true')
        pool: server
        strategy:
          runOnce:
            deploy:
              steps:
                - task: ManualValidation@0
                  displayName: "Manual Approval for HIGH Vulnerabilities"
                  inputs:
                    notifyUsers: 'name.lastname'
                    instructions: 'Vulnerabilities detected. Review and approve to continue.'
```

### Recomendações para a integração do pipeline CI/CD

**Configurar como uma etapa do pipeline:**
- Incluir varreduras Trivy como uma etapa obrigatória na configuração do seu pipeline.

**Varreduras de imagem do contêiner:**
- Verificar a imagem final do Docker antes de enviar para o registo para identificar vulnerabilidades no software incluído.

**Varreduras pré-compilação:**
- Examinar Dockerfiles ou outras configurações de IaC para detetar possíveis problemas de segurança durante a fase de desenvolvimento.

**Gerar relatórios para conformidade:**
- Exporte relatórios Trivy em formatos como JSON ou SARIF para integração com ferramentas de conformidade ou auditorias.

**Otimizar o desempenho da varredura:**
- Armazene em cache as atualizações do banco de dados do Trivy para evitar baixá-las repetidamente, especialmente em ambientes com vários pipelines.


### Exportando relatorio para o Elastic

Use o Logstash para consumir os relatórios. Segue exemplo de uma pipelina do logstash:

```json title:logstash.conf
logstash.conf: |

input {
	file {
		path => "/data/trivy.json"
		start_position => "beginning"
		codec => "json"
	}
}

filter{
	if [Results] {
		mutate {
			add_field => { "[@metadata][filtered_results]" => "%{[Results]}" }
		}
		split { field => "[@metadata][filtered_results]" }
		mutate {
			replace => { "[Target]" => "[@metadata][filtered_results][Target]" }
			replace => { "[Class]" => "[@metadata][filtered_results][Class]" }
			replace => { "[Type]" => "[@metadata][filtered_results][Type]" }
			replace => { "[Vulnerabilities]" => "[@metadata][filtered_results][Vulnerabilities]" }
		}
	} else {
		drop { }
	}
}

output {
	elasticsearch {
		hosts => ''
		user => ''
		password => ''
		index => "trivy-vulnerabilities-%{+YYYY.MM.dd}"
	}
	stdout { codec => rubydebug }
}
```

## Trivy Kubernets Operator

### Overview

O Trivy Operator utiliza o Trivy para analisar continuamente o cluster do Kubernetes em busca de problemas de segurança. As análises são resumidas em relatórios de segurança como Definições de Recursos Personalizados do Kubernetes, que se tornam acessíveis através da API do Kubernetes. O Operator faz isso observando o Kubernetes em busca de alterações de estado e acionando automaticamente as verificações de segurança em resposta. 

Quando utilizado com o Trivy Kubernetes Operator, permite uma integração perfeita com os fluxos de trabalho do Kubernetes, tornando eficiente a avaliação e a comunicação de vulnerabilidades em todo o cluster. Por exemplo, uma verificação de vulnerabilidade é iniciada quando um novo Pod é criado. Desta forma, os utilizadores podem encontrar e visualizar os riscos relacionados com diferentes recursos de uma forma nativa do Kubernetes.

![image.png](/.attachments/image-af215749-f826-4448-9d63-08e3b32d49b8.png)

**Por que usar Trivy como uma ferramenta de análise de vulnerabilidade com Kubernetes?**
- Riscos específicos do Kubernetes:
	- Os clusters do Kubernetes são dinâmicos e geralmente envolvem várias imagens, configurações e dependências de contêineres. Configurações incorretas ou imagens desatualizadas podem expô-los a vulnerabilidades.
- A Trivy é especializada na verificação de:
	- Vulnerabilidades em imagens de contentores.
	- Configurações incorretas em manifestos do Kubernetes.
	- Conformidade com padrões de segurança, como CIS Benchmarks.
- Cobertura abrangente:
	- O Trivy fornece uma única ferramenta para verificar:
	- Imagens de contêineres.
	- Sistemas de arquivos e repositórios Git.
	- Manifestos do Kubernetes (gráficos YAML/Helm).
	- Ferramentas de infraestrutura como código (IaC), como Terraform.
- Ele identifica vulnerabilidades e exposições comuns (CVEs) e oferece recomendações acionáveis.


**Benefícios do uso do Trivy Kubernetes Operator sobre o Trivy CLI Standalone**

Embora o Trivy CLI Standalone seja uma ferramenta poderosa para varredura ad-hoc, o Trivy Kubernetes Operator oferece vantagens adicionais para ambientes Kubernetes:

- **Monitoramento em todo o cluster**:
	- O operador automatiza a varredura de recursos dentro do cluster Kubernetes, incluindo nós, pods e imagens, sem exigir acionadores manuais.
- **Varredura contínua**:
	- Verifica proativamente as workloads e suas dependências à medida que são criadas ou modificadas.
	- Detecta vulnerabilidades e configurações incorretas em tempo real, garantindo uma correção mais rápida.
- **Relatórios centralizados**:
	- As varreduras geram insights detalhados sobre:
		- workloads vulneráveis.
		- Imagens afetadas e seus namespaces.
		- Objetos do Kubernetes mal configurados.
	- Os resultados são armazenados em CRDs (Custom Resource Definitions) personalizados do Kubernetes, permitindo que os desenvolvedores e as equipes de segurança consultem os dados por meio de ferramentas do Kubernetes.
- **Nativo do Kubernetes**:
	- O operador é executado como um recurso do Kubernetes, o que significa que ele aproveita o ecossistema nativo do Kubernetes para agendamento, implantação e dimensionamento.
	- Ajusta automaticamente a frequência de varredura com base na disponibilidade de recursos do cluster.
- **Intervenção manual reduzida**:
	- Ao contrário da CLI, o operador não requer execução periódica ou scripting para automação. Funciona de forma autónoma dentro do cluster.


### Implementando o Trivy Operator

**1. Pré-requisitos**

Antes de implantar o Trivy Operator, verifique o seguinte:
- Um cluster Kubernetes em execução.
- Helm CLI instalado e configurado.
- Permissões para instalar recursos personalizados (função de administrador de cluster ou equivalente).


**2. Adicionar o Repositório do Helm do Operador Trivy**

Adicione o repositório Helm para o Trivy Operator:

```Shell ln:true
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm repo update
```

**3. Instalar o Trivy Operator**

Use o comando helm install para implantar o Trivy Operator:

```Shell title:bash
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --version 0.25.0
```

**4. Verificar a instalação**

Verificar se os pods do Trivy Operator estão a funcionar:

```Shell title:bash
kubectl get pods -n trivy-system
---
# output
NAME                                READY   STATUS    RESTARTS   AGE
trivy-operator-<hash>               1/1     Running   0          <age>
```

**5. Personalizando a instalação do Trivy Operator**

Você pode personalizar a configuração do Trivy Operator fornecendo um arquivo values.yaml ou substituindo os valores padrão com o sinalizador --set durante a instalação.

Exempo:

```yaml title:values.yaml
trivyOperator: 
  config: 
    severity: CRITICAL,HIGH 
    ignoreUnfixed: true 
    prometheus: 
      enabled: true # Permite exportar métricas no formato Prometheus.
service:
  metricsPort: 8080
trivy:
  ignoreUnfixed: true
serviceMonitor:
  # enabled determina se um serviceMonitor deve ser implantado
  enabled: false
```

Instale o Trivy Operator com a configuração personalizada:


```Shell title:Bash
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --version 0.25.0 \
  -f values.yaml
```

Output

```shell title:"Bash output"
NAME: trivy-operator
LAST DEPLOYED: Sun Dec  1 22:58:02 2024
NAMESPACE: trivy-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have installed Trivy Operator in the trivy-system namespace.
It is configured to discover Kubernetes workloads and resources in
all namespace(s).

Inspect created VulnerabilityReports by:

    kubectl get vulnerabilityreports --all-namespaces -o wide

Inspect created ConfigAuditReports by:

    kubectl get configauditreports --all-namespaces -o wide

Inspect the work log of trivy-operator by:

    kubectl logs -n trivy-system deployment/trivy-operator
```

Como podemos ver, o trivy agora se tornou comandos nativos do Kubernets. Como teste, vamos executar uma varredura numa workload

```shell title:bash
kubectl get vulnerabilityreports -n app-vulnerabilits -o wide
```

output:

```shell unwrap title:"Bash output" 
NAME                                                        REPOSITORY                       TAG     SCANNER   AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN
replicaset-react-application-7f797b95f4-react-application   anaisurlichs/react-example-app   8.0.0   Trivy     20h   8          58     37       2     0
```

Se for necessário remover o Trivy Operator:

```shell title:bash
helm uninstall trivy-operator --namespace trivy-system
kubectl delete namespace trivy-system
```

Para mains informacoes sobre quais foram os CRD criado, [acesse a página do Trivy](https://aquasecurity.github.io/trivy-operator/latest/docs/crds/)

---

## Trivy com ferramentas de observabilidade, como o Prometheus

A integração do Trivy com ferramentas de observabilidade aumenta a visibilidade e a conscientização de vulnerabilidades em clusters do Kubernetes.

- Integração do Prometheus:
	- Exportar métricas: O Trivy Kubernetes Operator expõe métricas sobre vulnerabilidades e resultados de verificação em um formato compatível com o Prometheus.
	- Insights em tempo real: O Prometheus extrai métricas dos endpoints do Trivy, permitindo que as equipes acompanhem as tendências ao longo do tempo, como:
		- Número de vulnerabilidades críticas por namespace.
		- Contagens de vulnerabilidades por tipo de carga de trabalho (por exemplo, pods, deployments).
		- Tempos de conclusão de varredura e taxas de sucesso.
- Dashboards do Grafana:
		- Use as métricas do Prometheus para criar dashboards do Grafana para visualizar os riscos de segurança em todo o cluster.
	- Os painéis podem mostrar:
		- Mapas de calor de vulnerabilidades em cargas de trabalho.
		- Tendências na resolução ou mitigação de CVEs.
		- Pontuações de conformidade com base nas configurações do Kubernetes.
- Resposta a incidentes:
	- As regras de alerta do Prometheus podem ser configuradas para limites específicos, como:
		- Um alto número de vulnerabilidades críticas detectadas em namespaces de produção.
		- Configurações incorretas recorrentes sinalizadas pelo Trivy.

### Integração do Trivy Operator com o Prometheus Stack (kube-prometheus-stack) usando o Helm

Para integrar o Trivy Kubernetes Operator com a [stack do Prometheus](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml), precisamos configurar o Trivy para exportar métricas no formato do Prometheus e configurar um ServiceMonitor para permitir que o Prometheus extraia essas métricas.


**1. Instalando o kube-prometheus-stack**

Instale o Chart do prometheus stack

```shell title:bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Vamos primeiro definir alguns modificações no value.yaml

```yaml title:values.yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}

grafana:
  sidecar:
    datasources:
      defaultDatasourceEnabled: true
```

Instaler o prometheus stack

```shell title:bash
helm install prometheus prometheus-community/kube-prometheus-stack 
	-n monitoring \
	--create-namespace \
	--version 48.3.1 \
	-f values.yaml
```

Verifique se todos os recursos foram provisionados

```shell uwnrap title:bash
kl get pods -n monitoring          
```

output:

```shell title:bash
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          19h
prometheus-grafana-6d88995fd5-6jvjk                      3/3     Running   0          19h
prometheus-kube-prometheus-operator-69c5dfdc44-h2w8v     1/1     Running   0          19h
prometheus-kube-state-metrics-786fbd7c69-bpgb6           1/1     Running   0          19h
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          19h
prometheus-prometheus-node-exporter-wpqhs                1/1     Running   0          19h
```

**2. Habilitar métricas do Prometheus no Trivy Operator**

Seguindo o deplymento do Trivy Operator realizado nesse documento, já temos as metricas disponíveis para consumo. Porém, vamos fazer algumas alterações:

```yaml title:values.yaml
trivyOperator: 
  config: 
    severity: CRITICAL,HIGH 
    ignoreUnfixed: true 
    prometheus: 
      enabled: true # Permite exportar métricas no formato Prometheus.
service:
  metricsPort: 8080
  headless: true
trivy:
  ignoreUnfixed: true
serviceMonitor:
  # enabled determina se um serviceMonitor deve ser implantado
  enabled: true
  namespace: prometheus
```

Execute novamente o helm:

```shell title:bash
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --version 0.25.0 \
  -f values.yaml
```
Em andamento...