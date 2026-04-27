# Git Workflow — Acqio

## Branches

| Tipo | Padrão | Exemplo |
|---|---|---|
| Feature | `feat/<ticket>-descricao` | `feat/PAY-123-adicionar-validacao-cpf` |
| Correção | `fix/<ticket>-descricao` | `fix/PAY-456-corrigir-timeout-grpc` |
| Manutenção | `chore/<ticket>-descricao` | `chore/PAY-789-atualizar-acqio-commons` |

- Branch base: `main`
- Um PR por mudança lógica — não agrupe features com refatorações

## Commits

Seguir [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(payment): adicionar validação de CPF no checkout
fix(grpc): corrigir timeout em chamadas ao serviço de fraude
chore(deps): atualizar acqio-commons para 2.3.1
docs(ai): atualizar convenções de testes
refactor(service): extrair lógica de retry para PaymentRetryService
test(payment): adicionar cenário de saldo insuficiente
```

- Um commit por mudança lógica
- Mensagem no imperativo e em português
- Escopo entre parênteses quando aplicável

## Pull Requests

- Título segue o mesmo padrão do commit principal
- Descrição: o quê mudou e por quê — não como
- PRs pequenos e focados: prefira múltiplos PRs a um PR gigante
- Nunca abra PR sem o gate de qualidade passando localmente

## O que Nunca Commitar

- Credenciais, secrets, tokens ou chaves de API
- Arquivos `.env` com valores reais
- Connection strings com usuário/senha
- Binários gerados pelo build
- Arquivos de IDE (`.idea/`, `.vscode/`) — use `.gitignore`
