# Requisitos

Como escrever, estruturar e validar requisitos de produto que engenharia consiga implementar e QA consiga testar sem ambiguidade.

---

## Princípios

- Requisito bom é o que pode ser testado — se não dá para escrever um teste, não é requisito
- Escreva para quem vai implementar, não para quem aprovou — detalhe suficiente para não gerar perguntas na sprint
- Separe o quê do como — requisito descreve comportamento esperado, nunca a implementação técnica
- Ambiguidade em requisito vira bug em produção — quando em dúvida, seja mais explícito
- Requisitos mudam — estruture para facilitar rastreabilidade de mudanças

---

## Tipos de Requisito

### Funcionais
Descrevem o que o sistema deve fazer — comportamentos, regras de negócio, fluxos.

**Formato:** `[Sujeito] deve [verbo + objeto] quando [condição] para que [resultado]`

```
O sistema deve bloquear a tentativa de pagamento quando o saldo disponível for menor
que o valor da transação, exibindo a mensagem "Saldo insuficiente" para o usuário.

O operador deve poder cancelar um pedido em status "Aguardando pagamento" em até
30 minutos após a criação, sem necessidade de aprovação adicional.
```

### Não-funcionais
Descrevem qualidades do sistema — desempenho, segurança, disponibilidade, usabilidade.

**Sempre com número e condição de medição:**
```
A listagem de pedidos deve retornar em até 500ms para 95% das requisições
sob carga de até 1.000 usuários simultâneos.

O sistema deve estar disponível 99,9% do tempo por mês, excluindo janelas
de manutenção programadas comunicadas com 48h de antecedência.
```

### Regras de Negócio
Restrições e políticas que o produto deve respeitar — frequentemente vêm de compliance, jurídico ou operações.

```
RN-001: O limite diário de transações por CPF é de R$ 5.000,00, independente
        do canal (app, web, API).
RN-002: Transações acima de R$ 1.000,00 exigem autenticação em dois fatores.
RN-003: Dados de cartão nunca devem ser armazenados após a tokenização.
```

---

## User Stories

### Formato
```
Como [tipo de usuário],
quero [ação ou capacidade],
para que [benefício ou resultado de negócio].
```

### Critérios de Aceite — Formato Gherkin
```
Dado que [contexto/pré-condição]
Quando [ação do usuário ou evento]
Então [resultado esperado]
E [resultado adicional, se houver]
```

### Exemplo completo
```markdown
## US-042: Cancelamento de pedido pelo operador

**Como** operador de atendimento,
**quero** cancelar pedidos em aberto diretamente pelo painel,
**para que** eu não precise acionar o time de engenharia para correções manuais no banco.

### Critérios de Aceite

**CA-01: Cancelamento dentro do prazo**
- Dado que o pedido está em status "Aguardando pagamento"
- E foi criado há menos de 30 minutos
- Quando o operador clica em "Cancelar pedido" e confirma
- Então o status muda para "Cancelado"
- E o cliente recebe notificação por e-mail em até 2 minutos
- E o log registra: id do operador, timestamp, motivo

**CA-02: Cancelamento fora do prazo**
- Dado que o pedido está em status "Aguardando pagamento"
- E foi criado há mais de 30 minutos
- Quando o operador tenta cancelar
- Então o sistema exibe "Prazo de cancelamento expirado. Entre em contato com o supervisor."
- E o botão "Cancelar" fica desabilitado

**CA-03: Pedido em status incompatível**
- Dado que o pedido está em qualquer status diferente de "Aguardando pagamento"
- Quando o operador acessa a tela do pedido
- Então o botão "Cancelar" não é exibido

### Fora do Escopo
- Cancelamento automático por timeout (tratado na US-051)
- Reembolso automático (tratado na US-043)
- Cancelamento de pedidos com entrega já iniciada

### Dependências
- US-043 (Reembolso) deve estar pronta antes do deploy desta
- Serviço de notificação por e-mail operacional

### Dúvidas em Aberto
- [ ] O que acontece com pedidos criados antes da mudança? Migração necessária?
- [ ] Operador precisa informar motivo do cancelamento? Obrigatório ou opcional?
```

---

## Checklist de Qualidade de Requisito

Antes de levar para a sprint, cada requisito deve passar por:

**Clareza**
- [ ] Qualquer pessoa do time consegue ler e entender sem perguntar nada?
- [ ] Usa linguagem do domínio do usuário, não jargão interno?
- [ ] Está livre de termos ambíguos: "rápido", "fácil", "adequado", "quando necessário"?

**Completude**
- [ ] Cobre o fluxo feliz (happy path)?
- [ ] Cobre os principais fluxos alternativos?
- [ ] Cobre os fluxos de erro?
- [ ] Define comportamento em casos de borda?

**Testabilidade**
- [ ] Dá para escrever um teste automatizado a partir dos critérios de aceite?
- [ ] O resultado esperado é observável e mensurável?

**Rastreabilidade**
- [ ] Está linkado ao problema de negócio que originou?
- [ ] Tem identificador único (US-XXX, RN-XXX)?
- [ ] Dependências com outros requisitos estão explícitas?

**Viabilidade**
- [ ] Engenharia foi consultada sobre viabilidade técnica?
- [ ] Não prescreve implementação técnica específica?

---

## Matriz de Priorização MoSCoW

Use quando há mais requisitos do que capacidade para a entrega:

| Categoria | Critério | Ação |
|---|---|---|
| **Must Have** | Sem isso o produto não funciona ou viola regulação | Entra obrigatoriamente |
| **Should Have** | Importante mas há workaround temporário | Entra se couber |
| **Could Have** | Agrega valor mas impacto baixo se ausente | Backlog para próxima iteração |
| **Won't Have** | Fora de escopo desta entrega — decisão explícita | Documentado como exclusão |

Ao classificar, pergunte: "Se tirarmos isso do escopo, o que acontece com o usuário hoje?"

---

## Rastreabilidade: Problema → Requisito → Entrega

Mantenha esta cadeia documentada e navegável:

```
Problema de negócio (Discovery)
  └── Objetivo de produto (OKR / Meta)
        └── Epic
              └── User Story (US-XXX)
                    └── Critérios de Aceite
                          └── Tasks de engenharia
                                └── PR / Commit
```

Ferramentas: mantenha os IDs consistentes entre Jira/Linear, Confluence/Notion e PRD.
