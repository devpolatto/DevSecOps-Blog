---
title: Escrevendo um bom CLAUDE.md
tags:
  - Claude
  - AI
  - best-practices
  - agents
enableToc: true
---

# Os LLMs são (principalmente) apátridas

LLMs são funções sem estado. Seus pesos ficam congelados quando são usados para inferência, então eles não aprendem com o tempo. A única coisa que o modelo sabe sobre sua base de código são os tokens que você coloca nela.

Da mesma forma, os arneses de agentes de codificação, como o Claude Code, geralmente exigem que você gerencie explicitamente a memória dos agentes. `CLAUDE.md` (ou `AGENTS.md`) é o único arquivo que por padrão entra cada conversa você tem com o agente.

Isto tem três implicações importantes:

- Os agentes de codificação não sabem absolutamente nada sobre sua base de código no início de cada sessão.
- O agente deve ser informado sobre qualquer coisa que seja importante saber sobre sua base de código toda vez que você iniciar uma sessão.
- `CLAUDE.md` é a maneira preferida de fazer isso.

# CLAUDE.md integra Claude à sua base de código

Como Claude não sabe nada sobre sua base de código no início de cada sessão, você deve usar `CLAUDE.md` para integrar Claude em sua base de código. Em alto nível, isso significa que deve abranger:

- **O QUE**: conte a Claude sobre a tecnologia, sua pilha, a estrutura do projeto. Dê a Claude um mapa da base de código. Isto é especialmente importante em monorepos! Diga a Claude quais são os aplicativos, quais são os pacotes compartilhados e para que serve tudo para que ele saiba onde procurar as coisas
- **PORQUÊ**: diga a Claude o propósito do projeto e o que tudo está fazendo no repositório. Qual é o propósito e a função das diferentes partes do projeto?
- **COMO**: diga a Claude como isso deve funcionar no projeto. Por exemplo, você usa bun em vez de node? Você deseja incluir todas as informações necessárias para realmente fazer um trabalho significativo no projeto. Como Claude pode verificar as mudanças de Claude? Como ele pode executar testes, verificações de tipo e etapas de compilação?

Mas a maneira como você faz isso é importante! Não tente encher todos os comandos que Claude pode precisar executar em seu `CLAUDE.md` arquivo - você obterá resultados abaixo do ideal.

# Claude muitas vezes ignora CLAUDE.md

Independentemente do modelo que você esteja usando, você pode notar que Claude frequentemente ignora seu CLAUDE.md conteúdo do arquivo.

Você mesmo pode investigar isso colocando um proxy de registro entre a CLI do código Claude e a API Anthropic usando ANTHROPIC_BASE_URL. O código Claude injeta o seguinte lembrete do sistema com seu CLAUDE.md arquivo na mensagem do usuário para o agente:

```xml
<system-reminder>
      IMPORTANT: this context may or may not be relevant to your tasks. 
      You should not respond to this context unless it is highly relevant to your task.
</system-reminder>
```

Como resultado, Claude ignorará o conteúdo do seu CLAUDE.md se decidir que não é relevante para a sua tarefa atual. Quanto mais informações você tiver no arquivo que não esteja universalmente aplicável para as tarefas em que você o tem trabalhando, é mais provável que Claude ignore suas instruções no arquivo.

Por que a Anthropic adicionou isso? É difícil dizer com certeza, mas podemos especular um pouco. A maioria CLAUDE.md os arquivos que encontramos incluem várias instruções no arquivo que não são amplamente aplicável. Muitos usuários tratam o arquivo como uma forma de adicionar "hotfixes" a comportamentos dos quais não gostaram, anexando muitas instruções que não eram necessariamente amplamente aplicáveis.

Só podemos supor que a equipe do Código Claude descobriu que, ao dizer a Claude para ignorar as instruções ruins, o arnês realmente produziu melhores resultados.

# Criando um bom CLAUDE.md

A seção a seguir fornece uma série de recomendações sobre como escrever um bom CLAUDE.md arquivo a seguir melhores práticas de engenharia de contexto.

Sua quilometragem pode variar. Nem todas essas regras são necessariamente ideais para todas as configurações. Como qualquer outra coisa, fique à vontade para quebrar as regras uma vez...

1. você entende quando e por que não há problema em quebrá-los
2. você tem um bom motivo para fazer isso

**Referências**

 -[Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)