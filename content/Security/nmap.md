---
title: Nmap (Network Mapper) 
tags:
  - Linux
  - Security
enableToc: true
---

O Nmap (Network Mapper) é uma ferramenta de código aberto amplamente utilizada para exploração de rede e auditoria de segurança. Ele é projetado para descobrir hosts e serviços em uma rede, criando um "mapa" da rede. O Nmap é conhecido por sua flexibilidade, velocidade e capacidade de fornecer informações detalhadas sobre os sistemas-alvo.

**Funcionalidades do Nmap**

- **Descoberta de hosts**: O Nmap pode identificar quais hosts estão ativos em uma rede, utilizando técnicas como ping sweep e varredura ARP.
- **Detecção de serviços**: Ele pode identificar quais serviços estão rodando em um host, incluindo o número da porta e o protocolo associado (TCP/UDP).
- **Detecção de sistema operacional**: O Nmap pode tentar determinar o sistema operacional em execução em um host, analisando as respostas de rede e características do sistema.
- **Varredura de vulnerabilidades**: O Nmap pode ser usado para identificar vulnerabilidades conhecidas em serviços e sistemas, utilizando scripts de detecção de vulnerabilidades (Nmap Scripting Engine - NSE).
- **Varredura de firewall**: Ele pode ajudar a identificar regras de firewall e filtragem de pacotes, analisando as respostas de rede.

# Fase de reconhecimento com Nmap

## Descoberta de hosts

O Nmap não usa apenas os scans de portas para decidir se o host está ativo. Ele primeiro tenta fazer **host discovery** usando métodos padrão que são:

     - ICMP Echo Request (ping)
     - TCP SYN para portas 80 e 443
     - TCP ACK para porta 80
     - (em alguns casos) ICMP timestamp

⚠️ importante: Alguns hosts podem estar configurados para não responder a pings ICMP. Normalmente, na Azure por exemplo, as VMs não respondem a pings ICMP por padrão. Nesse caso, podemos usar a varredura de portas para identificar hosts ativos.

Se nenhum desses probes receber resposta (SYN/ACK, RST, ICMP reply, etc.), o Nmap considera o host como down e não executa o scan de porta solicitado (-sS e -sU), ou seja, mesmo que a porta 22 esteja aberta no TCP, o Nmap nem chega a enviar os pacotes SYN para a porta 22 porque já decidiu que o host está "morto" na fase de descoberta.

**Override**: Vamos explorar alguma formar de pular ou melhorar o host discovery para garantir que o Nmap realize a varredura de portas em todos os hosts da rede, mesmo que eles não respondam aos probes padrão.

1. Usando a opção `-Pn` para desabilitar o host discovery.

     O `-Pn` instrui o Nmap a pular a fase de descoberta de host e assumir que todos os hosts estão ativos. Isso é útil quando sabemos que os hosts podem não responder a pings ou outros probes.

     ```bash
     sudo nmap -Pn -sS -p 22 10.248.24.0/24
     ```

     O Nmap vai tentar o SYN scan na porta 22 em todos os 256 IPs, mesmo sem respostas iniciais.

2. Usando `-PS*` e `-PA*` para enviar probes TCP SYN/ACK para portas específicas.

     Este método envia pacotes TCP SYN ou ACK para portas específicas, aumentando a chance de identificar hosts ativos que não respondem a ICMP. Basicamente, você pode definir uma ou mais portas para enviar os probes. Essa porta deve ser uma porta que provavelmente estará aberta nos hosts alvo. Se alguma dessas portas responder, o Nmap considerará o host como ativo e prosseguirá com scan.

     ```bash
     sudo nmap -PS22,80,443,20001,20003 -PA22,80,443,20001,20003 -sS 10.248.24.0/24
     ```

3. Usando `-PE`, `-PP`, `-PM` para enviar probes ICMP específicos.

     Esses parâmetros permitem enviar diferentes tipos de pacotes ICMP para descobrir hosts. O `-PE` envia um Echo Request, o `-PP` envia um Timestamp Request e o `-PM` envia um Address Mask Request. Isso pode ajudar a identificar hosts que respondem a diferentes tipos de ICMP.

     ```bash
     sudo nmap -PE -PP -PM -sS 10.248.24.0/24
     ```

Vamos salvar essa saída para análise posterior.

Uma flag muito útil para salvar a saída é `-oG` (Grepable), que gera uma saída fácil de ler e filtrar. Segue exemplo:

```bash
sudo nmap -PS22,80,443,20001,20003 -PA22,80,443,20001,20003 -sS -oG alive-hosts.gnmap
```

Segue um comando único:

```bash
# Descobre hosts → filtra só os ativos → salva em arquivo
sudo nmap -PS22,80,443,20001,20003 -PA22,80,443,20001,20003 -sS -p- 10.248.24.0/24 -oG - | awk '/Up/{print $2}' > alive-hosts.txt
```

## Fase de Enumeração / Scanning detalhado

Após identificar os hosts ativos e as portas abertas, o próximo passo é analisar as vulnerabilidades associadas a esses serviços. Podemos usar várias ferramentas para essa etapa, como Nmap com scripts NSE, Nikto para servidores web, e Metasploit para exploração de vulnerabilidades conhecidas.

O objetivo dessa fase é coletar o máximo de informações úteis sobre cada host ativo, sem tentar explorar vulnerabilidades ainda.

Queremos responder perguntas como:

- Quais portas estão abertas?
- Quais serviços estão rodando?
- Quais versões exatas desses serviços?
- Existe algum banner / mensagem de login que dê pistas?
- Qual sistema operacional (ou tipo aproximado)?
- Há algum indicativo de tecnologias web (CMS, frameworks, etc.)?

Essa fase é fundamental porque a qualidade da enumeração determina o sucesso das fases seguintes.

**O que fazer agora (passos recomendados):**

1. Scan de portas completo + detecção de versão (o mais importante agora)

     ```bash
     # Scan mais completo em todos os hosts da lista
     sudo nmap -Pn -sV -sC -p 22,443,3001,3002,20000,20001,20002,20003 -T4 -iL alive-hosts.txt -oA nmap-full-scan
     ```

     ```bash
     # output
     Nmap scan report for 10.248.24.180
     Host is up (0.072s latency).

     PORT      STATE    SERVICE        VERSION
     22/tcp    open     ssh            OpenSSH 9.7 (protocol 2.0)
     3001/tcp  open     nessus?
     3002/tcp  open     exlm-agent?
     20000/tcp filtered dnp
     20001/tcp filtered microsan
     20002/tcp filtered commtact-http
     20003/tcp filtered commtact-https

     Nmap scan report for 10.248.24.181
     Host is up (0.056s latency).

     PORT      STATE    SERVICE        VERSION
     22/tcp    open     ssh            OpenSSH 9.7 (protocol 2.0)
     3001/tcp  open     nessus?
     3002/tcp  open     exlm-agent?
     20000/tcp filtered dnp
     20001/tcp filtered microsan
     20002/tcp filtered commtact-http
     20003/tcp filtered commtact-https

     Nmap scan report for 10.248.24.182
     Host is up (0.10s latency).

     PORT      STATE    SERVICE        VERSION
     22/tcp    open     ssh            OpenSSH 9.7 (protocol 2.0)
     3001/tcp  open     nessus?
     3002/tcp  open     exlm-agent?
     20000/tcp filtered dnp
     20001/tcp filtered microsan
     20002/tcp filtered commtact-http
     20003/tcp filtered commtact-https

     Nmap scan report for 10.248.24.199
     Host is up (0.10s latency).

     PORT      STATE    SERVICE        VERSION
     22/tcp    filtered ssh
     3001/tcp  open     nessus?
     3002/tcp  open     exlm-agent?
     20000/tcp open     ssh            OpenSSH 9.7 (protocol 2.0)
     20001/tcp open     ssh            OpenSSH 9.7 (protocol 2.0)
     20002/tcp open     ssh            OpenSSH 9.7 (protocol 2.0)
     20003/tcp filtered commtact-https

     Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
     Nmap done: 4 IP addresses (4 hosts up) scanned in 42.35 seconds
     ```

     Opções úteis:
          `-sS`     → SYN scan (rápido e discreto)
          `-sV`     → detecção de versão do serviço
          `-sC`     → roda os scripts padrão do Nmap (muitos deles são de enumeração leve)
          `-p-`     → todas as 65.535 portas TCP
          `-T4`     → velocidade mais alta (equilíbrio entre velocidade e confiabilidade)
          `-iL`     → lê os hosts do arquivo
          `-oA`     → gera saída em 3 formatos (normal, XML e grepable)

     Detecção de Sistema Operacional:

     ```bash
     sudo nmap -Pn -O -T4 -iL alive-hosts.txt -oN os-detection.txt
     ```

     ```bash
     # output
     Nmap scan report for 10.248.24.180
     Host is up (0.086s latency).
     Not shown: 998 filtered tcp ports (no-response)
     PORT     STATE SERVICE
     22/tcp   open  ssh
     3001/tcp open  nessus
     Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
     Device type: general purpose
     Running (JUST GUESSING): Linux 4.X|5.X (85%)
     OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
     Aggressive OS guesses: Linux 4.15 - 5.8 (85%), Linux 5.0 (85%), Linux 5.0 - 5.4 (85%)
     No exact OS matches for host (test conditions non-ideal).

     Nmap scan report for 10.248.24.181
     Host is up (0.072s latency).
     Not shown: 998 filtered tcp ports (no-response)
     PORT     STATE SERVICE
     22/tcp   open  ssh
     3001/tcp open  nessus
     Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
     Device type: general purpose
     Running (JUST GUESSING): Linux 2.6.X|4.X|5.X (85%)
     OS CPE: cpe:/o:linux:linux_kernel:2.6.32 cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
     Aggressive OS guesses: Linux 2.6.32 (85%), Linux 4.15 - 5.8 (85%), Linux 5.0 - 5.4 (85%)
     No exact OS matches for host (test conditions non-ideal).

     Nmap scan report for 10.248.24.182
     Host is up (0.094s latency).
     Not shown: 998 filtered tcp ports (no-response)
     PORT     STATE SERVICE
     22/tcp   open  ssh
     3001/tcp open  nessus
     Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
     Device type: general purpose
     Running (JUST GUESSING): Linux 2.6.X|4.X|5.X (85%)
     OS CPE: cpe:/o:linux:linux_kernel:2.6.32 cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
     Aggressive OS guesses: Linux 2.6.32 (85%), Linux 4.15 - 5.8 (85%), Linux 5.0 - 5.4 (85%)
     No exact OS matches for host (test conditions non-ideal).

     Nmap scan report for 10.248.24.199
     Host is up (0.088s latency).
     Not shown: 998 filtered tcp ports (no-response)
     PORT      STATE SERVICE
     3001/tcp  open  nessus
     20000/tcp open  dnp
     Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
     OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
     No OS matches for host

     OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
     Nmap done: 4 IP addresses (4 hosts up) scanned in 23.84 seconds
     ```

     Vamos analizar o output acima:

     Principais pontos que o Nmap está nos dizendo:

     - **Apenas 2 portas abertas** (22 e 3001)
     - **998 portas filtradas** (firewall está bloqueando / não respondendo)
     - **Falta de portas fechadas visíveis** → isso é o principal motivo do aviso
          O Nmap precisa ver pelo menos uma porta aberta e uma porta fechada para fazer uma fingerprinting confiável do sistema operacional.
     - **Está apenas chutando** (“JUST GUESSING”, 85% de confiança)
     - **Suspeita de Linux** (kernel 2.6, 4.x ou 5.x), mas **sem nenhuma certeza**

     Resumo: O Nmap não conseguiu identificar o sistema operacional com confiabilidade por causa do firewall muito restritivo e da quantidade muito pequena de portas abertas.

     O mecanismo de fingerprinting de sistema operacional do Nmap depende fortemente de comparar o comportamento da pilha TCP/IP do alvo com assinaturas conhecidas. Para isso, ele precisa observar diferenças sutis em:

     - Como o sistema responde a pacotes TCP com flags incomuns (SYN+FIN, sem flags, etc.)
     - Ordem dos campos no cabeçalho TCP (window size, TTL inicial, opções TCP, DF bit, etc.)
     - Respostas a probes que testam retransmissão, fragmentação, etc.

     Mas o mais importante de tudo:

     O Nmap precisa de pelo menos 1 porta aberta e 1 porta fechada visíveis para fazer uma comparação confiável.

     Nesse caso:
     - Portas abertas visíveis: 22 (SSH) e 3001/3002 (Nginx reverse proxy)
     - Portas fechadas visíveis: nenhuma
     - Todas as outras 65.532 portas estão como filtered (filtradas pelo firewall), ou seja, o firewall não devolve RST (reset) quando a porta está fechada.

     Quando não há porta fechada respondendo com RST, o Nmap perde a maior parte das informações que ele usa para diferenciar sistemas operacionais. Ele acaba tendo que “chutar” com base em poucas respostas (geralmente só da porta 22) e do TTL / distância de rede.

     **Resultado**:

     Ele só consegue dizer “provavelmente algum Linux kernel 2.6/4.x/5.x” com baixa confiança (~85%), mas não consegue ir além disso.

     O que podemos fazer para tentar melhorar a precisão do OS detection?

     1. Forçar scan com OS detection agressivo:

          ```bash
          sudo nmap -Pn -O --osscan-guess -p 22,443,3001,3002 -T4 10.248.24.182 -oN os-all-ports.txt
          ```

     2. Usar probes mais agressivas / diferentes

          ```bash
          sudo nmap -Pn -O --osscan-limit --defeat-rst-ratelimit -p 22,443,3001,3002 10.248.24.182
          ```

     Não desanime, se mesmo assim o Nmap não conseguir identificar o sistema operacional com precisão, é provável que o firewall esteja bloqueando muitos dos probes necessários para uma fingerprinting confiável. Nesse caso, pode ser necessário tentar outras abordagens, como análise manual de banners, ou até mesmo tentar obter acesso ao sistema para coletar informações diretamente.

     A fase de enumeração é como uma investigação policial: quanto mais pistas você coleta, melhor preparado estará para as próximas etapas. Nem sempre você encontrará tudo o que precisa na primeira tentativa. Seja meticuloso e documente tudo cuidadosamente.

## Fase de Análise de Vulnerabilidades

Após a enumeração detalhada, o próximo passo é analisar as vulnerabilidades associadas aos serviços e versões identificados. Podemos usar várias ferramentas para essa etapa, como Nmap com scripts NSE, Nikto para servidores web, e Metasploit para exploração de vulnerabilidades conhecidas.

OK, vamos por parte. Atualmente, nós conhecemos as portas 22 (OpenSSH 9.7), podemos começar por aí.

OpenSSH 9.7 é uma versão muito recente (lançada em 2024), o que significa que vulnerabilidades críticas públicas são bastante raras. A maioria dos problemas reais que encontramos em SSH hoje em dia não vem de falhas no código do OpenSSH em si, mas sim de:

- Configurações inseguras
- Algoritmos criptográficos obsoletos ou fracos
- Políticas de autenticação ruins
- Uso de chaves fracas ou senhas permitidas

**O que fazer agora na porta 22?**

Aqui está uma sequência lógica e prática que a maioria dos pentesters segue quando encontra um SSH recente:

1. Coletar informações básicas do banner e negociação (5 segundos)

     ```bash
     nmap -sV --script=banner -p 22 10.248.24.182
     ```

     ```bash
     # output
     Nmap scan report for 10.248.24.182
     Host is up (0.037s latency).

     PORT   STATE SERVICE VERSION
     22/tcp open  ssh     OpenSSH 9.7 (protocol 2.0)
     |_banner: SSH-2.0-OpenSSH_9.7

     Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
     Nmap done: 1 IP address (1 host up) scanned in 0.53 seconds
     ```

     ou use o Netcat para coletar o banner:

     ```bash
     nc -v 10.248.24.182 22
     ```

     ```bash
     # output
     Connection to 10.248.24.182 22 port [tcp/ssh] succeeded!
     SSH-2.0-OpenSSH_9.7
     ```

     ou use o próprio SSH client:

     ```bash
     ssh -vvv -o PreferredAuthentications=no 10.248.24.182 -p 22
     ```

     ```bash
     # output

     # O OpenSSH 9.6p1 Ubuntu-3ubuntu13.14 é uma versão corrigida que resolve várias vulnerabilidades de segurança, incluindo CVE-2024-6387
     OpenSSH_9.6p1 Ubuntu-3ubuntu13.14, OpenSSL 3.0.13 30 Jan 2024
     ...
     Remote protocol version 2.0, remote software version OpenSSH_9.7
     ...
     # Todos os algoritmos escolhidos são modernos e considerados seguros em 2025/2026. Não há algoritmos obsoletos ou fracos sendo oferecidos na negociação real.
     debug1: kex: algorithm: ecdh-sha2-nistp256
     debug1: kex: host key algorithm: ssh-ed25519
     debug1: kex: server->client cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
     debug1: kex: client->server cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
     ...

     # Métodos de autenticação permitidos
     debug1: Authentications that can continue: publickey
     debug1: No more authentication methods to try.
     ...

     # Banner / Mensagem pré-login
     Authorized uses only. All activity may be monitored and reported.
     ...

     # Host key: Usa Ed25519 (chave muito segura e moderna). Fingerprint SHA256 é o padrão atual (não SHA1)
     Server host key: ssh-ed25519 SHA256:f5u0kvdKYXxo/Qdn0OjiIGb+MHMN4z50SF+Ng58cG7M
     ...
     ```
     Anote as informações coletadas:
          - Versão exata do OpenSSH (9.7)
          - Algoritmos de criptografia suportados
          - Métodos de autenticação permitidos
          - Mensagem de banner pré-login
          - Fingerprint da chave do host

     Existe várias formar de se explorar o SSH, como por exemplo:

     - Verificar se root login está permitido (teste passivo)
          `ssh -v -l root 10.248.24.182`
     - Scan Nmap com scripts SSH (opcional, mas recomendado)
          `sudo nmap --script "ssh2-enum-algos,ssh-auth-methods,ssh-hostkey" -p 22 10.248.24.182`
     - Listar todos os algoritmos suportados (opcional)

          ```bash
          ssh -Q kex 10.248.24.182
          ssh -Q cipher 10.248.24.182
          ssh -Q mac 10.248.24.182
          ```

     Com as informações coletadas, podemos pesquisar vulnerabilidades conhecidas associadas ao OpenSSH 9.6 e aos algoritmos suportados.

     Lembre-se de que, mesmo que não existam vulnerabilidades críticas conhecidas, ainda é importante avaliar a configuração do SSH para garantir que esteja seguindo as melhores práticas de segurança.

     Como investigar fica a sua criatividade, mas aqui estão algumas dicas:

     - Consulte bancos de dados de vulnerabilidades como CVE Details, NVD, Exploit-DB
     - Verifique listas de discussão e fóruns de segurança
     - Use ferramentas de análise de vulnerabilidades como OpenVAS, Nessus, etc.