---
title: Deployment Escale down
tags:
  - Kubernets
  - Escale
enableToc: true
---
Reduzir uma implantação (deployment) para zero réplicas é uma forma limpa e reversível de parar todos os pods sem excluir a configuração da implantação. Isso é útil para suspender temporariamente serviços, economizar recursos ou realizar manutenções sem perder o histórico ou as definições do deployment.

---

## ✅ **Como Escalar uma Implantação para Zero Réplicas**

Use o comando abaixo para reduzir o número de réplicas de um deployment para zero:

```shell
kubectl scale deployment <nome-do-deployment> --replicas=0 -n <namespace>
```

**Parâmetros:**
- `<nome-do-deployment>`: Nome do deployment que você deseja pausar.
- `<namespace>`: Namespace onde o deployment está localizado (se não estiver no `default`).

**Exemplo real:**

```shell
kubectl scale deployment my-api-client --replicas=0 -n production
```

Esse comando instrui o Kubernetes a finalizar todos os pods gerenciados por esse deployment, sem apagar nenhuma configuração ou metadado. O deployment permanece disponível para ser reativado posteriormente.

---

## 🔄 **Como Escalar Novamente (Reativar)**

Para reativar o serviço, basta definir o número de réplicas desejado:

```shell
kubectl scale deployment my-api-client --replicas=1 -n production
```

Você pode ajustar o valor de `--replicas` conforme a necessidade do seu ambiente.

---

## 📝 **Observações Importantes**

- Essa abordagem é recomendada para suspensões temporárias, testes, manutenções ou economia de recursos.
- Não exclui o deployment, facilitando a reversão rápida.
- O deployment pode ser escalado para qualquer número de réplicas posteriormente.
- Certifique-se de que não há dependências críticas antes de pausar o serviço.

---

**Referência oficial:** [Documentação do Kubernetes - kubectl scale](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#scale)