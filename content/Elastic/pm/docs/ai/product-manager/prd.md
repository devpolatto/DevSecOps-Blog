# PRD — Product Requirements Document

Como estruturar, escrever e manter um PRD que sirva de contrato compartilhado entre produto, engenharia, design e negócio.

---

## Princípios

- PRD não é especificação técnica — é o problema, o contexto e o resultado esperado
- Um bom PRD elimina reuniões de alinhamento desnecessárias, não cria mais
- Escreva assumindo que o leitor não estava em nenhuma das reuniões anteriores
- PRD vivo: atualize quando premissas mudam, não crie uma versão 2 do zero
- Curto e completo é melhor que longo e exaustivo — cada seção deve ganhar seu espaço

---

## Estrutura do PRD

### 1. Cabeçalho

```markdown
# [Nome da Iniciativa]

| Campo | Valor |
|---|---|
| Status | Draft / Em Revisão / Aprovado / Em Desenvolvimento / Lançado / Descontinuado |
| PM | [Nome] |
| Engenharia | [Tech Lead] |
| Design | [Designer] |
| Criado em | YYYY-MM-DD |
| Última atualização | YYYY-MM-DD |
| Links | [Discovery] · [Figma] · [Epic no Jira] · [Métricas] |
```

---

### 2. TL;DR (Obrigatório — máximo 5 linhas)

Responde: o que estamos fazendo e por quê? Quem lê só isso deve sair capaz de explicar a iniciativa para alguém.

```markdown
## TL;DR

Estamos implementando cancelamento de pedidos pelo painel de operações para eliminar
a dependência de intervenção manual no banco de dados, que hoje custa em média 45 minutos
de engenharia por semana e causa erros em 12% dos casos. O impacto esperado é reduzir
esse custo a zero e aumentar o NPS de operadores de 32 para 50+ em 90 dias.
```

---

### 3. Contexto e Problema

**O quê e por quê agora.** Não a solução — o problema.

```markdown
## Contexto e Problema

### Situação atual
[Descreva como o processo funciona hoje — seja específico, use números quando possível]

### Problema
[Qual é a dor concreta — para o usuário, para o negócio, para o time interno]

### Por que agora
[O que mudou que torna isso prioritário hoje: crescimento, regulação, feedback acumulado,
oportunidade de mercado, dependência de outra iniciativa]

### Evidências
- [Dado quantitativo: "X% dos tickets de suporte mencionam Y"]
- [Dado qualitativo: citação direta de entrevista de usuário]
- [Dado de negócio: custo, churn, receita em risco]
```

---

### 4. Objetivos e Métricas de Sucesso

**O que define "funcionou"?** Sem métricas, não há como saber se valeu a pena.

```markdown
## Objetivos e Métricas de Sucesso

### Objetivo principal
[Uma frase. Ex: "Eliminar a dependência de engenharia para cancelamentos operacionais"]

### Métricas primárias (como medimos sucesso)
| Métrica | Baseline atual | Meta em 90 dias | Como medir |
|---|---|---|---|
| Tempo médio de cancelamento | 45 min | < 2 min | Amplitude |
| Tickets de suporte sobre cancelamento | 80/semana | < 10/semana | Zendesk |

### Métricas secundárias (o que não pode piorar)
| Métrica | Baseline atual | Limite aceitável |
|---|---|---|
| NPS operadores | 32 | ≥ 32 |
| Taxa de erro em cancelamentos | 12% | ≤ 5% |

### O que não vamos medir nesta entrega
[Explique o que está fora do escopo de medição e por quê]
```

---

### 5. Usuários e Stakeholders

```markdown
## Usuários e Stakeholders

### Usuário primário
**[Persona / Papel]** — [descrição em 2 linhas do contexto de uso]

Comportamento atual: [o que fazem hoje]
Necessidade principal: [o que precisam conseguir]
Critério de sucesso para eles: [como saberão que funcionou]

### Usuários secundários
[Quem mais é afetado, mesmo que não seja o usuário principal]

### Stakeholders
| Stakeholder | Interesse | Nível de envolvimento |
|---|---|---|
| [Nome/Time] | [O que ganham ou perdem] | Aprovação / Consulta / Informação |
```

---

### 6. Escopo

**O que entra e o que não entra — seja explícito nos dois.**

```markdown
## Escopo

### Esta entrega inclui
- [Feature ou comportamento 1]
- [Feature ou comportamento 2]

### Esta entrega não inclui (explicitamente)
- [O que foi considerado e decidido excluir, e por quê]
- [Itens de backlog futuro que poderiam ser confundidos com este escopo]

### Premissas
- [O que estamos assumindo como verdade para que isso funcione]
- Ex: "Assumimos que o serviço de notificação por e-mail estará operacional"

### Restrições
- [Limitações técnicas, regulatórias, de prazo ou de recursos que moldam a solução]
```

---

### 7. Solução Proposta

**Descreva o comportamento esperado, não a implementação técnica.**

```markdown
## Solução Proposta

### Visão geral
[Parágrafo descrevendo a solução em linguagem de negócio]

### Fluxos principais

#### Fluxo 1: [Nome]
[Descreva o fluxo do ponto de vista do usuário, passo a passo]
1. Usuário acessa X
2. Vê lista de Y com status Z
3. Clica em "Cancelar" → sistema valida condição A
4. Se válido: status muda para "Cancelado" + notificação disparada
5. Se inválido: mensagem de erro descritiva

#### Fluxo 2: [Nome]
[...]

### User Stories
[Link para as US detalhadas, ou inclua aqui se o PRD for o único documento]

### Mockups / Protótipos
[Link para Figma ou inclua screenshots com anotações]
```

---

### 8. Requisitos Não-Funcionais

```markdown
## Requisitos Não-Funcionais

| Dimensão | Requisito | Justificativa |
|---|---|---|
| Performance | Resposta em < 500ms p95 | Fluxo operacional crítico |
| Disponibilidade | 99,9% uptime | Operadores trabalham 24/7 |
| Segurança | Log de auditoria para toda ação | Conformidade com política interna |
| Escalabilidade | Suportar 10x o volume atual | Crescimento projetado em 12 meses |
```

---

### 9. Dependências e Riscos

```markdown
## Dependências e Riscos

### Dependências
| Dependência | Time responsável | Status | Impacto se atrasar |
|---|---|---|---|
| Serviço de notificação e-mail | Plataforma | Em desenvolvimento | Bloqueia CA-01 |
| Aprovação do jurídico para o fluxo de cancelamento | Jurídico | Pendente | Bloqueia lançamento |

### Riscos
| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Serviço de e-mail atrasar | Média | Alto | Lançar sem notificação e-mail em fase 1 |
| Operadores cancelarem pedidos indevidamente | Baixa | Alto | Requerer confirmação + campo de motivo obrigatório |
| Volume de cancelamentos maior que o esperado | Baixa | Médio | Monitorar na primeira semana, definir threshold de alerta |
```

---

### 10. Plano de Lançamento

```markdown
## Plano de Lançamento

### Estratégia de rollout
- [ ] Feature flag — ativar gradualmente: 5% → 25% → 100%
- [ ] Rollout por segmento: operadores internos primeiro, depois parceiros
- [ ] Lançamento direto (sem flag)

### Critérios de go/no-go
- [ ] Todos os critérios de aceite validados em homologação
- [ ] Runbook de rollback documentado e testado
- [ ] Monitoramento e alertas configurados
- [ ] Time de suporte treinado

### Plano de rollback
[O que fazemos se algo der errado nas primeiras 24h]

### Comunicação
- Interna: [o que comunicar, para quem, quando]
- Externa: [release notes, changelog, comunicado para usuários se aplicável]
```

---

### 11. Histórico de Mudanças

```markdown
## Histórico de Mudanças

| Data | Versão | Mudança | Autor |
|---|---|---|---|
| YYYY-MM-DD | 1.0 | Criação do documento | [Nome] |
| YYYY-MM-DD | 1.1 | Adicionado requisito de log de auditoria após revisão com jurídico | [Nome] |
```

---

## Checklist de PRD Pronto para Revisão

**Conteúdo**
- [ ] TL;DR escrito — qualquer pessoa do time entende em 30 segundos
- [ ] Problema descrito com dados, não com opinião
- [ ] Métricas de sucesso definidas com baseline e meta
- [ ] Escopo negativo explícito (o que não entra)
- [ ] Premissas e restrições documentadas
- [ ] Riscos mapeados com mitigação

**Processo**
- [ ] Engenharia revisou e não tem bloqueadores técnicos óbvios
- [ ] Design revisou (se houver interface)
- [ ] Stakeholders-chave foram consultados
- [ ] Discovery que originou o PRD está linkado

**Qualidade**
- [ ] Não prescreve implementação técnica
- [ ] Não usa termos ambíguos sem definição
- [ ] Todas as siglas e termos de domínio estão explicados na primeira ocorrência

---

## Antipadrões Comuns

| Antipadrão | Por que é problema | Como corrigir |
|---|---|---|
| PRD como especificação técnica | Engenharia perde autonomia de solução; ficará desatualizado rápido | Descreva comportamento, não implementação |
| Métricas de sucesso ausentes | Impossível saber se funcionou ou priorizar iterações | Defina antes de começar, mesmo que imperfeitas |
| Escopo implícito | O que não está escrito vira suposição diferente em cada cabeça | Escreva o que não entra, explicitamente |
| "Será definido posteriormente" em seções críticas | Bloqueia engenharia ou cria retrabalho | Deixe a seção em branco com `[EM ABERTO — proprietário: X, prazo: Y]` |
| PRD atualizado só na criação | Documento fica inútil após primeira sprint | Cada mudança de escopo deve ser registrada no histórico |
