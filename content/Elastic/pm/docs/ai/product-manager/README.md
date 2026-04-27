# Stack: Product Manager

Skills para assistência ao trabalho de produto: discovery, requisitos, PRD, métricas, priorização e comunicação com stakeholders. Cobrindo o ciclo completo de uma iniciativa, do problema vago ao lançamento.

---

## Skills Disponíveis

| Skill | Propósito |
|---|---|
| `discovery.md` | Conduzir sessões de discovery, banco de perguntas, síntese pós-entrevista |
| `requirements.md` | Escrever user stories, critérios de aceite, regras de negócio |
| `prd.md` | Estruturar e escrever o PRD completo |
| `metrics.md` | Definir North Star, input metrics, interpretar experimentos |
| `prioritization.md` | RICE, ICE, backlog review, comunicar decisões |
| `stakeholder-communication.md` | Mapa de stakeholders, cadência, decisões, incidentes |

---

## Fluxo das Skills

As skills cobrem o ciclo de vida completo de uma iniciativa em sequência — mas não são independentes. Cada uma alimenta a seguinte.

```
Problema identificado
        │
        ▼
   DISCOVERY          ← ponto de entrada obrigatório
        │
        │  Síntese de Discovery aprovada
        ▼
  REQUIREMENTS        ← transforma o problema em comportamentos esperados
        │
        │  User Stories + Critérios de Aceite
        ▼
      PRD             ← consolida tudo em contrato compartilhado
        │
        ├──────────────────────────┐
        ▼                          ▼
    METRICS              PRIORITIZATION
  (define o sucesso)    (decide o que entra)
        │                          │
        └──────────┬───────────────┘
                   ▼
       STAKEHOLDER COMMUNICATION
         (permeia todo o fluxo)
```

---

## Como Cada Skill Alimenta a Próxima

### Discovery → Requirements

A **Síntese de Discovery** (template no final de `discovery.md`) é a matéria-prima direta dos requisitos:

| Discovery produz | Requirements consome |
|---|---|
| "O problema é X, afeta Y pessoas, custa Z" | Escopo e justificativa dos requisitos |
| Workaround atual do usuário | Fluxos alternativos e casos de borda |
| Restrições e não-negociáveis | Regras de Negócio (RN-XXX) |
| Hipóteses confirmadas/refutadas | Premissas documentadas no PRD |

**Sinal de discovery insuficiente:** ao escrever os critérios de aceite em Gherkin, você não sabe o que colocar no "Dado que..." — faltam contexto e pré-condições reais.

### Requirements → PRD

User Stories e Critérios de Aceite alimentam diretamente a **seção Solução Proposta** e **Escopo** do PRD. O PRD não reescreve os requisitos — referencia e adiciona contexto de negócio, métricas e plano de lançamento.

`requirements.md` é o detalhe técnico de produto. `prd.md` é o contrato executivo. Um PRD sem requirements por baixo é uma apresentação. Requirements sem PRD são tickets órfãos sem contexto estratégico.

### Discovery → Metrics (paralelo, não sequencial)

As métricas nascem no discovery, não no PRD. Quando você identifica o problema ("cancelamentos levam 45 min e têm 12% de erro"), você já tem o baseline e sabe o que medir. `metrics.md` estrutura essa definição antes que o time comece a construir.

Definir métricas depois que a feature está pronta é o erro mais comum — nesse ponto, você otimiza para o que é fácil de medir.

### Metrics + Requirements → Prioritization

A priorização usa as métricas para calcular impacto (o "I" do RICE) e os requirements para estimar esforço com a engenharia. Sem esses dois insumos, o score do RICE é chute disfarçado de framework.

`prioritization.md` também decide o que **não entra** — e essa lista negativa vai explícita na seção Escopo do PRD.

### Stakeholder Communication — transversal

Não tem posição fixa no fluxo. Está presente em todos os momentos:

| Momento | O que a skill provê |
|---|---|
| Antes do discovery | Mapa de stakeholders — quem chamar para as sessões |
| Durante o discovery | Como sintetizar e confirmar o entendimento |
| Ao escrever o PRD | Quem precisa revisar (Aprovação / Consulta / Informação) |
| Durante o desenvolvimento | Cadência de updates, template quinzenal |
| Ao lançar | Briefing de suporte, comunicação de go-live |
| Quando algo dá errado | Template de comunicado de incidente |

---

## Os Três Artefatos que Precisam Existir Sempre

Se o time só conseguir manter três documentos vivos, estes são os que mais importam:

```
1. Síntese de Discovery      ← por que estamos fazendo isso
2. PRD                       ← o que estamos fazendo e o que define sucesso
3. Decision Log              ← por que decidimos o que decidimos
```

User stories, métricas detalhadas e comunicados são derivados destes três.

---

## Onde o Fluxo Costuma Quebrar

| Ponto de quebra | Sintoma | Causa raiz |
|---|---|---|
| Entre Discovery e Requirements | Requisitos sem problema documentado | Discovery pulado ou raso demais |
| Entre Requirements e PRD | PRD com seções "a definir" | Requirements incompleto — faltam casos de borda |
| Entre Metrics e PRD | Métricas vagas: "aumentar engajamento" | Métricas definidas após o discovery, não durante |
| Entre Prioritization e Escopo | Escopo inflado no PRD | Priorização feita sem custo real de esforço |
| Em Stakeholder Communication | Time descobre mudança de escopo na demo | Updates inconsistentes ou ausentes |

---

## Escolhas de Design

### Por que discovery é o ponto de entrada obrigatório?
Requisitos escritos sem discovery documentado descrevem soluções, não problemas. O agente não tem como validar se um requisito faz sentido sem o contexto do problema que originou. Discovery força a pergunta "por quê?" antes do "o quê?".

### Por que requirements e PRD são skills separadas?
Requisitos têm audiência técnica — engenharia e QA precisam de critérios de aceite detalhados, casos de borda e regras de negócio precisas. PRD tem audiência executiva — liderança e stakeholders precisam de contexto, métricas e decisões de escopo. Misturar os dois gera documentos que servem mal a ambos os públicos.

### Por que metrics aparece como paralela e não posterior ao PRD?
Se métricas são definidas depois que o PRD está escrito, a tendência é escolher o que já é fácil de instrumentar, não o que realmente mede o valor entregue. Definir métricas durante o discovery, enquanto o problema ainda está fresco, força a pergunta: "como saberemos que resolvemos o problema certo?".

### Por que stakeholder communication é transversal?
Comunicação não é uma fase — é uma prática contínua. Ter uma skill separada evita que seja tratada como tarefa pontual ("vou comunicar quando lançar") e lembra que alinhamento precisa acontecer antes de cada marco, não depois.

### Por que o agente carrega skills por momento e não todas de uma vez?
Carregar todas as skills simultaneamente polui o contexto com convenções que não são relevantes para a tarefa em andamento. Escrever critérios de aceite não requer o fluxo de RICE. Preparar um comunicado de incidente não requer a estrutura do PRD. Contexto focado produz output mais preciso.
