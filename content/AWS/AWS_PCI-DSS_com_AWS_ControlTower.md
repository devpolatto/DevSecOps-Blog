---
title: AWS PCI-DSS com AWS Control Tower
tags: [AWS, PCI-DSS, AWS Control Tower]
enableToc: true
---

# Entendendo o AWS Control Tower visando PCI

O AWS Control Tower é um serviço que ajuda você a configurar e administrar um ambiente AWS seguro e com várias contas. Ele foi projetado para ajudar a implementar as práticas recomendadas e os padrões de conformidade da AWS de forma eficaz. Embora não tenha sido projetado explicitamente para a conformidade com o PCI DSS, o Control Tower estabelece as bases para a criação de uma infraestrutura segura da AWS alinhada com vários requisitos de segurança.
## Estrutura e isolamento de contas

Um dos aspectos essenciais da conformidade com o PCI DSS é o isolamento dos ambientes de dados do titular do cartão para minimizar o escopo das avaliações de conformidade. O AWS Control Tower facilita a criação de contas AWS bem organizadas, fornecendo contas separadas para diferentes ambientes, como desenvolvimento, teste e produção. Recomendamos usar esse recurso para criar um conjunto separado de contas para armazenar e processar os dados do titular do cartão. Esse isolamento de contas garante que os dados do titular do cartão sejam compartimentados, reduzindo o risco de acesso não autorizado e o escopo das auditorias.
## Implementação de guardrails e linhas de base de segurança

O AWS Control Tower permite que você implemente regras de configuração do AWS e políticas de controle de serviços (SCPs) como grades de proteção e linhas de base de segurança. Ao definir e aplicar padrões de segurança, você pode garantir que todas as contas do AWS em seu ambiente cumpram as medidas de segurança necessárias, alinhadas aos requisitos do PCI DSS.
## Automatização do provisionamento

Com o AWS Control Tower, você pode automatizar o provisionamento de contas da AWS usando o AWS Organizations e o AWS Service Catalog. Ao utilizar modelos e configurações predefinidos, você pode aplicar consistentemente as configurações de segurança em todas as contas, simplificando o processo de adesão aos requisitos do PCI DSS.
## Registro e monitoramento

Estar em conformidade com o PCI DSS significa mostrar que você tem práticas robustas de registro e monitoramento em vigor. O AWS Control Tower incentiva o uso do AWS CloudTrail para registro e do AWS CloudWatch para monitoramento de recursos e aplicativos da AWS. A configuração adequada desses serviços permite capturar e reter os registros de auditoria necessários para as avaliações de conformidade com o PCI DSS.

Você pode usar o Control Tower apenas para os logs do plano de controle da AWS (chamadas de API da nuvem), mas geralmente recomendo armazenar também os eventos críticos relevantes para a auditoria no CloudTrail. Como um serviço gerenciado, o CloudTrail oferece fortes garantias de armazenamento de logs durável e protegido contra violações.
## Controle de acesso e gerenciamento de identidade

O AWS Identity and Access Management (IAM) é um componente vital para garantir o acesso seguro aos recursos da AWS. O AWS Control Tower defende as práticas recomendadas de IAM, permitindo que você gerencie o acesso do usuário de forma eficaz. Ao seguir esses princípios de IAM, você pode controlar o acesso aos dados do titular do cartão, atendendo aos principais requisitos de controle de acesso do PCI DSS. O AWS IAM Identity Center (antigo AWS Single Sign-On) permite que você gerencie facilmente o acesso a várias contas da AWS de forma centralizada.
## Automação da verificação de conformidade

Embora o AWS Control Tower em si não realize verificações de conformidade com o PCI DSS, ele pode trabalhar em conjunto com outros serviços da AWS e ferramentas de terceiros para automatizar as verificações de conformidade e monitorar continuamente a postura de segurança em relação aos requisitos do PCI DSS.

# Estrutura de organização

A estrutura de conta/OU sugerida pelo AWS Control Tower pode ser adequada para a conformidade com o PCI-DSS (Payment Card Industry Data Security Standard), mas pode exigir alguma personalização, dependendo de seus requisitos específicos. O PCI-DSS tem requisitos rigorosos em relação à segurança de dados, o que inclui segmentação de rede, controle de acesso e monitoramento, todos os quais precisam ser integrados à estrutura da sua conta AWS.

Aqui estão algumas considerações importantes ao usar a estrutura sugerida do AWS Control Tower em um contexto de PCI-DSS:
## 1. Estrutura da conta e segmentação da UO

O AWS Control Tower normalmente recomenda o uso de uma estrutura de várias contas em que diferentes cargas de trabalho e ambientes (por exemplo, produção, desenvolvimento, teste) são isolados em contas separadas. Essa estrutura se alinha bem aos princípios de segmentação de rede do PCI-DSS, pois permite que você isole logicamente os ambientes que processam os dados do titular do cartão de outras cargas de trabalho que não o fazem.

Para a conformidade com o PCI-DSS, você provavelmente desejaria criar uma conta separada específica do PCI-DSS para cargas de trabalho que manipulam dados do titular do cartão e colocá-la em sua própria unidade organizacional (OU). Essa conta PCI deve ser rigorosamente controlada e monitorada.

Exemplo de estrutura:
- **Conta mestre/gerencial**: Usada para gerenciar sua organização AWS.
- **UO de segurança**: contém contas para operações de segurança (por exemplo, registo, ferramentas de segurança)
- **UO de serviços partilhados**: utilizada para recursos partilhados (por exemplo, Active Diretory, gestão de VPC).
- **UO PCI**: Dedicada a ambientes compatíveis com PCI, contendo:
	- **Conta de produção PCI**: Aloja ambientes de dados do titular do cartão (CDE).
	- **Conta de preparação/teste da PCI**: Utilizada para testar cargas de trabalho em conformidade com a PCI.
	- **Produção não PCI**: Cargas de trabalho não PCI que não lidam com dados do titular do cartão.
	Poderá ser necessário criar OUs ou contas adicionais, dependendo dos requisitos específicos do PCI-DSS e da estratégia de conformidade.

## 2. Personalização da Landing Zone

O AWS Control Tower automatiza a configuração de proteções, políticas e linhas de base (por meio do AWS Service Control Policies, ou SCPs) que ajudam na governança e na conformidade. No entanto, esses controles prontos para uso não são específicos do PCI-DSS. Você precisará personalizar e implementar controles específicos do PCI-DSS, tais como:
- **Criptografia de dados em trânsito e em repouso (requisito 3 do PCI-DSS)**: Certifique-se de que todo o armazenamento de dados na conta da PCI seja criptografado usando o AWS KMS ou chaves gerenciadas pelo cliente.
- **Registro e monitoramento (requisito 10 do PCI-DSS)**: Use os registros do AWS CloudTrail, do AWS Config e do AWS CloudWatch para monitorar todas as atividades no ambiente da PCI. Os registros devem ser imutáveis e centralizados em uma conta de segurança para análise.
- **Controle de acesso (requisito 7 do PCI-DSS)**: Imponha o acesso com privilégios mínimos usando o AWS Identity and Access Management (IAM) e configure a autenticação multifator (MFA) para acesso privilegiado aos ambientes da PCI.

Você também pode ativar serviços adicionais da AWS para fins de conformidade, como o AWS Audit Manager, que pode ajudá-lo a automatizar a coleta de evidências e a geração de relatórios para o PCI-DSS.

## 3. Guardrails e políticas

O AWS Control Tower aplica proteções preventivas e detectivas. No entanto, algumas dessas proteções padrão podem precisar ser ampliadas para atender aos requisitos do PCI-DSS. Por exemplo:
- **Restringir o acesso à rede** de e para o ambiente da PCI usando grupos de segurança VPC e NACLs, possivelmente utilizando o AWS Transit Gateway para segmentação de rede.
- **Implemente SCPs** que limitem determinados usos ou configurações de serviços da AWS que violariam a conformidade com a PCI, como não permitir a implantação de buckets S3 não criptografados.
SCPs personalizados podem ser necessários para um controle mais granular sobre as contas da PCI. Por exemplo, você pode restringir alterações nas configurações de segurança, proibir o uso de serviços inseguros e impor a criptografia.

## 4. Ferramentas de automação e segurança

Aproveite o AWS Security Hub, o AWS Config Rules e o AWS Systems Manager para monitorar e aplicar continuamente a conformidade com a PCI em suas contas.

O AWS Security Hub integra-se ao AWS Config para fornecer monitoramento contínuo e gerenciamento da postura de segurança. Para o PCI-DSS, você pode ativar o pacote de padrões de segurança PCI-DSS v3.2.1 para avaliar automaticamente seu ambiente em relação a esses requisitos.

O AWS Config pode ser personalizado com regras específicas do PCI para garantir a conformidade contínua.
## 5.  Gerenciamento de auditorias e documentação

A conformidade com o PCI-DSS envolve auditorias e relatórios regulares. Use o AWS Artifact para fazer download de relatórios de conformidade com o PCI-DSS para os serviços da AWS e incorporá-los aos seus próprios materiais de auditoria. A automação do AWS Control Tower, incluindo o registro centralizado e o rastreamento de configuração, simplifica a preparação da auditoria.

## 6. Ferramentas de terceiros

Dependendo da complexidade da sua carga de trabalho do PCI-DSS, talvez você precise de ferramentas de terceiros para aumentar os recursos de monitoramento de segurança, auditoria ou criptografia. Essas ferramentas podem ser integradas à estrutura de sua conta do AWS e, ao mesmo tempo, manter a conformidade.

---

Embora a estrutura de contas do AWS Control Tower seja uma base sólida para um ambiente em conformidade com o PCI-DSS, ela exigirá uma adaptação significativa para atender às necessidades específicas do PCI-DSS. Você precisará criar contas PCI dedicadas, implementar proteções e SCPs adicionais, configurar o monitoramento adequado e garantir uma segmentação rigorosa entre cargas de trabalho PCI e não PCI.

O AWS Control Tower pode simplificar o gerenciamento dessas contas, fornecendo governança e controle centralizados, mas os aspectos de conformidade com a PCI-DSS ainda exigirão configuração cuidadosa e gerenciamento contínuo.

---
# Conclusão

Em conclusão, alcançar a conformidade com o PCI DSS é uma responsabilidade compartilhada entre a AWS e sua organização. O AWS Control Tower é uma ferramenta poderosa para criar um ambiente AWS seguro e com várias contas, alinhado com as práticas recomendadas, fornecendo assim uma base sólida para atender aos requisitos do PCI DSS.

Ao aproveitar o AWS Control Tower juntamente com outros serviços da AWS e seguir as orientações da AWS sobre a conformidade com o PCI DSS, você pode aprimorar sua postura de segurança e lidar com confiança com as transações de cartão de pagamento na nuvem. Lembre-se de que manter a conformidade é um processo contínuo, e avaliações e melhorias regulares são cruciais para proteger os dados do titular do cartão de forma eficaz.

### Links
https://opstree.com/blog/2024/01/02/architecting-success-best-practices-for-implementing-aws-control-tower/

