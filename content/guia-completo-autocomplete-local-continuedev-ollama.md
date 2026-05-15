# Guia Completo: Autocomplete e Sugestões com Continue.dev e Ollama

> Crie seu próprio sistema de autocomplete inteligente rodando 100% localmente, sem dependências de APIs externas.

## Introdução

Autocomplete é um superpoder no desenvolvimento moderno. Mas será que você precisa realmente enviar seu código para a nuvem a cada keystroke? A combinação de **Continue.dev** (IDE companion com suporte a modelos locais) e **Ollama** (executor local de LLMs) oferece uma alternativa potente: autocomplete rápido, privado e sob seu controle total.

Neste guia, você aprenderá a:

- Configurar Continue.dev + Ollama do zero
- Escolher e otimizar modelos LLM para autocomplete
- Tunar parâmetros para equilibrar qualidade vs latência
- Otimizar o uso de GPU para máxima performance
- Monitorar e resolver problemas de performance
- Expandir além do autocomplete com o ecossistema Continue.dev

Este guia assume uma máquina Linux com GPU NVIDIA e experiência básica com desenvolvimento. Os conceitos, porém, aplicam-se a qualquer setup.

---

## Parte 1: Arquitetura e Componentes

### O que é Continue.dev?

[**Continue.dev**](https://docs.continue.dev/ide-extensions/install) é um plugin/extension para IDEs (VS Code, JetBrains) que funciona como um "AI coding companion". Diferentemente de ferramentas cloud-first como GitHub Copilot, Continue.dev é **agnóstico em relação ao modelo**: você escolhe qual LLM usar — seja local (Ollama), remoto (Claude, OpenAI), ou aberto (Mistral, Llama, etc).

Principais capacidades:

- **Autocomplete**: sugestões de código inline enquanto você digita
- **Chat**: conversa contextual sobre o código
- **Edit**: refactoring assistido
- **Slash commands**: ações específicas (test, docs, etc)

### O que é Ollama?

**Ollama** é um runtime para executar Large Language Models (LLMs) localmente, sem necessidade de APIs externas. Ele:

- Gerencia download, cache e execução de modelos
- Oferece uma API HTTP simples para integração
- Otimiza automaticamente para CPU/GPU da sua máquina
- Roda modelos em formato GGUF (quantizado, eficiente)

### Fluxo de Dados

```bash
Você digita código na IDE
    ↓
Continue.dev detecta contexto (função atual, imports, etc)
    ↓
Envia para Ollama via HTTP (localhost:11434)
    ↓
Ollama carrega modelo em GPU/CPU
    ↓
Modelo gera sugestão (50-256 tokens)
    ↓
Continue.dev renderiza inline na IDE
```

A latência típica: **50-300ms** dependendo do modelo e hardware.

---

## Parte 2: Sua Stack de Hardware

Para aproveitar este guia ao máximo, conhecer seu hardware é essencial:

```bash
Processador:  AMD Ryzen™ 7 5700G (16 threads, integrado Radeon)
Memória RAM:  32.0 GiB (consumo base ~10GB)
GPU:          NVIDIA GeForce RTX™ 4060 Ti (8GB VRAM)
```

**Por que isso importa?**

- **RTX 4060 Ti (8GB)**: Consegue rodar modelos 7-13B quantizados com folga (4.8-6.5GB). Maiores (34B+) necessitam offloading para CPU.
- **16 threads**: Suficiente para paralelizar cache de embeddings e fallback para CPU.
- **32GB RAM**: Memória total para modelo em VRAM + sistema + browser aberto.

Se seu hardware é diferente, **ajuste as recomendações proporcionalmente** (veja a seção de troubleshooting).

---

## Parte 3: Instalação e Configuração Inicial

### Passo 1: Instalar Ollama

```bash
# No Linux, download direto do site
curl -fsSL https://ollama.ai/install.sh | sh

# Ou via package manager
sudo apt install ollama  # Debian/Ubuntu
```

Verifique:

```bash
ollama --version
ollama serve  # Inicia o daemon
```

O Ollama roda por padrão em `http://localhost:11434`. Deixe rodando em background (systemd, screen, etc).

### Passo 2: Baixar Modelos

Para **autocomplete**, você quer um modelo rápido, não muito grande. Recomendações:

```bash
# Modelo principal (recomendado)
ollama pull qwen2.5-coder:7b

# Alternativas
ollama pull codellama:7b          # Treino específico para código
ollama pull deepseek-coder:6.7b   # Muito rápido, bom custo-benefício
ollama pull mistral:7b            # Genérico mas versátil

# Modelo de embeddings (para contexto inteligente)
ollama pull nomic-embed-text:latest
```

Verifique os modelos baixados:

```bash
ollama list
```

### Passo 3: Instalar Continue.dev

**Para VS Code:**

```bash
# Via marketplace: procure por "Continue"
# Ou instale via CLI:
code --install-extension Continue.Dev.continue
```

**Para JetBrains (IntelliJ, PyCharm, etc):**

- Abra Settings → Plugins → Marketplace
- Procure "Continue"
- Instale

Após instalar, uma aba "Continue" aparecerá na IDE.

### Passo 4: Configurar Continue.dev

O arquivo de configuração fica em:

- **Linux/Mac**: `~/.continue/config.yaml`
- **Windows**: `%USERPROFILE%\.continue\config.json`

Crie ou edite `config.yaml`:

```yml
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: qwen-coder-fast
    provider: ollama
    model: qwen-coder-fast:latest
    roles:
      - autocomplete
    autocompleteOptions:
      debounceDelay: 350
      maxPromptTokens: 1024
      temperature: 0.2
      maxTokens: 256
      topP: 0.9
  - name: Nomic Embed
    provider: ollama
    model: nomic-embed-text:latest
    roles:
      - embed
```

Salve e reinicie a IDE. Continue.dev deve reconhecer o Ollama.

---

## Parte 4: Escolhendo o Modelo Certo

Não existe "melhor modelo". A escolha depende do trade-off entre:

| Critério | Impacto | Prioridade |
|----------|--------|-----------|
| **Latência** | Quanto mais rápido, melhor UX no autocomplete | 🔴 Alta |
| **Qualidade** | Precisão e relevância das sugestões | 🔴 Alta |
| **VRAM** | Cabe na sua GPU? | 🟠 Média |
| **Contexto** | Quantos tokens de código anterior analisa | 🟠 Média |
| **Custo computacional** | CPU + GPU load enquanto digita | 🟡 Baixa |

### Recomendação para RTX 4060 Ti (8GB)

```yaml
Modelo Recomendado:
  Nome: Qwen2.5-Coder 7B
  VRAM: ~4.8 GB
  Latência: 80-120ms (RTX 4060 Ti)
  Qualidade: Excelente (treino específico para código)
  Contexto: até 4096 tokens

Por quê?
  ✅ Cabe confortavelmente na VRAM
  ✅ Treino específico para Python, JavaScript, Go, Rust, etc
  ✅ Velocidade aceitável para autocomplete
  ✅ Bom suporte a Continue.dev

Alternativas:
  • CodeLlama 7B: melhor para C/C++/CUDA
  • DeepSeek Coder 6.7B: mais rápido, qualidade similar
  • Mistral 7B: genérico, não otimizado para código
```

---

## Parte 5: Tuning de Parâmetros

O arquivo `config.yaml` do Continue.dev oferece controles finos:

```yaml
name: Local Config
version: 1.0.0
schema: v1

models:
  - name: Qwen2.5-Coder 7B
    provider: ollama
    model: qwen2.5-coder:7b
    roles:
      - autocomplete
    autocompleteOptions:
      debounceDelay: 350          # ms a aguardar após você parar de digitar
      maxPromptTokens: 4096       # máx contexto a enviar ao modelo
      onlyMyCode: true            # Inclui apenas o código contido no repositório para contextualização.
      useCache: true              # Ativa cache de respostas
    defaultCompletionOptions:
      temperature: 0.2            # criatividade (0=determinístico, 1=aleatório)
      maxTokens: 512              # tamanho máx da sugestão
      topP: 0.9                   # núcleo de amostragem (controla diversidade)

  - name: Qwen2.5-Coder (Chat)
    provider: ollama
    model: qwen2.5-coder:7b
    roles:
      - chat
    completionOptions:
      temperature: 0.5            # mais criativo para chat
      maxTokens: 512
      topP: 0.95
```

### Explicação de Cada Parâmetro

**`debounceDelay: 250`**

- Aguarda 250ms após você parar de digitar antes de chamar o modelo
- Evita sobrecarga com requisições a cada keystroke
- Balanceamento: 250ms é imperceptível, <100ms pode sobrecarregar

**`maxPromptTokens: 2048`**

- Quantidade máxima de contexto enviado (seu código anterior)
- 2048 tokens ≈ 4-8 KB de código
- ✅ Bom para funções contextualizadas
- ⚠️ Acima de 4096 começa a ficar lento

**`temperature: 0.3` (autocomplete)**

- Valores baixos (0.1-0.3): sugestões previsíveis, conservadoras
- Valores altos (0.7-1.0): sugestões criativas, podem ser estranhas
- Para autocomplete, **baixo é melhor** (você quer código que faz sentido)
- Para chat, use 0.5-0.7 (mais conversacional)

**`maxTokens: 256`**

- Tamanho máximo da sugestão
- 256 tokens ≈ 40-60 linhas de código
- Recomendação: 128-256 para autocomplete

**`topP: 0.9` (nucleus sampling)**

- Controla diversidade das sugestões
- 0.9 significa "considere os tokens que somam 90% de probabilidade"
- Valores altos = mais diversidade, mais risco de erros
- Valores baixos = mais concentrado, mais seguro
- Para autocomplete: 0.85-0.95 é bom

### Exemplo de Tuning para Diferentes Cenários

- **Scenario A: Máxima Qualidade (latência não importa)**

```yaml
debounceDelay: 500
maxPromptTokens: 4096
temperature: 0.2
maxTokens: 256
topP: 0.85
```

- **Scenario B: Máxima Velocidade (qualidade OK)**

```yaml
debounceDelay: 100
maxPromptTokens: 512
temperature: 0.1
maxTokens: 128
topP: 0.8
```

- **Scenario C: Balanceado (recomendado)**

```yaml
debounceDelay: 250        # Padrão
maxPromptTokens: 2048
temperature: 0.3
maxTokens: 256
topP: 0.9
```

---

## Parte 6: Otimizações de GPU

A GPU é o gargalo crítico. Maximizar seu uso reduz latência de 500ms para 80ms.

### Monitorar GPU

```bash
# Terminal 1: Observe em tempo real
watch -n 0.1 nvidia-smi

# Terminal 2: Enquanto você usa a IDE
# Você verá: CUDA, GPU %, Memory %, Temperature
```

Esperado durante autocomplete:

```bash
GPU   100% (saturado)
Mem   ~5.8 GB / 8GB
Temp  60-75°C
Power ~120W
```

### Criar Modelfile Customizado

Ollama permite customizar parâmetros de execução. Crie um `Modelfile`:

```dockerfile
# Modelfile (sem extensão)
FROM qwen2.5-coder:7b

# Contexto: balance entre memória e qualidade
PARAMETER num_ctx 2048

# GPU: força máximo na GPU
PARAMETER num_gpu 99

# CPU: use ~12-14 threads (seu Ryzen tem 16)
PARAMETER num_thread 12

# Cache: otimiza hits de KV cache
PARAMETER num_batch 128
PARAMETER num_ubatch 64

# LLaMA-specific: numa optimization (se suportado)
PARAMETER rope_frequency_base 10000
PARAMETER rope_frequency_scale 1
```

Build:

```bash
ollama create qwen-coder-optimized -f ~/.ollama/Modelfile
```

Use em `config.yaml`:

```yaml
models:
  - name: qwen-coder-optimized
    provider: ollama
    model: qwen-coder-optimized
    roles:
      - autocomplete
```

### Verificar Alocação de GPU

Depois de iniciar autocomplete, rode:

```bash
ollama ps
```

Output esperado:

```bash
NAME                       ID              SIZE      PROCESSOR    CONTEXT
qwen-coder-optimized       736ac009cbe0    4.8 GB    100% GPU     2048
```

Se mostrar `100% CPU`, seu modelo não está usando GPU. Diagnóstico:

```bash
# 1. Verifique se CUDA está instalado
nvidia-smi

# 2. Verifique logs do Ollama
journalctl -u ollama -n 50

# 3. Force GPU no Modelfile
# Tente: PARAMETER num_gpu 32 (alguns drivers precisam de valores específicos)

# 4. Último recurso: recrie o container
ollama rm qwen-coder-optimized
ollama pull qwen2.5-coder:7b
```

---

## Parte 7: Embeddings para Contexto Inteligente

Até agora, falamos de "contexto = código antes da linha". Mas **embeddings** trazem contexto semanticamente relevante de todo o projeto.

### Como Funciona

```bash
Quando você digita em arquivo_x.py:
  ↓
Ollama extrai embedding (vetor 768D) da linha atual
  ↓
Procura por embeddings similares em todo o projeto
  ↓
Envia código similar + o código anterior ao modelo
  ↓
Modelo gera sugestão mais contextualizada
```

### Configurar Embeddings

No `config.yaml`:

```yml
models:
  - name: Nomic Embed
    provider: ollama
    model: nomic-embed-text:latest
    roles:
      - embed
```

Baixe o modelo (pequeno, ~300MB):

```bash
ollama pull nomic-embed-text:latest
```

**Vantagens:**

- Sugestões mais relevantes (entende semântica, não apenas sintaxe)
- Código similar de outras partes do projeto é descoberto automaticamente
- Sem overhead de latência (embeddings são rápidos)

**Desvantagem:**

- Usa RAM/VRAM adicional (~500MB)

---

## Parte 8: Troubleshooting e Monitoramento

### Problema: Autocomplete Lento (>500ms)

**Diagnóstico:**

```bash
# 1. Verifique GPU
nvidia-smi
# Se Memory < 4GB em uso: modelo não está na GPU

# 2. Verifique CPU
top -b -n 1 | head -n 20
# Se CPU 100%, GPU 0%: rodar em CPU (muito lento)

# 3. Verifique logs
tail -f ~/.ollama/logs/server.log
# ou
journalctl -u ollama -f

# 4. Teste Ollama diretamente
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2.5-coder:7b","prompt":"def hello():\n","stream":false}' \
  | jq '.response'
```

**Soluções Comuns:**

| Sintoma | Causa | Solução |
|--------|-------|---------|
| Latência 500-800ms | Modelo em CPU | Aumentar VRAM ou usar modelo menor |
| Latência 200-300ms | Esperado em primeira requisição | Cache warming, normal |
| Latência oscila (100-500ms) | Modelo em VRAM + swap | Reduzir maxPromptTokens |
| Crashes da IDE | OOM (out of memory) | Reduzir maxPromptTokens ou context size |

### Problema: Ollama Não Detectado

```bash
# 1. Verifique se Ollama está rodando
curl -s http://localhost:11434/api/tags | jq .

# 2. Se falhar, inicie Ollama
ollama serve

# 3. Se Ollama crasha, verifique logs
journalctl -u ollama -n 100

# 4. Possível conflito de porta
# Mude em ~/.ollama/server.conf
# export OLLAMA_HOST=127.0.0.1:11434
```

### Monitoramento Contínuo

Script para monitorar performance:

```bash
#!/bin/bash
# monitor_autocomplete.sh

while true; do
  echo "=== $(date) ==="
  
  # GPU
  nvidia-smi --query-gpu=index,name,memory.used,memory.total,utilization.gpu,temperature.gpu \
    --format=csv,noheader
  
  # Ollama
  ollama ps
  
  # Latência (testa modelo)
  time curl -s -X POST http://localhost:11434/api/generate \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen2.5-coder:7b","prompt":"def ","stream":false}' > /dev/null
  
  sleep 5
done
```

---

## Parte 9: Checklist de Instalação

Use este checklist para garantir tudo funciona:

- [ ] Ollama instalado e rodando (`ollama serve`)
- [ ] Modelos baixados (`ollama pull qwen2.5-coder:7b` + `nomic-embed-text`)
- [ ] Continue.dev instalado na IDE
- [ ] `~/.continue/config.yaml` configurado
- [ ] Ollama conectado (`curl http://localhost:11434/api/tags`)
- [ ] First autocomplete testado (digitar função no editor)
- [ ] GPU confirmada em uso (`nvidia-smi` mostra GPU %)
- [ ] Latência aceitável (<300ms)
- [ ] Embeddings habilitados e funcionando
- [ ] Chat testado (seleção de código + prompt)

---

## Parte 10: Próximos Passos

### Curto Prazo (Semana 1)

1. **Customize debounceDelay** para seu gosto (tente 100, 250, 500)
2. **Experimente maxPromptTokens** (512, 1024, 2048) — qual latência você prefere?
3. **Teste modelos alternativos** (CodeLlama, DeepSeek) se não gostar de Qwen

### Médio Prazo (Mês 1)

1. **Integre com seu workflow** — slash commands customizados para suas tarefas
2. **Compare qualidade** — guarde exemplos de sugestões boas/ruins, ajuste parâmetros
3. **Monitore recursos** — acompanhe uso de RAM/GPU/CPU para otimizar

### Longo Prazo (Contínuo)

1. **Explore RAG/embeddings avançados** para projetos grandes
2. **Considere fine-tuning** de modelos se tem tarefas específicas (linguagens raras, padrões de código únicos)
3. **Integre com CI/CD** — use Continue.dev em scripts de fix de código

---

## Conclusão

Você agora tem um sistema de **autocomplete inteligente, privado e rápido** rodando 100% localmente. Não é Copilot, mas oferece:

✅ **Privacidade**: seu código nunca sai da máquina  
✅ **Controle**: você escolhe modelo, parâmetros, tudo  
✅ **Velocidade**: 80-120ms é mais rápido que muitos serviços cloud (Variado)
✅ **Custo**: zero (depois da GPU amortizada)  
✅ **Extensibilidade**: customize conforme precisa  

**Próximo passo recomendado:** Instale tudo, teste durante uma semana, e ajuste `debounceDelay` + `temperature` até sentir natural. Não existe configuração "perfeita" — existe configuração que funciona para *seu* estilo de programação.

---

## Referências

- [Continue.dev Docs](https://docs.continue.dev) — documentação oficial
- [Ollama Docs](https://ollama.ai) — guias e modelos disponíveis
- [Qwen2.5-Coder](https://huggingface.co/Qwen/Qwen2.5-Coder) — modelo recomendado
- [Nomic Embed](https://www.nomic.ai/blog/nomic-embed-text-v1) — embeddings eficientes

Boa sorte com seu autocomplete local! 🚀
