---
title: "Conformidade PCI-DSS com Elastic SIEM: Mapeamento de Requisitos e Implementação"
tags:
  - Elasticsearch
  - PCI-DSS
  - Compliance
  - SIEM
  - ILM
  - Security
enableToc: true
---

![[elasticsearch-logo.png]]


# Como o Elastic SIEM Atende aos Requisitos Críticos do PCI-DSS

> Um guia estratégico para justificar o investimento em Elastic SIEM demonstrando cobertura dos requisitos 7, 10, e 11 do PCI-DSS, com foco em redução de risco e custo operacional.

## Introdução: Por Que PCI-DSS Importa

O **PCI Data Security Standard (PCI-DSS)** é o padrão de segurança obrigatório para qualquer organização que processa, armazena ou transmite dados de cartões de pagamento. Não é opcional — é um requisito regulatório que pode resultar em multas de até **$100.000 por dia** em caso de não conformidade.

Mas aqui está o insight estratégico: **PCI-DSS não é apenas sobre compliance** — é sobre reduzir riscos reais:

- Detecção de ameaças em tempo real
- Proteção de dados sensíveis
- Resposta rápida a incidentes
- Prova auditável de segurança

Organizações que implementam PCI-DSS corretamente veem benefícios colaterais:
- **Redução de 40-60%** no tempo de resposta a incidentes
- **Diminuição de 70%** em falsos positivos de segurança (com regras bem configuradas)
- **Economia de 30-50%** em investigações manuais (automação)
- **Conformidade contínua** em vez de esforço anual

O **Elastic SIEM** foi construído especificamente para atender a esses requisitos. Este artigo mapeia exatamente como, e por quê.

---

## Os Requisitos Críticos do PCI-DSS

O PCI-DSS tem 12 requisitos principais. Três deles são **diretamente suportados pelo Elastic SIEM**:

| Requisito | Foco | Impacto | Elastic Coverage |
| --- | --- | --- | --- |
| **Req. 7** | Restringir acesso a dados | Quem vê o quê? | RBAC + Kibana |
| **Req. 10** | Manter e revisar trilhas de auditoria | O que aconteceu quando? | Beats + Log aggregation |
| **Req. 11** | Monitorar e testar segurança | Está funcionando? | Detecção + Alertas |

Vamos explorar cada um em profundidade.

---

## Requisito 7: Restringir Acesso a Dados (PCI-DSS 7.1–7.3)

### O que o PCI-DSS Exige

> *"Restringir acesso a dados de cartões apenas para usuários cuja função requer acesso."*

Em outras palavras: **least privilege access**. Nem todo mundo deve ver logs de segurança — apenas quem precisa.

### Por Que Importa

Violações de dados frequentemente começam com acesso não autorizado:
- Um funcionário descontente vê dados que não deveria
- Um atacante comprometeu uma conta de baixo privilégio
- Um consultor temporário ficou com acesso após sair

O PCI-DSS exige que você **prove** que apenas as pessoas certas podem acessar dados sensíveis.

### Como Elastic SIEM Cobre (RBAC)

Elastic fornece **Role-Based Access Control (RBAC)** nativo, permitindo granularidade extrema:

```yaml
# Exemplo: Analista de Segurança Júnior
Pode ver:
  ✅ Logs de autenticação (SSH logins, falhas)
  ✅ Alertas de malware detectados
  ❌ Dados de cartão de crédito
  ❌ Configurações de compliance
  ❌ Dados de auditoria da equipe de finança

# Exemplo: Gerente de Compliance
Pode ver:
  ✅ Relatórios de conformidade PCI-DSS
  ✅ Mudanças de configuração (audit log)
  ✅ Alertas não resolvidos
  ❌ Payloads técnicos de malware
  ❌ Configurações de detecção (precisa de admin)
```

**Implementação Técnica**:
- Roles definidas no Elasticsearch (`_security/role/`)
- Políticas de query para filtrar dados por `event.dataset`, `user.role`, etc.
- Acesso ao Kibana restrito por aplicação (`kibana-.kibana`)
- Auditoria completa de quem acessou o quê (logged em `.security-audit-logs`)

### Impacto no Audit Anual

**Antes (sem RBAC adequado)**:
- Auditor pergunta: "Quantos usuários têm acesso a logs de segurança?"
- Resposta: "Huh... acho que uns 30?"
- Resultado: ❌ Falha na conformidade

**Depois (com Elastic RBAC)**:
- Auditor pergunta: "Prove que apenas os aprovados têm acesso"
- Resposta: "Aqui estão os 5 roles configuradas, as 12 pessoas atribuídas, e o audit log de cada acesso"
- Resultado: ✅ Aprovado

---

## Requisito 10: Trilhas de Auditoria (PCI-DSS 10.2–10.5)

Este é o **coração do PCI-DSS**. Literalmente: "Você sabe o que aconteceu ontem às 3 da manhã?"

### Req. 10.2: Registrar Ações do Usuário

> *"Implementar mecanismos de logging automáticos para todos os acessos a dados de cartões."*

Especificamente, você precisa logar:
- ✅ Acesso bem-sucedido e malsucedido a dados sensíveis
- ✅ Ações de administrador (mudanças de config)
- ✅ Execução de programas críticos
- ✅ Mudanças em logs de auditoria

### Como Elastic SIEM Cobre (Auditbeat + Filebeat)

**Auditbeat** captura eventos a nível de kernel no Linux:

```bash
# Exemplo: Auditbeat detecta um login SSH
timestamp: 2026-04-15T14:32:45Z
user.name: "john.doe"
user.id: 1001
source.ip: "203.0.113.45"
event.action: "ssh_login"
event.outcome: "success"
process.name: "sshd"
host.name: "prod-db-01"
```

**Filebeat** coleta logs do sistema:

```bash
# Exemplo: Filebeat lê /var/log/auth.log
timestamp: 2026-04-15T14:32:45Z
message: "Accepted publickey for john.doe from 203.0.113.45 port 54321 ssh2"
event.dataset: "system.auth"
event.outcome: "success"
user.name: "john.doe"
source.ip: "203.0.113.45"
```

**Resultado**: Uma trilha completa e imutável de quem fez o quê, quando, e de onde.

### Req. 10.5: Proteger Logs de Alteração Não Autorizada

> *"Logs devem ser protegidos contra alteração ou exclusão, e retenção deve ser >= 12 meses."*

Este é o **desafio real**. Qualquer pessoa com `root` pode deletar logs. Como você impede isso?

**Resposta: Index Lifecycle Management (ILM) + Immutability**

### Como Elastic SIEM Cobre (ILM Policy)

Elastic oferece **políticas automáticas de ciclo de vida** que garantem:

1. **Hot Phase (0–30 dias)**: Índices otimizados para escrita
   - Novos eventos chegam aqui
   - Performance máxima
   - Suporta busca em tempo real

2. **Warm Phase (30–90 dias)**: Otimizado para leitura
   - Forcemerge reduz segmentos (mais eficiente)
   - Pode ser movido para storage mais barato
   - Ainda buscável

3. **Cold Phase (90–365 dias)**: Arquivo imutável
   - **FREEZE**: Índices congelados, zero mutação possível
   - Nenhum usuário (nem root) pode deletar ou modificar
   - Ainda buscável se necessário
   - Storage muito barato

4. **Delete Phase (365+ dias)**: Automática
   - Deleta após 12 meses (ou conforme política)
   - Retenção controlada, não ad-hoc

**Implementação**:

```json
{
  "policy": "pci-logs-12month",
  "phases": {
    "hot": {
      "min_age": "0ms",
      "actions": {
        "rollover": { "max_age": "30d", "max_primary_shard_size": "50gb" }
      }
    },
    "warm": {
      "min_age": "3d",
      "actions": {
        "allocate": { "require": { "data": "warm" } },
        "forcemerge": { "max_num_segments": 1 }
      }
    },
    "cold": {
      "min_age": "90d",
      "actions": {
        "allocate": { "require": { "data": "cold" } },
        "freeze": {}
      }
    },
    "delete": {
      "min_age": "365d",
      "actions": { "delete": {} }
    }
  }
}
```

**Por que isso importa para compliance**:
- Auditor: "Os logs foram alterados nos últimos 12 meses?"
- Você: "Não, a política ILM garante imutabilidade automática"
- Auditor: "Posso ver evidência?"
- Você: "Sim, aqui está o hash do índice congelado e o registro de quem tentou modificar"

### Req. 10.4: Revisar Logs Regularmente

> *"Revisar todos os logs de acesso a dados de cartões diariamente."*

Ninguém quer revisar 50GB de logs por dia manualmente.

**Como Elastic SIEM Cobre (Alertas + Dashboards)**:
- Regras de detecção **automaticamente** sinalizam anomalias
- Dashboards resumem eventos críticos
- Alertas em tempo real (via email, Slack, PagerDuty)

**Exemplo**:
- Regra: "10 tentativas de login SSH falhadas em 15 segundos"
- Resultado: Alerta automático + investigação assistida

---

## Requisito 11: Monitorar e Testar Segurança (PCI-DSS 11.1–11.5)

### O que o PCI-DSS Exige

> *"Implementar processos para monitorar e testar regularmente a eficácia dos controles de segurança."*

Em tradução livre: "Prove que sua segurança está funcionando, não só implementada."

### Req. 11.3: Testes de Penetração Anuais

Você precisa fazer pen-tests anualmente e demonstrar que:
- ✅ Vulnerabilidades conhecidas foram corrigidas
- ✅ Novos controles estão funcionando
- ✅ Resposta a incidentes é rápida

### Req. 11.5: Monitoramento de Integridade de Arquivos

> *"Implementar um mecanismo de detecção de mudanças para notificar sobre alterações não autorizadas em arquivos críticos."*

Arquivos críticos: `/etc`, `/bin`, `/sbin`, diretórios de aplicação de pagamento.

### Como Elastic SIEM Cobre (File Integrity Module)

**Auditbeat tem um módulo nativo** que monitora mudanças de arquivo:

```yaml
auditbeat.modules:
  - module: file_integrity
    paths:
      - /etc                    # Configurações do sistema
      - /bin
      - /sbin
      - /usr/sbin
      - /opt/payment-app        # Aplicação de pagamento
    scan_rate_per_sec: 50MiB
    max_file_size: 100MiB
```

**Eventos gerados**:
```json
{
  "event.action": "updated",
  "file.path": "/etc/shadow",
  "file.hash.sha256": "d4f5c6e7...",
  "file.mode.mode": "0000",
  "message": "/etc/shadow was modified",
  "event.outcome": "success"
}
```

**Alerta automático**: Uma regra de detecção pode sinalizar mudanças não autorizadas:
- Admin esperava a mudança? → Ligar para confirmar
- Ninguém esperava? → Investigar imediatamente (possível breach)

---

## A Junção: ILM como Enabler de Conformidade Contínua

Aqui está o que **diferencia Elastic SIEM** de soluções ad-hoc:

| Componente | Sem SIEM | Com Elastic SIEM |
| --- | --- | --- |
| **Logging** | Cada sistema loga diferente | Unified via Beats |
| **Retenção** | Cron job manual (frágil) | **ILM automático** (robusto) |
| **Imutabilidade** | Torcer para ninguém mexer | **Freeze automático** (garantido) |
| **Auditoria** | Logs de acesso (se houver) | **Audit trail completo** em Elasticsearch |
| **Detecção** | Nada automático | **Regras de detecção** 24/7 |
| **Investigação** | Buscar em vários logs | **Kibana correlação** |

---

## Matriz: Requisito → Elastic Feature → ROI

| Requisito | O que Precisa | Feature Elastic | Resultado |
| --- | --- | --- | --- |
| **Req. 7** | Acesso restrito | RBAC + Kibana | ✅ Compliance auditável |
| **Req. 10.2** | Logs completos | Auditbeat + Filebeat | ✅ Trilha de 100% dos eventos |
| **Req. 10.5** | Imutabilidade 12 meses | ILM + Freeze | ✅ Impossível falsificar logs |
| **Req. 11.5** | Detecção de mudanças | File Integrity Module | ✅ Alertas automáticos |

---

## O Caso de Negócio: Por Que Investir em Elastic SIEM

### Cenário 1: Sem Elastic SIEM (Ferramentas Tradicionais)

```
Custo anual:
- Syslog server + storage: $30k
- Splunk licenses: $150k
- Gerenciamento manual: 2 FTEs × $150k = $300k
- Pen-test anual: $50k
- Multa por falha de compliance: $100k/dia × 1 dia = $100k (assumindo 1 dia de falha/ano)
────────────────────
TOTAL: ~$630k/ano
```

**Problemas**:
- Logs em múltiplos lugares (não auditáveis)
- Retenção manual (frágil)
- Busca lenta (investigações lentas)
- Alertas limitados (reativo vs proativo)

### Cenário 2: Com Elastic SIEM

```
Custo anual:
- Elastic Cloud (SaaS): $60k
- Gerenciamento: 1 FTE × $150k = $150k
- Pen-test (mais confiante agora): $30k
- Multa evitada: $0
────────────────────
TOTAL: ~$240k/ano
```

**Benefícios**:
- Logs centralizados + auditáveis
- Retenção automática
- Busca rápida (investigação em minutos)
- Detecção proativa (menos incidentes)

**ROI**: ~60% economia anual + redução de risco de breach

---

## Implementação Prática: Passo-a-Passo

### 1. Avaliação do Requisito Atual

```bash
# Pergunta-chave: Onde estão seus logs hoje?
- SSH logs: /var/log/auth.log (disperso por hosts)
- App logs: /var/log/myapp.log (onde está?)
- DB logs: Oracle audit trail (como acesso?)
- Cambio de config: Espera, não tem log?
```

**Resposta esperada com Elastic**:
```
Tudo em um único índice (ou namespace de índices) no Elasticsearch
Buscável
Imutável após 90 dias
Auditável (quem acessou quê)
```

### 2. Definir Política ILM

```yaml
# pci-logs-12month.json
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": { "max_age": "30d", "max_primary_shard_size": "50gb" }
        }
      },
      "warm": { "min_age": "3d", ... },
      "cold": { "min_age": "90d", "actions": { "freeze": {} } },
      "delete": { "min_age": "365d", "actions": { "delete": {} } }
    }
  }
}
```

Aplicar a todos os índices PCI: `auditbeat-pci-*`, `filebeat-pci-*`, etc.

### 3. Configurar RBAC

```bash
POST _security/role/compliance_officer_pci
{
  "cluster": [],
  "indices": [{
    "names": ["auditbeat-pci-*", "filebeat-pci-*"],
    "privileges": ["read", "view_index_metadata"]
  }],
  "applications": [{
    "application": "kibana-.kibana",
    "privileges": ["feature_discover.read", "feature_dashboard.read"],
    "resources": ["*"]
  }]
}
```

### 4. Criar Dashboards e Alertas

**Dashboard Compliance Diário**:
- Eventos de autenticação (logins bem-sucedidos/falhados)
- Mudanças de arquivo em diretórios críticos
- Atividade de usuários privilegiados
- Alertas disparados (e resolvidos)

**Alertas Automáticos**:
- 5+ tentativas de login falhadas em 10 minutos → Escalate
- Mudança em `/etc/passwd` → Investigar
- Acesso a dados de cartão por usuário não autorizado → Bloquear

---

## Validação: Teste Seu Compliance

### Exercício: Simulação de Auditoria

```bash
# 1. Gere logs fake
for i in {1..100}; do
  echo "$(date) - Simulated login attempt" >> /var/log/auth.log
done

# 2. Verifique se Filebeat capturou
GET filebeat-pci-*/_search
{
  "query": { "match": { "message": "Simulated" } }
}

# 3. Confirme imutabilidade
GET auditbeat-pci-*/_ilm/explain
# Deve mostrar: "step": "complete"  (cold phase congelado)
```

---

## Conclusão: De Checkbox para Vantagem Competitiva

PCI-DSS começou como um "checkbox" — conformidade ou multa. Mas organizações sofisticadas transformaram em **vantagem competitiva**:

- **Confiança do cliente**: "Seu sistema realmente é seguro?"  → "Sim, auditado por PCI-DSS"
- **Redução de risco**: Menos breaches = menos crisis response
- **Eficiência operacional**: Automação reduz manual work
- **Velocidade de resposta**: Incidentes resolvidos em horas, não dias

**Elastic SIEM** fornece a infraestrutura para fazer isso **não como um projeto anual**, mas como **conformidade contínua**.

O resultado?
- ✅ Auditor feliz (compliance comprovada)
- ✅ CISO feliz (visibilidade 24/7)
- ✅ CFO feliz (ROI claro)
- ✅ Clientes felizes (segurança confiável)

---

## Referências

1. [PCI DSS v4.0 — Official Requirements](https://www.pcisecuritystandards.org/document_library/)
2. [Elastic SIEM Documentation](https://www.elastic.co/security)
3. [Index Lifecycle Management — Best Practices](https://www.elastic.co/docs/manage-data/lifecycle/)
4. [RBAC em Elasticsearch](https://www.elastic.co/docs/deploy-manage/users-roles/cloud-enterprise-orchestrator/manage-users-roles)
5. Artigo relacionado: [[Monitoramento_de_Segurança_para_Conformidade_com_PCI-DSS|Monitoramento de Segurança para Conformidade com PCI-DSS]]
6. Artigo relacionado: [[fleet-server-vs-beats-autonomos|Fleet Server vs Beats Autônomos]]
