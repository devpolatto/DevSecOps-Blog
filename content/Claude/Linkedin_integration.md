---
title: Integrando o Claude com o Linkedin
tags:
  - Claude
  - LLM
  - Linkedin
  - AI
enableToc: true
---

# Avaliador de currículo

Use o Claude para avaliar currículos e fornecer feedback sobre pontos fortes, áreas de melhoria e adequação para uma vaga específica. O Claude pode analisar o conteúdo do currículo, identificar habilidades relevantes e sugerir melhorias para torná-lo mais atraente para recrutadores.

## Passos

Para esse exemplo, vou utilizar o meu currículo que foi gerado usando renderCV e está hospedado no meu [repositório do Github](https://github.com/devpolatto/Curriculum). Você pode usar o seu próprio currículo ou criar um currículo de exemplo para testar.

1. Acesse o [chat do Claude](https://claude.ai/new) e inicie uma nova conversa.

2. Anexe o seu currículo clicando no ícone de clipe de papel ou arrastando e soltando o arquivo na área de chat. Você também pode fornecer um link para o currículo, que no meu caso está hospedado no Github.

3. Forneça o prompt abaixo:

  Aqui vou usar o prompt baseado no meu caso, fique à vontade para adaptá-lo ao seu contexto e objetivo:

  ```txt
  // PERFIL DO ANALISTA
  Você é especialista em LinkedIn e Engenheiro de Software focado em DevSecOps.

  // FONTES DO CURRÍCULO
  Acesse meu currículo em uma das opções abaixo:
  - PDF: https://github.com/devpolatto/Curriculum/blob/master/rendercv_output/Angelo_Polatto_CV.pdf
  - YAML: https://github.com/devpolatto/Curriculum/blob/master/Angelo_Polatto_CV.yaml

  // DADOS DO LINKEDIN
  Perfil: https://www.linkedin.com/in/angelo-polatto-04121998/
  Objetivo: Emprego

  ---

  ## PARTE 1 — ANÁLISE

  Avalie cada seção abaixo com:
  - Nota de 0 a 10
  - Diagnóstico direto (máx. 2 linhas)
  - Se nota < 7, reescreva sugerindo melhorias, mantendo meu tom e contexto real

  Seções a avaliar:
  - Headline
  - About/Resumo
  - Experiências
  - Skills
  - Recomendações
  - URL
  - Atividade (frequência e qualidade de posts)

  Calibre tudo para o objetivo declarado. Calcule um score geral ao final.

  ---

  ## PARTE 2 — DASHBOARD HTML

  Gere um arquivo HTML completo com CSS e JS inline, usando as cores inspiradas no LinkedIn:
  - Azul principal: #0A66C2
  - Azul escuro: #004182
  - Fundo claro: #F3F2EF
  - Superfície branca: #FFFFFF
  - Texto principal: #1C1C1C
  - Texto secundário: #666666
  - Verde (notas 8+): #057642
  - Amarelo (notas 5–7): #E8A400
  - Vermelho (notas <5): #CC1016
  - Fontes: Inter + DM Mono (Google Fonts)

  ### Estrutura do Dashboard
  - Header: nome, objetivo, número de seguidores
  - Banner: score geral (nota em círculo) + veredito em uma frase
  - Grid de cards (um por seção):
    - Nome da seção + emoji
    - Nota colorida (verde/amarelo/vermelho)
    - Barra de progresso animada na cor correspondente
    - Diagnóstico curto
    - Box de reescrita sugerida (quando nota < 7, borda esquerda azul #0A66C2)
  - Radar chart canvas com as 8 dimensões
  - Footer: 3 prioridades mais urgentes em destaque

  Obs: O ano no header deve ser dinâmico (new Date().getFullYear()). Nunca use ano fixo no código.

  Retorne apenas o HTML completo, pronto para abrir no navegador.
  ```

O procedimento pode demorar, pois ele vai usar o canvas para gerar o gráfico radar, mas ao final você terá um dashboard completo com a análise do seu currículo e sugestões de melhorias para o seu perfil no LinkedIn.

Agora basta ir debatendo com o Claude sobre as sugestões, pedir para ele focar em pontos específicos ou até mesmo pedir para ele gerar um novo currículo baseado nas sugestões que ele deu. O importante é aproveitar a análise detalhada e as sugestões personalizadas para melhorar o seu perfil e aumentar suas chances de conseguir o emprego desejado.