# Discovery

Guia para conduzir sessões de discovery com stakeholders, usuários e times de engenharia. O objetivo é transformar um problema vago em entendimento compartilhado antes de qualquer solução ser proposta.

---

## Princípios

- Descubra o problema real antes de qualquer solução — sintomas e causas raiz são diferentes
- "Por quê?" é mais valioso do que "O quê?" — pergunte pelo menos três vezes antes de aceitar uma resposta como suficiente
- Silêncio é dado — o que as pessoas não mencionam espontaneamente importa tanto quanto o que mencionam
- Validar premissas cedo custa pouco; validar tarde custa muito
- Nunca entre numa sessão de discovery sem hipóteses — elas estruturam o que você precisa confirmar ou refutar

---

## Estrutura de uma Sessão de Discovery

### Antes da sessão
- Defina claramente o que você já sabe e o que precisa descobrir
- Liste as hipóteses que quer testar (ex: "acreditamos que o problema é X porque Y")
- Identifique quem deve estar na sala: usuário final, stakeholder de negócio, engenharia, suporte
- Limite a 60 minutos — sessões longas perdem foco

### Abertura (5 min)
- Contextualize sem contaminar: explique o propósito sem revelar hipóteses ou soluções
- "Estamos aqui para entender melhor como vocês fazem X hoje, não para apresentar soluções"
- Peça permissão para tomar notas e fazer perguntas de acompanhamento

### Exploração (40 min)
- Siga o roteiro de perguntas mas abandone-o quando uma resposta abre algo inesperado
- Documente literalmente o que é dito — não interprete em tempo real
- Marque momentos de alta emoção (frustração, entusiasmo) — eles indicam onde está a dor real

### Síntese (10 min)
- Resuma o que você ouviu e confirme com o entrevistado
- "Se eu entendi corretamente, o maior problema hoje é... Está certo?"
- Liste o que ainda ficou em aberto

### Após a sessão (antes da próxima)
- Escreva os insights enquanto a memória está fresca
- Separe fatos observados de interpretações suas
- Atualize as hipóteses: confirmadas, refutadas, novas

---

## Banco de Perguntas por Contexto

### Entender o problema atual
```
Descreva como você faz [tarefa] hoje, do início ao fim.
O que mais te frustra nesse processo?
Quando foi a última vez que isso deu errado? O que aconteceu?
Você já tentou resolver isso de outra forma? O que funcionou e o que não funcionou?
Se você pudesse mudar uma coisa nesse processo hoje, o que seria?
```

### Escalar o impacto
```
Com que frequência isso acontece?
Quantas pessoas são afetadas?
O que acontece quando isso falha? Qual é o custo — em tempo, dinheiro, reputação?
Isso bloqueia completamente o trabalho ou é um inconveniente?
Você tem dados sobre isso?
```

### Entender o usuário
```
Quem mais passa por esse problema além de você?
Como os outros na sua equipe lidam com isso?
O que você precisa saber ou fazer antes de conseguir completar essa tarefa?
O que uma solução boa precisaria ter para você confiar nela?
```

### Desafiar soluções propostas (quando o stakeholder já chega com solução)
```
Que problema específico essa solução resolve?
Se essa solução existisse amanhã, o que mudaria no seu dia a dia?
Quais outros problemas ficariam sem solução?
O que poderia fazer essa solução não funcionar?
Já tentamos algo parecido antes?
```

### Entender contexto de negócio
```
Por que isso é prioridade agora e não há seis meses?
Quem mais tem interesse no resultado — positivo ou negativo?
O que define sucesso para você daqui a três meses?
O que definitivamente não pode piorar?
Existe um prazo ou evento externo que estamos amarrados?
```

### Perguntas de fechamento
```
Tem alguma coisa importante que eu não perguntei?
Se você pudesse me dar um único conselho para não errarmos nessa, qual seria?
Quem mais eu deveria conversar?
```

---

## Sinais de Discovery Incompleto

Avance para a próxima fase somente quando conseguir responder todas:

- [ ] O problema está descrito em termos de comportamento do usuário, não de feature solicitada
- [ ] Sabemos quem é afetado e com qual frequência
- [ ] Sabemos o que acontece hoje quando o problema ocorre (workaround atual)
- [ ] Sabemos o custo do problema (tempo, dinheiro, satisfação)
- [ ] Sabemos quem são os stakeholders e quais são seus critérios de sucesso
- [ ] As hipóteses iniciais foram confirmadas, refutadas ou ajustadas
- [ ] Identificamos restrições e não-negociáveis

---

## Template de Síntese Pós-Discovery

```markdown
## Síntese de Discovery — [Nome da Iniciativa]

**Data:** YYYY-MM-DD
**Participantes:** [nomes e papéis]

### O problema
[Uma ou duas frases descrevendo o problema em linguagem do usuário, não de produto]

### Quem é afetado
[Perfil do usuário, frequência, escala]

### Situação atual
[Como resolvem hoje — workarounds, ferramentas alternativas, processos manuais]

### Custo do problema
[Impacto quantificável: tempo perdido, erros, churn, receita, satisfação]

### O que aprendemos que não sabíamos antes
[Surpresas, premissas refutadas, nuances inesperadas]

### Hipóteses atualizadas
| Hipótese | Status | Evidência |
|---|---|---|
| ... | Confirmada / Refutada / Incerta | ... |

### Perguntas ainda em aberto
[O que ainda não sabemos e precisamos descobrir]

### Próximos passos
[O que fazemos com isso agora]
```
