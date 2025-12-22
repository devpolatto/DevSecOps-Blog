---
title: A variável de ambiente PATH
tags:
  - Linux
  - PATH
enableToc: true
---

A variável PATH informa ao shell onde procurar arquivos executáveis quando você executa um comando. Ela contém uma lista de diretórios separados por dois pontos, por exemplo, `/usr/local/bin:/usr/bin:/bin`.

Quando você executa um comando (por exemplo, python), o shell procura esses diretórios em **ordem** e executa o primeiro executável correspondente que encontrar.

# Precedência de diretório
Na maioria das distribuições Linux, `/usr/local/bin` é listado antes de `/usr/bin` no PATH. Por exemplo:

```shell
echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin
```

Isso significa que se um executável (por exemplo, python) existir em `/usr/local/bin` e `/usr/bin`, o que estiver em `/usr/local/bin` será executado porque o shell verifica primeiro `/usr/local/bin`.

## Por que /usr/local/bin tem precedência

- **Instalações do sistema versus instalações do usuário**:
	- `/usr/bin` (e /bin) normalmente contém binários instalados pelo gerenciador de pacotes do sistema (por exemplo, apt, yum, dnf) como parte do software padrão do sistema operacional.
	- O `/usr/local/bin` é reservado para o software instalado manualmente pelo usuário ou administrador, geralmente a partir do código-fonte, de scripts de terceiros ou de ferramentas como pip, npm ou cargo.
- **Filosofia de design**:
	- A precedência de `/usr/local/bin` permite que os usuários substituam o software fornecido pelo sistema por versões personalizadas ou mais recentes sem modificar os diretórios do sistema, que geralmente são protegidos ou gerenciados pelo gerenciador de pacotes.
	- Isso segue o padrão de hierarquia do sistema de arquivos (FHS), que designa `/usr/local` para software instalado localmente que não faz parte da distribuição do sistema operacional.