---
title: Alocação de Endereçamento IP
tags:
  - IP_address
  - Network
  - AWS
  - Azure
  - VNET
  - VPC
enableToc: true
---

# Alocação de Endereçamento IP

Este documento estabelece o padrão para alocação, organização e gestão de endereços IP utilizados nas redes hospedadas na plataforma Microsoft Azure.
O objetivo é evitar sobreposição de endereços, melhorar a escalabilidade e padronizar a criação de VNets e Subnets em todas as subscriptions.

# Objetivos
- Organizar a distribuição de ranges IP entre ambientes e subscriptions.
- Prevenir conflitos ou sobreposições durante a criação de VNets, peerings e VPNs.
- Definir uma estrutura hierárquica clara e escalável para alocação de IP.
- Facilitar governança, automação, auditoria e troubleshooting.

# Ranges Reservados por Ambiente

Cada /12 fornece aproximadamente 1 milhão de IPs, garantindo escalabilidade.

## Aure

| Ambiente       | Range IP Reservado      | Descrição                           |
|----------------|-------------------------|-------------------------------------|
| Global         | 10.32.0.0/12            | Endereços para workloads de produção|
| Produção       | 10.48.0.0/12            | Endereços para workloads de produção|
| Homologação    | 10.64.0.0/12            | Endereços para workloads de homologação|
| Desenvolvimento| 10.80.0.0/12            | Endereços para workloads de desenvolvimento|
| Produção GTW   | 10.96.0.0/12            | Endereços para workloads de produção GTW|

## AWS

| Ambiente       | Range IP Reservado      | Descrição                           |
|----------------|-------------------------|-------------------------------------|
| Global         | 10.112.0.0/12            | Endereços para workloads de produção|
| Produção       | 10.128.0.0/12            | Endereços para workloads de produção|
| Homologação    | 10.144.0.0/12            | Endereços para workloads de homologação|
| Desenvolvimento| 10.160.0.0/12            | Endereços para workloads de desenvolvimento|

Se novos ambientes/contas forem criados, novos ranges /12 devem ser alocados seguindo o mesmo padrão.

# Padrão de Segmentação por VNet/VPC e Subnet

Cada VNet/VPC deve ser segmentada em /24 ou /25 subnets, dependendo da necessidade de IPs.

Recomendado: Quando for criar um VNet/VPC, pense no escopo de um /24, mas aloque um /25 para permitir crescimento futuro. Por que isso? Porque um /24 oferece 256 endereços IP, enquanto um /25 oferece 128 endereços IP, permitindo que você tenha subnets adicionais no futuro sem precisar reconfigurar a VNet/VPC inteira.

Ao criar uma nova Vnet na Azure, você pode ver uma lista de Vnets disponíveis. Por exemplo:

**Primeio /24 10.48.0.0/24**
Vnet A (Já alocada): 10.48.0.0/25
Vnet A-reserva (Não alocada): 10.48.0.128/25

**Segundo /24 10.48.1.0/24**
Vnet B (Já alocada): 10.48.1.0/25
Vnet B-reserva (Já alocada): 10.48.1.128/25

**Terceiro /24 10.48.2.0/24**
Vnet C (Nova Vnet a ser criada): 10.48.2.0/25
Vnet C-reserva (Nova Vnet a disposição): 10.48.2.128/25

Percebe-se que a Vnet A está dentro do range 10.48.0.0/24, mas segmentadas em /25 para permitir crescimento futuro. Por que isso é uma boa prática? Porque você pode criar uma nova Vnet (Vnet C) sem precisar reconfigurar a Vnet A, já que ambas estão em, range range /24 distintos.

A criação da Vnet B não receberia o range 10.48.0.128/25 (A-reserva) porque este range já está reservado (ainda não registrado na VNET) para a Vnet A. Assim, a Vnet B recebe o próximo range disponível dentro do /24, que é 10.48.1.0/25, e assim por diante.

Isso nos permite reduzir o range total utilizado, mantendo a flexibilidade para futuras expansões.

# Regras de Ouro

- Nunca criar VNets com prefixos fora do range de seu ambiente.
- Nunca duplicar ou sobrepor ranges entre subscriptions.
- Sempre validar overlap antes de criar VNets/Subnets.
- Subnets devem estar documentadas e seguir o padrão definido.
