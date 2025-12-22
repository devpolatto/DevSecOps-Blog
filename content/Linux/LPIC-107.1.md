---
title: 107.1 - Administrar Contas de usuários, grupos e arquivos de sistema relacionados
tags:
  - LPIC1
  - Linux
enableToc: true
---
# Usuários

## Criar

Para criar um usuário no Linux por meio da CLI, você pode usar o comando useradd ou adduser para criar um usuário. Veja a seguir as etapas:

Use `useradd` (nível inferior, mais manual) ou `adduser` (interativo, fácil de usar).
#### useradd

```shell
sudo useradd -m -s /bin/bash username
```

- `-m:` Cria um diretório inicial para o usuário (`/home/username`).
- `-s /bin/bash`: Define o shell padrão (substitua /bin/bash pelo shell de sua preferência, por exemplo, /bin/zsh).
- Caso o usuário não necessite de um shell interativo, use a opção -s /sbin/nologin. Esse é um shell especial que impede o usuário de acessar uma sessão de shell interativa. Se o usuário tentar fazer login (por exemplo, por meio de SSH ou de um terminal), ele será negado com uma mensagem do tipo “Esta conta não está disponível no momento”.
#### adduser

```shell
sudo adduser username
```

- Esse comando solicita que você defina uma senha e preencha os detalhes do usuário (por exemplo, nome completo, telefone).
- Ele cria automaticamente um diretório inicial e define um shell padrão.

## Deletar

### Excluir totalmente uma conta de usuário

```shell
userdel username
```

- Isso remove a conta do usuário e seu grupo principal (**se não houver outros usuários nele**).
- Para excluir também o diretório pessoal do usuário e o spool de e-mail, adicione o sinalizador `-r`

**Observações**:
- Certifique-se de que você tenha privilégios sudo para executar esses comandos.
- Verifique a associação ao grupo antes/depois com getent group groupname ou members groupname (conforme mostrado na resposta anterior).
- Seja cauteloso com `userdel -r,` pois ele exclui **permanentemente** os dados do usuário.
- Se o usuário estiver conectado, talvez seja necessário encerrar a sessão primeiro (pkill -u username).
# Grupos

## Criar

Para criar um novo grupo:

```
sudo groupadd mygroup
```

Para adicionar um usuário à um grupo no Linux por meio da CLI, você pode usar o comando  usermod ou gpasswd para adicionar o usuário a um grupo. Veja a seguir as etapas:

### Adicionar usuário a um grupo

Você pode adicionar o usuário a um grupo durante a criação ou depois.

- Adicionar durante a criação do usuário

```shell
# Durante a criação
sudo useradd -m -s /bin/bash -G groupname username

# ou

# Isso adiciona o usuário ao grupo durante ou após a criação.
sudo adduser username groupname
```

- Adicionar a um grupo existente (após a criação do usuário)

```shell
sudo usermod -a -G groupname username

# ou

sudo gpasswd -a username groupname
```

- `-a`: Anexa o grupo aos grupos existentes do usuário (sem -a, ele substitui os grupos).
- `-G`: Especifica o(s) grupo(s) suplementar(es).
## Listar

### Listar usuários do grupo

Para listar os usuários em um grupo no Linux por meio da linha de comando, use um destes comandos:

#### Getent

```shell
getent group groupname

# output

sudo:x:27:username
```

Substitua groupname pelo grupo que deseja verificar. A saída mostra o nome do grupo, a senha, o GID e uma lista de usuários separada por vírgulas.

#### Groups

Para mostrar os grupos aos quais um usuários pertence, use o seguinte comando:

```shell
groups username

# output

username : username adm cdrom sudo dip plugdev users lpadmin docker devops
```

Uso do `grep` com `/etc/group` (pesquisa diretamente no arquivo de grupo):

```
grep '^groupname' /etc/group
```

Para excluir um usuário de um grupo no Linux, você pode usar o comando `gpasswd` ou `deluser`. Veja como:

## Deletar

### Remover um usuário de um grupo específico

#### gpasswd

```
gpasswd -d username group
```

Isso remove o usuário do grupo especificado sem afetar sua conta.

#### deluser

```shell
deluser username groupname
```
