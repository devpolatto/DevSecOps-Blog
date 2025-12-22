---
title: Particionamento e Rotulagem de Disco de 1TB no Linux
tags:
  - Linux
  - Disk
enableToc: true
---

## Guia Completo: Particionamento e Rotulagem de Disco de 1TB no Linux

Este guia detalha o processo para particionar, formatar, rotular, montar e criar links simbólicos para um disco de 1TB, utilizando `/dev/sda` como exemplo. Siga cada etapa cuidadosamente para garantir a correta configuração do armazenamento.

---

### 1. Identifique o Disco
Confirme qual disco será particionado. Use o comando abaixo para listar os discos disponíveis:

```bash
lsblk
```
Procure por `/dev/sda` na lista. Certifique-se de que está selecionando o disco correto para evitar perda de dados.

---

### 2. (Opcional) Apague Partições Existentes
Se o disco já possui partições (ex: `/dev/sda1`), apague-as com o `fdisk`:

```bash
sudo fdisk /dev/sda
```

No menu do `fdisk`:
- Pressione `d` para deletar a partição existente.
- Repita se houver mais de uma partição.
- Pressione `w` para salvar as alterações e sair.

---

### 3. Crie Duas Novas Partições
Abra novamente o `fdisk` para criar as novas partições:

```bash
sudo fdisk /dev/sda
```

No menu interativo:
1. Pressione `n` para criar uma nova partição.
	- Escolha `primary` se solicitado.
	- Para a primeira partição (`sda1`), defina o tamanho como `+500G`.
	- Se aparecer mensagem sobre assinatura (signature), escolha `Y` para remover.
2. Repita o processo para criar a segunda partição (`sda2`) usando o espaço restante.
3. Pressione `w` para gravar a tabela de partições e sair.

Exemplo de sessão:
```bash
Command (m for help): n
Partition number (1-128, default 1): 1
First sector (default):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (default): +500G

Command (m for help): n
Partition number (2-128, default 2): 2
First sector (default):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (default):

Command (m for help): w
```

---

### 4. Formate as Partições e Defina os Rótulos
Formate cada partição com o sistema de arquivos desejado (exemplo: `ext4`) e atribua rótulos:

```bash
sudo mkfs.ext4 -L data-apps /dev/sda1
sudo mkfs.ext4 -L backup /dev/sda2
```

---

### 5. Verifique as Partições e Rótulos
Confirme se as partições e rótulos foram criados corretamente:

```bash
lsblk -o NAME,SIZE,LABEL
```

Saída esperada:
```bash
sda         931.5G
├─sda1        500G data-apps
└─sda2      431.5G backup
```

---

### 6. Monte as Partições
Crie os pontos de montagem e monte as partições:

```bash
sudo mkdir -p /mnt/data-apps /mnt/backup
sudo mount /dev/sda1 /mnt/data-apps
sudo mount /dev/sda2 /mnt/backup
```

---

### 7. (Opcional) Adicione ao `/etc/fstab`
Para montar automaticamente as partições na inicialização, adicione as seguintes linhas ao arquivo `/etc/fstab`:

```bash
echo "LABEL=data-apps  /mnt/data-apps  ext4  defaults  0  2" | sudo tee -a /etc/fstab
echo "LABEL=backup     /mnt/backup     ext4  defaults  0  2" | sudo tee -a /etc/fstab
```

---

### 8. Crie Links Simbólicos (Opcional)
Se desejar acessar os diretórios por caminhos alternativos, crie links simbólicos:

```bash
sudo ln -s /mnt/data-apps /var/lib/data-apps
sudo ln -s /mnt/backup /var/lib/backup
```

Verifique os links:
```bash
ls -l /var/lib/data-apps /var/lib/backup
```
Saída esperada:
```bash
lrwxrwxrwx 1 root root 15 Nov 24 12:00 /var/lib/data-apps -> /mnt/data-apps
lrwxrwxrwx 1 root root 13 Nov 24 12:00 /var/lib/backup -> /mnt/backup
```

---

### 9. Defina Permissões de Acesso
Altere o proprietário dos diretórios para o usuário desejado (exemplo: `polatto`):

```bash
sudo chown -R polatto:polatto /mnt/data-apps /mnt/backup
sudo chown -R polatto:polatto /var/lib/data-apps /var/lib/backup
```

---

## Resumo Visual da Estrutura Final

```bash
sda         931.5G
├─sda1        500G data-apps
└─sda2      431.5G backup
```

---

**Dicas de Segurança:**
- Sempre faça backup de dados importantes antes de particionar discos.
- Certifique-se de que está operando no disco correto para evitar perda de dados.