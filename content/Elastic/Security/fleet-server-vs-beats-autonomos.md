---
title: Fleet Server vs Beats Autônomos - Arquitetura e Trade-offs
tags:
  - Elasticsearch
  - Beats
  - Fleet Server
  - Arquitetura
  - DevSecOps
enableToc: true
---

# Fleet Server vs Beats Autônomos: Qual escolher?

> Uma análise profunda das duas abordagens de deploy do Elastic Beats, seus trade-offs arquiteturais e quando cada uma é a melhor escolha para sua infraestrutura.

## Introdução

Ao construir uma solução SIEM com Elastic, uma das primeiras decisões arquiteturais é: **como vou gerenciar e implantar os agents de coleta de dados?**

O Elastic oferece duas abordagens fundamentalmente diferentes:

1. **Fleet Server**: Uma solução centralizada de gerenciamento dentro do Elastic Stack
2. **Beats Autônomos**: Agentes instalados e configurados manualmente em cada host

Essa decisão impacta diretamente em:
- Complexidade operacional
- Escalabilidade da infraestrutura
- Flexibilidade das configurações
- Capacidade de resposta a mudanças
- Custo total de propriedade

Este artigo explora as duas abordagens, seus casos de uso, e oferece uma matriz de decisão para ajudá-lo a escolher a estratégia certa para seu ambiente.

---

## O que é Fleet Server?

**Fleet Server** é uma solução de gerenciamento centralizado de agentes Elastic. Ele funciona como um "control plane" dentro do Elastic Stack que:

- **Implanta agentes remotamente** em múltiplos hosts
- **Distribui configurações** de forma centralizada via Kibana UI
- **Atualiza agentes** em massa sem intervenção manual
- **Inscreve novos hosts** dinamicamente (com políticas de inscrição)
- **Fornece observabilidade** sobre o status de cada agente

### Como Fleet Server Funciona

```
┌─────────────────────────────────────────┐
│     Elastic Cloud / Elasticsearch       │
│  ┌─────────────────────────────────────┐│
│  │    Fleet Server (control plane)      ││
│  └──────────────┬──────────────────────┘│
│                 │                        │
│  ┌──────────────▼──────────────────────┐│
│  │      Kibana Management UI           ││
│  └──────────────────────────────────────┘│
└────────────────┬────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
    ┌───▼──┐         ┌───▼──┐
    │Agent1│         │Agent2│  ... AgentN
    └──────┘         └──────┘
```

Fleet Server gerencia políticas (*policies*) que definem qual agente roda onde. Uma política pode incluir Auditbeat, Filebeat, ou qualquer outro Beat, com configurações específicas.

### Características Principais

| Aspecto | Descrição |
|---------|-----------|
| **Gerenciamento** | Centralizado via Kibana |
| **Configuração** | UI ou YAML (policy-as-code) |
| **Atualização** | Automática ou agendada |
| **Inscrição** | Dinâmica (enrollment tokens) |
| **Observabilidade** | Status de cada agente em tempo real |
| **Escalabilidade** | Suporta milhares de agentes |

---

## O que são Beats Autônomos?

**Beats Autônomos** são agentes Elastic instalados e configurados manualmente em cada host, de forma independente. Não há um "control plane" — cada Beat é responsável por sua própria configuração.

### Como Beats Autônomos Funcionam

```
┌──────────────────┐
│  Elasticsearch   │
└────────┬─────────┘
         │
    ┌────┴─────────────────────┐
    │                          │
┌───▼─────┐            ┌───────▼───┐
│Host 1   │            │Host 2     │
│Auditbeat│ ────┐   ┌──│Auditbeat  │
│Filebeat │     │   │  │Filebeat   │
└─────────┘     │   │  └───────────┘
                └───┴─┐
                  (independentes)
```

Cada host gerencia sua própria configuração via arquivo YAML local (`auditbeat.yml`, `filebeat.yml`, etc.). O Elasticsearch recebe dados, mas não há comunicação de controle reversa.

### Características Principais

| Aspecto | Descrição |
|---------|-----------|
| **Gerenciamento** | Descentralizado (config files) |
| **Configuração** | YAML local em cada host |
| **Atualização** | Manual (via config mgmt ou SSH) |
| **Inscrição** | Não aplicável (credenciais hardcoded) |
| **Observabilidade** | Logs locais + Elasticsearch metrics |
| **Escalabilidade** | Depende da estratégia de deployment |

---

## Fleet Server: Prós e Contras

### ✅ Vantagens

1. **Gerenciamento Centralizado**
   - Uma única interface (Kibana) para gerenciar 100+ agentes
   - Sem necessidade de SSH em cada host
   - Políticas versioned e auditadas

2. **Escalabilidade Operacional**
   - Adicione novos hosts sem configuração manual
   - Tokens de inscrição (*enrollment tokens*) permitem self-service
   - Reduz overhead de DevOps

3. **Atualizações em Massa**
   - Distribua novas configurações de uma vez
   - Rollout gradual (canary) possível
   - Sem downtime de agente

4. **Observabilidade de Agentes**
   - Dashboard mostrando status de cada agente (online, offline, versão)
   - Alertas para agentes que caíram
   - Histórico de mudanças de configuração

5. **Compliance e Auditoria**
   - Trilha de auditoria de quem mudou qual configuração
   - Garantia de consistência de configuração
   - Relatórios de conformidade mais fáceis

### ❌ Desvantagens

1. **Complexidade Adicional**
   - Fleet Server é um componente extra para gerenciar
   - Requer rede entre hosts e Fleet Server
   - Aumenta overhead para ambientes pequenos (< 10 hosts)

2. **Menos Flexibilidade**
   - Menos customização possível via UI
   - Configurações muito personalizadas ficam difíceis
   - ILM, index patterns, e templates complexos precisam de workarounds

3. **Dinâmica de Cloud Limitada**
   - Como mencionado no artigo original [[Monitoramento_de_Segurança_para_Conformidade_com_PCI-DSS|original]]: "Ainda não foi possível configurar o Fleet para ingressar Hosts dinâmicos na cloud"
   - VMSS da Azure, Auto Scaling Groups da AWS requerem tooling customizado
   - Não há suporte nativo para ephemeral infrastructure (containers, Lambda)

4. **Segurança de Rede**
   - Requer comunicação bidirecional entre hosts e Fleet Server
   - Todos os hosts precisam alcançar o Fleet Server
   - Mais superfície de ataque

5. **Custo**
   - Fleet Server consome recursos adicionais (vCPU, memória)
   - Requer alta disponibilidade do próprio Fleet Server
   - Pode não vale a pena em ambientes pequenos

---

## Beats Autônomos: Prós e Contras

### ✅ Vantagens

1. **Controle Total**
   - Configuração 100% customizável via YAML
   - Fluxos de dados personalizados (índices customizados, templates, ILM)
   - Sem limitações impostas pela UI do Kibana

2. **Sem Dependência Central**
   - Não depende de Fleet Server estar online
   - Agentes continuam coletando mesmo se Fleet Server cair
   - Reduz pontos de falha

3. **Simplicidade para Ambientes Pequenos**
   - Sem overhead de componentes extras
   - Instalação direta: `apt install auditbeat`
   - Menor footprint de recurso

4. **Integração com Infrastructure-as-Code**
   - Fácil de gerenciar via Terraform, Ansible, CloudFormation
   - YAML local se integra naturalmente com GitOps
   - Histórico de mudanças no Git

5. **Suporte a Infraestrutura Dinâmica**
   - Funciona bem com containers (build a imagem, deploy)
   - Sem necessidade de inscrição dinâmica
   - Scripts de inicialização podem injetar credenciais

### ❌ Desvantagens

1. **Gerenciamento Manual em Escala**
   - Atualizar 100 hosts = 100 mudanças de config (ou Ansible, Puppet, etc.)
   - Sem interface centralizada
   - Risco de configurações divergentes

2. **Sem Observabilidade de Agentes**
   - Não há dashboard mostrando "qual agente está online?"
   - Precisa monitorar logs locais ou metrics do Elasticsearch
   - Alertas sobre agente caído requerem scripting customizado

3. **Atualizações Lentas**
   - Cada host requer atualização manual ou via config management
   - Sem rollout gradual nativo
   - Maior risco de inconsistência

4. **Esforço DevOps**
   - Requer pipeline de deployment (Ansible, Terraform, etc.)
   - Debugging requer SSH em múltiplos hosts
   - Maior curva de aprendizado para newcomers

5. **Sem Auditoria Centralizada**
   - Mudanças de config não são rastreadas centralmente
   - Compliance reports precisam ser construídos manualmente
   - Risco de configurações deixarem de estar atualizadas

---

## Matriz de Decisão: Fleet Server vs Beats Autônomos

| Critério | Fleet Server | Beats Autônomos |
|----------|--------------|-----------------|
| **Número de hosts** | 20+ | < 20 |
| **Infraestrutura dinâmica** (VMSS, ASG) | Não suportado | ✅ Bem suportado |
| **Customização de config** | Limitada | ✅ Sem limites |
| **Curva de aprendizado** | Média-Alta | Baixa |
| **Gerenciamento em escala** | ✅ Fácil | Difícil |
| **Custo operacional** | Médio-Alto | Baixo |
| **Observabilidade de agentes** | ✅ Excelente | Limitada |
| **Tempo para produção** | Mais longo | Mais rápido |
| **Suporte a compliance** | ✅ Melhor | Requer scripting |
| **Integração com IaC** | Possível, mas indireto | ✅ Natural |

---

## Casos de Uso Recomendados

### Use Fleet Server Se:

✅ Você tem **20+ hosts** para gerenciar  
✅ Seu time é **DevSecOps experiente** (confortável com complexidade)  
✅ Você precisa de **atualizações rápidas** em massa  
✅ **Compliance e auditoria** centralizadas são críticas  
✅ Você quer **observabilidade de agentes** em Kibana  
✅ Sua infraestrutura é **estável** (VMs long-lived, não containers)  

**Exemplo**: Ambiente corporativo com 50 VMs on-prem ou em EC2 dedicadas, gerenciado por um time de segurança centralizado.

---

### Use Beats Autônomos Se:

✅ Você tem **< 20 hosts** ou **poucos hosts críticos**  
✅ Precisa de **customização agressiva** (ILM, índices, fluxos de dados)  
✅ Sua infraestrutura é **dinâmica** (Kubernetes, containers, VMSS)  
✅ Você quer **controle total** via GitOps/IaC  
✅ **Menos complexidade operacional** é uma prioridade  
✅ Você tem **experiência com Ansible/Terraform** e prefere infrastructure-as-code  

**Exemplo**: Startup com infraestrutura em Kubernetes, onboarding contínuo de servidores via Terraform, configurações mantidas em Git.

---

## Abordagem Híbrida: O Melhor dos Dois Mundos?

Em alguns casos, você pode combinar as duas abordagens:

```yaml
Camada 1 - Estatária (Fleet Server):
  - 10 VMs críticas long-lived
  - Gerenciadas via Fleet Server
  - Monitoramento centralizado

Camada 2 - Dinâmica (Beats Autônomos):
  - Kubernetes cluster com 100+ pods
  - Cada pod tem Filebeat via sidecar/init container
  - Configurado via Helm values ou ConfigMap
```

**Benefícios**:
- Escalabilidade sem limites
- Observabilidade onde importa (tier crítico)
- Flexibilidade onde necessário (infraestrutura dinâmica)

**Desafios**:
- Dois modelos operacionais = mais complexidade
- Equipes precisam dominar ambos
- Monitoramento fragmentado

---

## Implementação Prática: Exemplo com Beats Autônomos

Se você escolher Beats Autônomos, aqui está o fluxo que utilizo para monitoramento de segurança com Auditbeat:

### 1. Configuração Base

```yaml
# auditbeat.yml
auditbeat.modules:
  - module: auditd
    audit_rule_files: ["$${path.config}/audit.rules.d/*.conf"]

  - module: file_integrity
    paths:
      - /bin
      - /usr/bin
      - /sbin
      - /usr/sbin
      - /etc
    max_file_size: 200MiB

  - module: system
    datasets:
      - package
    period: 2m

  - module: system
    datasets:
      - host
      - login
      - process
      - user
    state.period: 12h
    user.detect_password_changes: true
    process.hash.max_file_size: 200MiB
    process.hash.scan_rate_per_sec: 50MiB

    login.wtmp_file_pattern: /var/log/wtmp*
    login.btmp_file_pattern: /var/log/btmp*

setup.template.enabled: true
setup.template.name: "auditbeat-pci-%%{[agent.version]}"
setup.template.pattern: "auditbeat-pci-%%{[agent.version]}"
setup.template.fields: "$${path.config}/fields.yml"
setup.template.settings:
  index.number_of_shards: 1

tags: ["pci", "vmss-proxy"]

setup.ilm.enabled: true
setup.ilm.policy_name: "pci-logs-12month"

output.elasticsearch:
  hosts: ["${ELASTIC_HOSTS}"]
  api_key: "${API_KEY}"
  index: "auditbeat-pci-%%{[agent.version]}"

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/auditbeat/
  name: auditbeat.log
  keepfiles: 7
  permissions: 0644
```

### 2. Deployment com Terraform

```hcl
variable "auditbeat_version" {
  type    = string
  default = "8.11.1"
}

auditbeat_configuration = base64encode(templatefile("${path.module}/templates/security/auditbeat.yml.tpl", {
   ELASTIC_HOSTS = local.tfstate_elastic_cloud.elasticsearch.https_endpoint
   API_KEY       = data.azurerm_key_vault_secret.elasticsearch_pci_api_key.value
}))

data "template_cloudinit_config" "this" {
  gzip          = true
  base64_encode = true
  part {
    filename     = "init.cfg"
    content_type = "text/cloud-config"
    content = templatefile("${path.module}/templates/cloud-init-cfg.yml.tpl", {
      AUDITBEAT_CONFIGURATION         = local.auditbeat_configuration
      AUDITBEAT_VERSION               = var.auditbeat_version
    })
  }
}

resource "azurerm_linux_virtual_machine_scale_set" "proxy" {
   ...
  user_data           = data.template_cloudinit_config.this.rendered
  ...
}
```

Configuração via cloud-init `cloud-init-cfg.yml.tpl`:

```yaml
#cloud-config
package_update: true
package_upgrade: true

bootcmd:
  - [
      sh,
      -c,
      "until [ -b /dev/disk/azure/scsi1/lun${DISK_LUN} ]; do sleep 1; done",
    ]

packages:
  - curl
  - gpg
  - libpam-pwquality
  - auditd
  - systemd
  - jq

write_files:
  - path: /etc/auditbeat/auditbeat.yml
    owner: root:root
    permissions: "0644"
    encoding: b64
    content: ${AUDITBEAT_CONFIGURATION}

runcmd:
  - chmod 1777 /tmp
  - systemctl stop auditbeat
  - |
    wget -qO /tmp/auditbeat-${AUDITBEAT_VERSION}-amd64.deb \
      https://artifacts.elastic.co/downloads/beats/auditbeat/auditbeat-${AUDITBEAT_VERSION}-amd64.deb
  - dpkg -i --force-confold /tmp/auditbeat-${AUDITBEAT_VERSION}-amd64.deb
  - |
    auditbeat test config -c /etc/auditbeat/auditbeat.yml
    if [ $? -ne 0 ]; then
      echo "Auditbeat configuration is invalid. Aborting cloud-init script." >&2
      exit 1
    fi
  - systemctl enable auditd
  - systemctl start auditd
  - systemctl enable auditbeat
  - systemctl start auditbeat
  - rm /tmp/auditbeat-${AUDITBEAT_VERSION}-amd64.deb

output:
  all: "| tee -a /var/log/cloud-init-output.log"

final_message: "The system is finally up, after $UPTIME seconds"
```

## Conclusão

A escolha entre **Fleet Server** e **Beats Autônomos** não é sobre qual é "melhor" — é sobre qual se adapta melhor ao seu contexto:

- **Fleet Server** brilha em ambientes **escaláveis, estáveis e com muitos hosts**, onde o gerenciamento centralizado compensa a complexidade.

- **Beats Autônomos** são a escolha certa para **flexibilidade, infraestrutura dinâmica e times pequenos** que valorizam controle e simplicidade operacional.

A decisão também depende de sua **maturidade operacional**: times com forte background em infrastructure-as-code ganham com Beats Autônomos; times que precisam de guardrails e observabilidade centralizada ganham com Fleet Server.

**Recomendação prática**: Comece com **Beats Autônomos** se tiver dúvida. É mais fácil evoluir para Fleet Server depois do que o caminho inverso.

---

## Referências

1. [Elastic Fleet Server Documentation](https://www.elastic.co/docs/reference/fleet/deployment-models)
2. [Elastic Beats Reference](https://www.elastic.co/docs/reference/beats)
3. [Infrastructure-as-Code com Beats Autônomos](https://www.elastic.co/blog/infrastructure-as-code)
4. Artigo relacionado: [[Monitoramento_de_Segurança_para_Conformidade_com_PCI-DSS|Monitoramento de Segurança para Conformidade com PCI-DSS]]
