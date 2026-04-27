# Priorização

Como tomar e comunicar decisões de priorização com critérios explícitos, rastreáveis e defensáveis.

---

## Princípios

- Priorizar é decidir o que não fazer — se tudo é prioridade, nada é
- Critérios explícitos são mais justos que intuição — e mais fáceis de defender
- Envolva engenharia na estimativa de esforço antes de priorizar — custo muda tudo
- Revise a priorização quando o contexto muda, não apenas no trimestre seguinte
- Comunique o porquê de cada decisão, especialmente as impopulares

---

## Frameworks de Priorização

### RICE

Pontuação = (Reach × Impact × Confidence) / Effort

| Fator | O que mede | Como estimar |
|---|---|---|
| **Reach** | Quantas pessoas afetadas por período | Usuários/semana afetados |
| **Impact** | Intensidade do impacto por pessoa | 0.25 = mínimo, 0.5 = baixo, 1 = médio, 2 = alto, 3 = massivo |
| **Confidence** | Certeza nas estimativas | 100% = alta, 80% = média, 50% = baixa |
| **Effort** | Trabalho necessário | Person-months |

**Quando usar:** backlog com muitos itens de natureza diferente. O score cria uma ordem inicial — não substitui o julgamento final.

---

### ICE (simplificado)

Score = Impact × Confidence × Ease (inverso do esforço)

Cada dimensão de 1 a 10. **Quando usar:** priorização rápida em reunião, sem dados precisos.

---

### Matriz de Impacto × Esforço

```
Alto impacto │ Quick wins ★  │  Grandes apostas
             │               │
             ├───────────────┼───────────────
             │               │
Baixo impacto│ Descarte      │  Não vale agora
             │               │
             └───────────────┴───────────────
                Baixo esforço   Alto esforço
```

- **Quick wins:** faça agora
- **Grandes apostas:** planeje com cuidado, valide premissas antes
- **Descarte:** elimine do backlog, não desperdice capacidade
- **Não vale agora:** arquive, revisit se contexto mudar

---

### Oportunidade de Mercado (Jobs to Be Done)

Para decidir em quais segmentos ou problemas investir:

```
Oportunidade = Importância × (1 - Satisfação atual)
```

- Alta importância + baixa satisfação = oportunidade alta
- Alta importância + alta satisfação = mercado dominado, difícil de vencer
- Baixa importância = não é o problema certo a resolver

---

## Backlog Review

Faça periodicamente (sugestão: início de cada ciclo de planejamento):

**1. Inventário:** liste tudo que está no backlog com uma linha de descrição
**2. Validade:** remova itens que perderam contexto ou relevância — backlog não é arquivo
**3. Classificação:** separe em horizontes — agora (próximas 2 sprints), em breve (próximo trimestre), algum dia (backlog longo)
**4. Score:** aplique o framework escolhido nos itens do horizonte "agora"
**5. Sanity check:** o resultado faz sentido estratégico? Ajuste com julgamento onde necessário

---

## Comunicando Decisões de Priorização

Quando alguém questionar por que X não está sendo feito:

**Estrutura:**
1. Reconheça o valor do item questionado
2. Explique o critério usado para priorizar
3. Diga o que veio antes e por quê
4. Indique quando X será revisitado

```
"[X] é importante e está no backlog. Neste ciclo priorizamos [Y] porque [critério específico:
impacto em receita, dependência técnica, janela de mercado]. Vamos revisitar [X] no
planejamento do próximo trimestre. Se o contexto mudar antes disso, me avise."
```

**O que evitar:**
- "Não temos tempo" — isso não é critério, é sintoma
- "Está em análise" sem prazo ou responsável
- Prometer revisão sem agendar

---

## Template de Planejamento de Ciclo

```markdown
## Planejamento — [Time] — [Ciclo/Trimestre]

### Capacidade disponível
- Engenharia: [X] sprints × [Y] devs = [Z] story points estimados
- Reserva para bugs e operacional: [%]
- Capacidade líquida para features: [story points]

### Contexto estratégico
[O que mudou desde o último ciclo que influencia prioridades]

### Iniciativas priorizadas

| # | Iniciativa | Impacto esperado | Esforço | Score | Status |
|---|---|---|---|---|---|
| 1 | [Nome] | [Métrica + valor] | [P/M/G] | [score] | Confirmada |
| 2 | [Nome] | [Métrica + valor] | [P/M/G] | [score] | Confirmada |
| 3 | [Nome] | [Métrica + valor] | [P/M/G] | [score] | Se couber |

### O que ficou de fora e por quê
| Iniciativa | Motivo de exclusão | Quando revisitar |
|---|---|---|
| [Nome] | [Razão explícita] | [Próximo ciclo / Quando X acontecer] |

### Riscos e dependências críticas
[O que pode mudar esta priorização antes do fim do ciclo]

### Critérios para repriorização emergencial
[Em que situações vamos parar o planejado para atender algo novo]
```

---

## Sinais de que a Priorização Está Quebrada

- O time muda de foco mais de uma vez por sprint sem nova informação relevante
- Stakeholders conseguem inserir itens no topo do backlog sem critério explícito
- "Urgente" é o critério mais comum de priorização
- O time não sabe por que está fazendo o que está fazendo
- Itens ficam no backlog por mais de 6 meses sem decisão de fazer ou descartar

Se três ou mais desses sinais estão presentes, o problema não é de framework — é de processo e alinhamento com liderança. Isso requer conversa, não planilha.
