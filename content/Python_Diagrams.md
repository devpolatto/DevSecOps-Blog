---
title: Python Diagrams
tags:
  - Python
  - Diagrams
  - Azure
enableToc: true
---
A biblioteca Python [diagrams](https://diagrams.mingrammer.com/) é uma ferramenta projetada para criar diagramas de arquitetura de sistemas em nuvem, que pode ser particularmente útil para visualizar e documentar projetos de infraestrutura como código (IaC), como aqueles gerenciados pelo Terraform.

Ela foi criada para prototipar uma nova arquitetura de sistema sem nenhuma ferramenta de design. Você também pode descrever ou visualizar a arquitetura do sistema existente.

O Diagram as Code permite rastrear as alterações no diagrama da arquitetura em qualquer sistema de controle de versão.

O objetivo deste estudo é encontrar uma maneira simples e personalizável de extrair dados de um projeto Terraform e convertê-los em um diagrama. Isso não impede que ele seja usado em outros casos, como microsserviços, topologia de rede, etc. Como é feito por meio de código, o limite é desconhecido.

Segue um resumo de suas funcionalidades e como ele pode ser usado em conjunto com o Terraform:

# Principais recursos da biblioteca de diagramas

**Nós predefinidos**: fornece um rico conjunto de nós predefinidos que representam vários serviços em nuvem, plataformas locais e arquiteturas de software. Esses nós abrangem serviços populares de provedores como AWS, Azure, GCP, Kubernetes e muito mais.

**Sintaxe simples**: a biblioteca usa uma sintaxe simples e intuitiva que permite aos usuários criar diagramas complexos com o mínimo de código. Ela aproveita o gerenciador de contexto e a biblioteca padrão do Python para tornar o processo mais simples.

**Personalização**: os diagramas podem ser amplamente personalizados, incluindo o layout, a aparência e as conexões entre os nós.

**Formatos de saída**: suporta vários formatos de saída, como PNG, PDF e SVG, tornando-o versátil para diferentes necessidades de documentação.

# Modelo de exemplo
Utilizei o projeto OpenVPN como base para gerar o design.

**Importante**: o método que escolhi para este estudo não é definitivo. Acredito que possa haver uma maneira de tornar o escopo ainda mais dinâmico, mas há casos e casos.

## Fonte de dados

Com o projeto terraform provisionado, precisamos de uma maneira simples de consumir o estado terraform do projeto. Como o estado terraform usa um backend, neste caso uma conta do Azure Storage, não podemos depender dessa fonte de dados.

Foi então que percebi que a saída poderia retornar o JSON do projeto. Além disso, eu poderia definir o modelo JSON com base no arquivo output.tf. Por exemplo:

```json
output "default" {
  value = {
    "azurerm_resource_group" = {
      "name"     = azurerm_resource_group.this.name
      "location" = azurerm_resource_group.this.location
    }
    "vnet" = {
      "name"          = azurerm_virtual_network.this.name
      "address_space" = azurerm_virtual_network.this.address_space
      "subnet" = {
        "name"             = azurerm_subnet.this.name
        "address_prefixes" = azurerm_subnet.this.address_prefixes
        "interface" = {
          "name"               = azurerm_network_interface.this.name
          "private_ip_address" = azurerm_network_interface.this.private_ip_address
          "public_ip_address"  = azurerm_public_ip.this.ip_address
          "azurerm_network_security_group" = {
            "name" = azurerm_network_security_group.this.name
          }
        }
      }
      "azurerm_public_ip_prefix" = azurerm_public_ip_prefix.this.ip_prefix
    }
    "virtual_machine" = {
      "name" = azurerm_linux_virtual_machine.ovpn_server.name
      "managed_disk" = {
        "name" = azurerm_managed_disk.this.name
      }
    }
    "storage_account" = {
      "name" : azurerm_storage_container.this.name
    }
  }
}
```

Dessa forma, consegui aninhar os recursos antes mesmo de começar a trabalhar no código Python.

Em seguida, crie um diretório na raiz do projeto chamado diagramas

Depois, execute o terraform output apontando a saída para um arquivo JSON neste diretório

```bash
terraform output -json > diagrams/terraform-output.json 
```
Extração de dados concluída.

## Script Python

Dentro do diretório diagrams, crie um arquivo chamado main.py

Vamos descrever o que o script está executando:

1. Criamos um método para ler o JSON gerado anteriormente

    ```python
    def load_terraform_output(file_path):
      with open(file_path) as json_file:
          return json.load(json_file)
    ```

    Em seguida, precisamos extrair os objetos desse arquivo e expô-los no código.

    ```python
    def extract_values(tf_data):
    value = tf_data.get("default", {}).get("value", {})
    resource_group = value.get('azurerm_resource_group', {})
    virtual_machine = value.get('virtual_machine', {})
    vnet = value.get('vnet', {})
    subnet = vnet.get('subnet', {})
    storage_account = value.get('storage_account', {})
    return resource_group, virtual_machine, vnet, subnet, storage_account
    ```

2. Em seguida (opcional), definimos algumas configurações de estilização padrão para o Diagrama.
    Os diagramas foram desenvolvidos usando a biblioteca [Graphviz](https://www.graphviz.org/). Portanto, toda a documentação sobre estilização pode ser encontrada lá.

    ```python
    def get_graph_attributes():
    graph_attr = {
        "fontsize": "20",
        # "margin": "10",
        # "overlap_scaling": "-10",
        "lheight": "0.8",
        "size": "10.24,8.00!"
    }
    node_attr = {
        "margin": "10",
        "height": "20",
        "layout": "fdp",
        "sep": "10"
    }
    return graph_attr, node_attr
    ```

3. Finalmente... o core

    Criamos um método para acoplar todos os scripts que irão gerar o diagrama.
    Em seguida, usamos o gerenciador de contexto ``with`` para criar o escopo do diagrama.

    ```python
    def create_diagram(resource_group, virtual_machine, vnet, subnet, storage_account):
    graph_attr, node_attr = get_graph_attributes()

    with Diagram("OpenVPN Server", show=False, filename="diagram", direction="TB", graph_attr=graph_attr, node_attr=node_attr):
    ```

    Com o contexto do diagrama definido, podemos inserir qualquer nó, cluster ou aresta.

    O objetivo agora é obter cada recurso que extraímos do arquivo JSON e associá-lo a um objeto do diagrama. Veja um exemplo:

    ```python
    elastic = Elasticsearch("logs")
    grafana = Grafana("OpenVpn Dashboard")
    elastic - Edge(color="blue", style="dashed") - grafana
    vm_icon = VM(f"{virtual_machine.get('name', 'VM')}\n{vm_private_ip} - {vm_public_ip}", attrs=str(node_attr))
    disk_icon = Disks(virtual_machine.get('managed_disk', {}).get('name', 'Managed Disk'))
    ```

    Com os recursos já configurados, podemos passar para as ligações entre os recursos, por exemplo:

    ```python
    vm_icon - Edge(label=".ovpn file", color="brown", style="dashed") >> storage_account_icon
    vm_icon - Edge(label="OpenVPN Authentication", color="blue", style="dashed") >> active_directory
    vm_icon - Edge(label="User and metric logs", color="green", style="dashed") - elastic
    ```

Este é o Script completo

```python
#!/usr/bin/env python3

import json

from diagrams import Cluster, Diagram, Edge

from diagrams.azure.general import Resourcegroups
from diagrams.azure.network import VirtualNetworks, Subnets
from diagrams.azure.identity import Users, ActiveDirectory
from diagrams.azure.storage import StorageAccounts
from diagrams.azure.compute import VM, Disks

from diagrams.elastic.elasticsearch import Elasticsearch
from diagrams.onprem.monitoring import Grafana


def load_terraform_output(file_path):
    with open(file_path) as json_file:
        return json.load(json_file)


def extract_values(tf_data):
    value = tf_data.get("default", {}).get("value", {})
    resource_group = value.get('azurerm_resource_group', {})
    virtual_machine = value.get('virtual_machine', {})
    vnet = value.get('vnet', {})
    subnet = vnet.get('subnet', {})
    storage_account = value.get('storage_account', {})
    return resource_group, virtual_machine, vnet, subnet, storage_account


def get_graph_attributes():
    graph_attr = {
        "fontsize": "20",
        "lheight": "0.8",
        "size": "10.24,8.00!"
    }
    node_attr = {
        "margin": "10",
        "height": "20",
        "layout": "fdp",
        "sep": "10"
    }
    return graph_attr, node_attr

def create_diagram(resource_group, virtual_machine, vnet, subnet, storage_account):
    graph_attr, node_attr = get_graph_attributes()
    with Diagram("OpenVPN Server", show=False, filename="diagram", direction="TB", graph_attr=graph_attr, node_attr=node_attr):
        storage_account_icon = StorageAccounts(storage_account.get('name', 'Storage Account'))
        users = Users("Acqio Users")
        active_directory = ActiveDirectory("Azure AD")


        with Cluster("Logs", graph_attr=graph_attr):
            elastic = Elasticsearch("logs")
            grafana = Grafana("OpenVpn Dashboard")
            elastic - Edge(color="blue", style="dashed") - grafana


        with Cluster(f"RG: {resource_group.get('name', 'Resource Group')}", graph_attr=graph_attr):


          with Cluster(f"{vnet.get('name', 'VNet')}\n{vnet.get('address_space', 'Address Space')}", graph_attr=graph_attr):


            with Cluster(f"{subnet.get('name', 'Subnet')}\n{subnet.get('address_prefixes', 'Address Prefixes')}", graph_attr=graph_attr):

              vm_private_ip = subnet.get('interface', {}).get('private_ip_address', 'Private IP')
              vm_public_ip = subnet.get('interface', {}).get('public_ip_address', 'Public IP')
              vm_icon = VM(f"{virtual_machine.get('name', 'VM')}\n{vm_private_ip} - {vm_public_ip}", attrs=str(node_attr))
              disk_icon = Disks(virtual_machine.get('managed_disk', {}).get('name', 'Managed Disk'))


              vm_icon - Edge(label=".ovpn file", color="brown", style="dashed") >> storage_account_icon
              vm_icon - Edge(label="OpenVPN Authentication", color="blue", style="dashed") >> active_directory
              vm_icon - Edge(label="User and metric logs", color="green", style="dashed") - elastic
              users >> vm_icon
              vm_icon >> disk_icon

if __name__ == "__main__":
    tf_data = load_terraform_output('terraform-output.json')
    resource_group, virtual_machine, vnet, subnet, storage_account = extract_values(tf_data)
    create_diagram(resource_group, virtual_machine, vnet, subnet, storage_account)
```

Agora basta executar o script

```bash
# Bash
sudo chmod +x main.py
./main.py

# Python
python main.py
```
E esse foi o resultado:

![[assets/Python_Diagrams-1.png]]

Como mencionado acima, o método utilizado neste exemplo não é imutável. O fato de ser personalizável por meio de código oferece uma enorme liberdade de personalização e automação.