# Métricas de Produto

Como definir, acompanhar e interpretar métricas que orientem decisões de produto.

---

## Princípios

- Métrica sem baseline é decoração — saiba de onde você parte antes de definir para onde quer ir
- Uma métrica isolada mente — sempre acompanhe com contra-métricas que detectem efeitos colaterais
- Se você não consegue explicar a métrica em uma frase para um stakeholder não-técnico, ela é complexa demais para guiar decisões
- Métricas de vaidade (pageviews, downloads, usuários cadastrados) não revelam valor — prefira métricas de comportamento e resultado
- Medir o que é fácil de medir não é o mesmo que medir o que importa

---

## Framework: North Star + Input Metrics

### North Star Metric (NSM)
Uma única métrica que captura o valor entregue ao usuário e se correlaciona com crescimento sustentável do negócio.

```
North Star = [Quem] faz [ação que gera valor] com [frequência/volume que indica saúde]
```

Exemplos:
- Spotify: "Tempo ouvido por usuário ativo por semana"
- Airbnb: "Noites reservadas"
- Slack: "Mensagens enviadas por organização ativa por dia"

### Input Metrics (Alavancas)
Métricas que o time controla e que, quando melhoradas, movem a North Star:

```
NSM: Transações concluídas por merchant ativo por semana

Inputs:
├── Ativação: % merchants que completam 1ª transação em 7 dias
├── Engajamento: % merchants com ≥ 3 transações/semana
├── Conversão: % tentativas de transação que concluem com sucesso
└── Retenção: % merchants ativos no mês anterior que continuam ativos
```

---

## Tipos de Métricas

### Métricas de Negócio
| Tipo | Exemplos |
|---|---|
| Receita | MRR, ARR, receita por usuário (ARPU), LTV |
| Crescimento | Novos usuários, taxa de crescimento MoM |
| Retenção | Churn rate, retention rate por coorte |
| Eficiência | CAC, payback period, margem |

### Métricas de Produto
| Tipo | Exemplos |
|---|---|
| Ativação | % usuários que completam onboarding, tempo até 1º valor |
| Engajamento | DAU/MAU, frequência de uso, profundidade de uso (features usadas) |
| Retenção | D7, D30, D90 retention, cohort analysis |
| Satisfação | NPS, CSAT, CES (Customer Effort Score) |

### Métricas de Qualidade
| Tipo | Exemplos |
|---|---|
| Performance | p50, p95, p99 de latência; throughput |
| Confiabilidade | Uptime, taxa de erro, MTTR |
| Experiência | Taxa de erro de usuário, abandono de fluxo |

---

## Definindo Métricas para uma Iniciativa

Para cada iniciativa, defina antes de começar:

```markdown
## Métricas — [Nome da Iniciativa]

### Métrica primária de sucesso
**[Nome da métrica]**
- O que mede: [comportamento ou resultado]
- Baseline atual: [valor]
- Meta em [prazo]: [valor]
- Como medir: [ferramenta + query/evento + frequência]
- Responsável por acompanhar: [nome]

### Contra-métricas (o que não pode piorar)
| Métrica | Baseline | Limite de alerta |
|---|---|---|
| [Métrica] | [valor] | [threshold] |

### Guardrails (limites de segurança)
| Condição | Ação |
|---|---|
| Taxa de erro > 5% | Rollback imediato |
| Latência p95 > 1s | Investigar antes de expandir rollout |
| NPS cai > 10 pontos | Pausa no rollout + análise |
```

---

## Armadilhas Comuns

### Correlation ≠ Causation
Duas métricas que sobem juntas não significa que uma causa a outra. Antes de concluir que uma feature causou uma melhora, verifique:
- Havia outro fator simultâneo (campanha de marketing, sazonalidade)?
- O grupo de controle também melhorou?
- A melhora persiste após o efeito novidade?

### Goodhart's Law
"Quando uma métrica se torna um alvo, ela deixa de ser uma boa métrica." Quando o time otimiza para a métrica ao invés do comportamento que ela representa, surgem distorções:
- NPS melhorado por pedir avaliação só dos usuários satisfeitos
- DAU inflado por notificações que trazem usuários sem valor

Solução: combine métricas primárias com contra-métricas que detectem otimizações espúrias.

### Métricas de Lagging vs Leading
- **Lagging:** medem resultado passado (receita, churn) — difíceis de influenciar diretamente
- **Leading:** predizem resultado futuro (ativação, engajamento) — o que o time deve mover agora

Priorize leading indicators no dia a dia; use lagging para confirmar impacto de longo prazo.

---

## Leitura de Experimentos (A/B Tests)

Antes de concluir que um experimento foi positivo:

- [ ] O resultado é estatisticamente significativo? (p < 0,05, poder ≥ 80%)
- [ ] Rodou por pelo menos um ciclo semanal completo (efeito de dia da semana)?
- [ ] A métrica primária melhorou?
- [ ] As contra-métricas se mantiveram?
- [ ] O tamanho do efeito é prático, não só estatístico?
- [ ] Há segmentos onde o resultado foi diferente? (usuários novos vs. antigos, mobile vs. web)

**Quando NÃO há significância estatística:**
Ausência de evidência não é evidência de ausência. Se o experimento foi sub-potenciado (amostra pequena, duração curta), o resultado inconclusivo não significa que não há efeito.

---

## Template de Relatório de Métricas

Use para reviews periódicas e post-mortems de lançamento:

```markdown
## Relatório de Métricas — [Iniciativa] — [Período]

### Resumo
[Uma frase: a iniciativa está performando conforme esperado / abaixo / acima]

### Métrica primária
| | Baseline | Meta | Atual | Status |
|---|---|---|---|---|
| [Métrica] | [valor] | [valor] | [valor] | 🟢 / 🟡 / 🔴 |

### Contra-métricas
| Métrica | Baseline | Limite | Atual | Status |
|---|---|---|---|---|
| [Métrica] | [valor] | [valor] | [valor] | 🟢 / 🟡 / 🔴 |

### Análise
[O que os dados estão dizendo — fatos primeiro, interpretação depois]

### Hipóteses sobre desvios
[Se alguma métrica está fora do esperado, qual é a hipótese sobre o porquê]

### Ações
| Ação | Responsável | Prazo |
|---|---|---|
| [O que faremos] | [Nome] | [Data] |

### Próxima revisão
[Data e o que deve ter mudado para reavaliar]
```
