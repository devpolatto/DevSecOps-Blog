# Stack: Product Manager

> Para entender esta stack, leia `docs/ai/product-manager/README.md`
> Gerado por `scripts/generate-ai-rules.sh` — não edite manualmente.
> Para atualizar: `./scripts/generate-ai-rules.sh product-manager ./pm/`
> Fonte: `docs/ai/`

> Claude lê este arquivo ao abrir o projeto.
> Para cada tarefa, identifique o momento da iniciativa, carregue a skill
> correspondente em `docs/ai/product-manager/` e siga suas instruções.
> Para entender o fluxo completo entre skills, leia `docs/ai/product-manager/README.md`.

---

## Roteamento por Momento da Iniciativa

Antes de qualquer tarefa de produto, identifique em qual momento você está
e leia a skill correspondente em `docs/ai/product-manager/`.

| Você está fazendo | Skill a carregar |
|---|---|
| Preparar perguntas para entrevista com usuário ou stakeholder | `discovery.md` |
| Conduzir ou sintetizar uma sessão de discovery | `discovery.md` |
| Escrever user stories ou critérios de aceite | `requirements.md` |
| Definir regras de negócio ou fluxos de erro | `requirements.md` |
| Montar ou revisar um PRD | `prd.md` + `metrics.md` |
| Preparar o PRD para aprovação de liderança | `prd.md` + `stakeholder-communication.md` |
| Definir métricas de sucesso para uma iniciativa | `metrics.md` |
| Interpretar resultados de A/B test ou dashboard | `metrics.md` |
| Decidir o que entra na sprint ou no trimestre | `prioritization.md` |
| Fazer backlog review | `prioritization.md` |
| Redigir update semanal para stakeholders | `stakeholder-communication.md` |
| Comunicar mudança de escopo ou atraso | `stakeholder-communication.md` |
| Redigir comunicado de incidente em produção | `stakeholder-communication.md` |
| Revisar se uma iniciativa está pronta para o kick-off | todas as skills — checklist cruzado |

## Fluxo entre Skills

```
Problema identificado
        │
        ▼
   discovery.md        ← ponto de entrada obrigatório
        │
        ▼
  requirements.md      ← transforma problema em comportamentos esperados
        │
        ▼
      prd.md           ← consolida tudo em contrato compartilhado
        │
        ├──────────────────────────┐
        ▼                          ▼
    metrics.md          prioritization.md
  (define sucesso)      (decide o que entra)
        │                          │
        └──────────┬───────────────┘
                   ▼
    stakeholder-communication.md
         (permeia todo o fluxo)
```

Para entender como cada skill alimenta a próxima, leia `docs/ai/product-manager/README.md`.

## Os Três Artefatos Essenciais

1. **Síntese de Discovery** — por que estamos fazendo isso (template em `discovery.md`)
2. **PRD** — o que estamos fazendo e o que define sucesso (estrutura em `prd.md`)
3. **Decision Log** — por que decidimos o que decidimos (template em `stakeholder-communication.md`)

---

## Como Operar

1. **Identifique o momento** — use a tabela acima para escolher a skill
2. **Leia a skill** em `docs/ai/product-manager/<skill>.md` antes de gerar output
3. **Use os templates** da skill — não invente estrutura própria
4. **Se a tarefa cruza dois momentos** (ex: PRD + métricas), carregue as duas skills
5. **Dúvida sobre o fluxo?** Leia `docs/ai/product-manager/README.md`

## Comportamento

- Entregue o documento ou artefato solicitado diretamente — sem preâmbulo
- Use a linguagem do domínio de produto: problema, usuário, métrica, escopo, critério de aceite
- Quando faltar informação essencial para preencher um template, aponte o campo em branco
  com `[EM ABERTO — responsável: X, prazo: Y]` — não invente conteúdo
- Nunca coloque solução onde o template pede problema, nem implementação onde pede comportamento
