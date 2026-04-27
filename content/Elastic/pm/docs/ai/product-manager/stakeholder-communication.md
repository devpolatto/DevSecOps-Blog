# Comunicação com Stakeholders

Como estruturar comunicações, alinhar expectativas e gerenciar o relacionamento com stakeholders ao longo de uma iniciativa.

---

## Princípios

- Stakeholder sem informação preenche o vazio com suposição — comunique antes de ser perguntado
- Alinhamento não é consenso — é clareza sobre quem decide o quê e por quê
- Escalada é ferramenta, não fraqueza — use quando o impasse não pode ser resolvido no seu nível
- Documente decisões e o raciocínio por trás delas — a pessoa que participou da reunião vai esquecer
- Bad news early — informar um problema cedo dá tempo para agir; informar tarde só gera crise

---

## Mapa de Stakeholders

Antes de qualquer iniciativa, mapeie:

```markdown
## Mapa de Stakeholders — [Iniciativa]

| Stakeholder | Papel | Interesse principal | Influência | Posição atual | Abordagem |
|---|---|---|---|---|---|
| [Nome/Time] | Sponsor | ROI da iniciativa | Alta | Favorável | Atualização quinzenal |
| [Nome/Time] | Usuário final | Facilitar trabalho diário | Média | Neutro | Envolver no discovery |
| [Nome/Time] | Impactado | Não perder autonomia | Alta | Resistente | Consulta antes de definir escopo |
| [Nome/Time] | Informado | Conformidade | Baixa | Neutro | Update no lançamento |
```

**Níveis de envolvimento:**
- **Aprovação**: decisão final passa por ele — envolva antes de finalizar
- **Consulta**: opinião importa e deve ser considerada — consulte antes de decidir
- **Informação**: precisa saber, não influencia — comunique após decisão

---

## Cadência de Comunicação

### Durante o desenvolvimento

| Audiência | Formato | Frequência | Conteúdo |
|---|---|---|---|
| Sponsor / Liderança | Update escrito | Quinzenal | Status, riscos, bloqueadores, próximos passos |
| Time de produto + eng | Review de sprint | A cada sprint | O que foi entregue, o que muda |
| Stakeholders impactados | Check-in | Mensal ou em marcos | Progresso, mudanças de escopo, o que precisam saber |
| Time de suporte | Briefing | Antes do lançamento | O que muda, FAQs esperadas, onde escalar |

### Template de update quinzenal

```markdown
## Update — [Nome da Iniciativa] — Semana XX

**Status:** 🟢 No prazo / 🟡 Atenção / 🔴 Em risco

**O que avançamos:**
- [Conquista 1]
- [Conquista 2]

**O que está bloqueado:**
- [Bloqueador] — Responsável: [Nome] — Prazo para desbloqueio: [Data]

**Mudanças de escopo ou prazo:**
- [Se houver — com justificativa]

**Próximos 2 sprints:**
- [O que entra]

**O que precisamos de vocês:**
- [Decisão pendente] — Prazo: [Data]
- [Aprovação necessária]
```

---

## Gerenciando Pedidos Não-Planejados

Quando um stakeholder pede uma feature ou mudança fora do planejamento:

**1. Ouça completamente antes de responder**
Entenda o problema real por trás do pedido — frequentemente há uma solução mais simples.

**2. Classifique o pedido**
- É urgente e impacta negócio diretamente? → Escale para priorização com liderança
- É importante mas não urgente? → Registre no backlog com contexto
- É uma preferência pessoal disfarçada de necessidade? → Gentilmente redirecione para o problema real

**3. Responda com clareza — sem promessas vagas**

```
"Entendo a necessidade. Vou analisar o impacto e trazer uma proposta até [data].
Para tomar essa decisão, preciso entender: isso bloqueia algo específico hoje,
ou estamos pensando em algo para os próximos meses?"
```

**4. Feche o loop — sempre**
Toda solicitação recebida deve ter uma resposta, mesmo que seja "decidimos não fazer e aqui está o porquê."

---

## Documentando Decisões

Use um Decision Log para cada iniciativa. Toda decisão relevante de produto deve ser registrada:

```markdown
## Decision Log — [Iniciativa]

### DEC-001 — [Título da decisão]
**Data:** YYYY-MM-DD
**Participantes:** [Nomes]
**Decisão:** [O que foi decidido — uma frase]
**Contexto:** [Por que essa decisão foi necessária]
**Alternativas consideradas:**
- Opção A: [descrição] — descartada porque [razão]
- Opção B: [descrição] — descartada porque [razão]
**Consequências:** [O que muda com essa decisão, o que fica de fora]
**Revisão:** [Quando e sob quais condições essa decisão deve ser revisitada]
```

---

## Gerenciando Conflitos e Impasses

**Quando dois stakeholders querem coisas diferentes:**

1. Entenda o interesse por trás de cada posição — frequentemente os interesses são compatíveis mesmo que as posições não sejam
2. Traga dados — opiniões conflitam, dados convergem
3. Proponha um experimento — "Vamos tentar X por 30 dias e medir Y"
4. Esclareça quem decide — se não há consenso e a decisão precisa ser tomada, deixe claro quem tem a palavra final
5. Escale conscientemente — documente o impasse, as alternativas e o que você recomenda, e leve para quem pode resolver

**O que não fazer:**
- Fingir que o conflito não existe
- Prometer para cada lado o que ele quer ouvir
- Tomar partido publicamente antes de entender todos os lados
- Deixar o impasse sem dono e sem prazo

---

## Comunicando Más Notícias

Quando algo der errado — atraso, bug em produção, mudança de escopo forçada:

**Estrutura:**
1. **O que aconteceu** — fatos, sem spin
2. **Impacto** — o que isso afeta concretamente
3. **O que já fizemos** — ações tomadas até agora
4. **Próximos passos** — o que vem a seguir e quando teremos mais informações
5. **O que precisamos** — se há algo que os stakeholders podem fazer para ajudar

**Tom:** direto e calmo. Pânico é contágio. Clareza é antídoto.

```markdown
## Comunicado — Incidente em Produção — [Data]

**O que aconteceu:** O deploy da versão 2.3.1 introduziu um bug que afeta
o fluxo de cancelamento de pedidos para operadores com perfil "atendimento".

**Impacto:** Aproximadamente 150 cancelamentos não puderam ser processados
entre 14h e 15h30 de hoje. Os pedidos afetados estão em estado consistente
e podem ser cancelados manualmente.

**O que já fizemos:** Revertemos o deploy às 15h32. O fluxo voltou ao normal.
Engenharia está investigando a causa raiz.

**Próximos passos:**
- Até 18h: lista completa dos pedidos afetados
- Até amanhã 10h: relatório de causa raiz e plano de correção
- Cancelamentos manuais sendo processados pelo time de operações agora

**O que precisamos de vocês:** Nada por enquanto. Manteremos vocês informados
a cada 2 horas até resolução completa.
```
