---
title: Redirecionando o Claude code para o Ollama
tags:
  - Claude
  - Ollama
  - LLM
enableToc: true
---

# Redirecionando o Claude Code para Ollama executado remotamente

Este documento descreve o processo de redirecionamento do código do Claude para o Ollama, permitindo que o Ollama seja executado remotamente. O objetivo é garantir que as chamadas ao Claude sejam redirecionadas para o Ollama, mantendo a funcionalidade e a eficiência.

**Pre-requisitos:**
- Hosted Ollama instance configurada e acessível remotamente.
- Acesso ao código do Claude para realizar as modificações necessárias.

## Passos para redirecionar o Claude code para o Ollama:

1. Preparação do Servidor LLM – headless

     Faça tudo via SSH a partir do seu host de trabalho.

     1.1 Instale o Ollama no servidor remoto seguindo as [instruções da documentação oficial](https://ollama.com/download). Se já tiver o Ollama instalado, certifique-se de que ele esteja atualizado com uma versão >= 0.14.0.

     1.2 Confirme que o serviço escuta na rede.

     Configure o Ollama para escutar em um endereço IP acessível, como `0.0.0.0` ou o IP específico do servidor.

     Crie um arquivo de configuração para o Ollama em /etc/ollama/env.conf com o seguinte conteúdo:

     ```conf
     OLLAMA_HOST=0.0.0.0:11434 # escuta em todas as interfaces na porta 11434
     OLLAMA_ORIGINS=* # se quiser permitir CORS de qualquer origem (útil pra web UI remota)
     OLLAMA_KEEP_ALIVE=-1 # ← mantém TODOS os modelos carregados forever
     ```

     Salve e reinicie o serviço do Ollama para aplicar as mudanças:

     ```shell
     sudo systemctl daemon-reload
     sudo systemctl restart ollama
     ```

     Se este método não funcionar, tente isso:

     Edite o service do Ollama para garantir que ele escute na rede. Abra o arquivo de serviço do Ollama:

     ```bash
     sudo systemctl edit ollama.service
     ```

     Isso vai permitir que você adicione uma configuração personalizada. Adicione as seguintes linhas para configurar o ambiente:

     ```shell
     [Service]
     Environment="OLLAMA_HOST=0.0.0.0"
     Environment="OLLAMA_KEEP_ALIVE=-1"
     ```

     Depois de salvar, recarregue o daemon e reinicie o serviço:

     ```shell
     sudo systemctl daemon-reload
     sudo systemctl restart ollama 
     ```

     1.3 (Opcional) Caso necessário, configure o firewall para permitir conexões na porta 11434:

     ```shell
     sudo ufw allow from <subnet>/24 to any port 11434 proto tcp
     sudo ufw reload
     ```

     1.4 (Opcional) Configure um proxy reverso (como Nginx) para expor o Ollama de forma segura, especialmente se for acessá-lo pela internet.

     1.5 (Opcional) Crie modelo default carregado no boot do Ollama para garantir que ele esteja sempre pronto para responder às solicitações:

     Escolha um modelo bom para coding (recomendação 2026):

     - `qwen2.5-coder:14b` ou `qwen2.5-coder:7b` (melhor custo-benefício)
     - `deepseek-coder-v2:16b` ou `deepseek-coder-v2:236b` (se tiver bastante VRAM)

     Crie o script de auto-load:

     ```shell
     cat <<EOF > /usr/local/bin/ollama-autoload.sh
     #!/usr/bin/env bash
     sleep 15
     ollama pull qwen3-coder:30b # ← troque pelo modelo que quiser
     ollama run qwen3-coder:30b --keepalive 10m
     EOF
     ```

     Dê permissão de execução ao script:

     ```shell
     chmod +x /usr/local/bin/ollama-autoload.sh
     ```

     crie um serviço systemd para o script de auto-load:

     ```shell
     cat <<EOF | sudo tee /etc/systemd/system/ollama-load-model.service
     [Unit]
     Description=Carrega modelo default no Ollama
     After=ollama.service
     Requires=ollama.service

     [Service]
     Type=oneshot
     User=polatto
     ExecStart=/usr/local/bin/ollama-autoload.sh
     RemainAfterExit=true

     [Install]
     WantedBy=multi-user.target
     EOF
     ```

     Ative o serviço para iniciar no boot:

     ```shell
     sudo systemctl daemon-reload
     sudo systemctl enable --now ollama-load-model.service
     ```

     Agora, verifique se o modelo está carregado:

     ```shell
     ollama ps
     systemctl status ollama-load-model.service
     ```

2. Configurando o Claude para usar o Ollama

     2.1 No código do Claude, localize as partes onde ele faz chamadas para o modelo de linguagem. Isso geralmente envolve chamadas HTTP para um endpoint local ou uso de uma biblioteca específica.

     2.2 Modifique as chamadas para apontar para o endpoint do Ollama. Por exemplo, se o Claude estiver fazendo chamadas para `http://localhost:11434`, altere para `http://<IP_DO_SERVIDOR>:11434`.

     Crie um arquivo para expor variáveis de ambiente para o Claude, por exemplo, `.claude-ollama.env`:

     Insira as seguintes variáveis, ajustando conforme necessário:

     ```env
     export ANTHROPIC_AUTH_TOKEN=ollama
     export ANTHROPIC_API_KEY=""
     export ANTHROPIC_BASE_URL=http://<IP_DO_SERVIDOR>:11434
     ```

     Carregue sempre que for usar:

     ```shell
     source .claude-ollama.env
     ```

     execute com o claude:

     ```shell
     claude --model qwen3-coder:30b
     ```

Referências:
- [Ollama Documentation](https://docs.ollama.com/integrations/claude-code)