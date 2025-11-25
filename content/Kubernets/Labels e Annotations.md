---
title: Labels e Annotations
tags:
  - Kubernets
enableToc: true
---
![[kubeIcon.png]]

Entender a diferença entre anotações e Labels no Kubernetes é crucial, pois eles servem a propósitos distintos na organização e gerenciamento do seu cluster. Aqui está um detalhamento:

# Labels
## Objetivo

Os rótulos são usados para organizar e selecionar recursos do Kubernetes. Eles ajudam a categorizar os recursos em grupos lógicos que podem ser consultados.

## Principais caraterísticas:

- Projetado para ser selecionável por meio de consultas.
- Comumente usado pelo Kubernetes para identificar recursos para agrupamento, filtragem ou aplicação de ações específicas.
- Anexado como pares de valores chave aos recursos.
## Exemplos de utilização:

- Identificar a aplicação de um recurso

```yaml 
labels:
  app.kubernetes.io/name: homepage
```

- Agrupamento de recursos por ambiente (por exemplo, preparação, produção):

```yaml title:"example.yaml"
labels:
  environment: production
```

- Seleção de recursos com selectores de etiquetas (por exemplo, em Implementações, Serviços ou Ingresses)

```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: homepage
```
## Casos de uso:

- Atribuição de recursos a grupos específicos (por exemplo, frontend vs. backend).
- Direcionamento de recursos para atualizações, implementações ou monitoramento.

# Annotations
## Objetivo: 

As anotações são utilizadas para anexar metadados aos recursos Kubernetes. Ao contrário das etiquetas, as anotações não se destinam a seleção ou agrupamento.

## Caraterísticas principais:

- Armazenar metadados não identificadores que aplicativos ou componentes do Kubernetes podem usar.
- Não podem ser consultadas ou selecionadas como as etiquetas.
- Projetado para armazenar informações descritivas ou de configuração.
- Pode armazenar valores complexos, como cadeias de caracteres JSON.

## Exemplos de utilização:

- Fornecer uma descrição ou contexto sobre um recurso:

```yaml
annotations:
  description: "This service is for the homepage."
```

- Armazenamento de metadados para automatização ou integrações:

```yaml
annotations:
  kubernetes.io/service-account.name: homepage
```

- Especificação da configuração de ferramentas externas:

```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

## Casos de uso:

- Adição de metadados para depuração, auditoria ou automação.
- Configuração de comportamento personalizado para ferramentas ou controladores externos.
- Armazenamento de histórico ou informações de status.


### **Comparison Table**

| Feature         | **Labels**                               | **Annotations**                                    |
| --------------- | ---------------------------------------- | -------------------------------------------------- |
| **Objetivo**    | Organizar e identificar recursos.        | Anexar metadados adicionais aos recursos.          |
| **Consultável** | Sim, através de selectores de etiquetas. | Não, não pode ser consultado.                      |
| **utilização**  | Agrupamento, seleção e filtragem         | Adição de dados descritivos ou não identificáveis. |
| **Tipo**        | Pares simples de valores-chave.          | Texto de forma livre ou dados do tipo JSON.        |
| **Exemplos**    | `environment: production`                | `description: "Used by CI/CD pipelines"`           |

---

# Analogia prática

Imagine que está a organizar uma biblioteca:

- **Labels**: São como etiquetas autocolantes nos livros que dizem _“Ficção”_, _“Não-ficção”_, _“Ficção Científica”_, etc. Estas etiquetas ajudam-no a agrupar e a encontrar rapidamente livros com base no seu tipo.
- Anotações**: São como notas dentro da capa do livro, fornecendo mais detalhes como _“Doado por Alice em 2024-01-01”_ ou _“Para ser devolvido até 2024-12-31”_. Acrescentam informação extra mas não são utilizadas para ordenar ou organizar.

# Dicas para iniciantes:

- Use rótulos quando precisar agrupar ou selecionar recursos.
	- Por exemplo, os Serviços utilizam selectores para encaminhar o tráfego para Pods com etiquetas correspondentes.
- Use anotações para armazenar metadados extras ou configurar integrações.
	- Por exemplo, as anotações são usadas para especificar como o cert-manager ou os controladores de entrada devem tratar um recurso.
- Mantenha os rótulos concisos e consistentes.
	- Use uma convenção de nomenclatura clara, como app.kubernetes.io/name ou environment.
- Não sobrecarregue os rótulos.
	- Se um valor não for necessário para a seleção de recursos, armazene-o em anotações.

Com esses conceitos em mente, você terá uma imagem mais clara de como usar rótulos e anotações de forma eficaz no Kubernetes! 😊