---
title: Privisionando recursos na AWS a partir de uma VM CI-CD do Azure
tags:
  - Azure
  - AWS
  - OpenID
  - IAM
enableToc: true
---
Para o cenário em que uma VM do Azure executará pipelines do Azure DevOps e provisionará recursos no AWS, é crucial estabelecer uma autenticação segura e contínua. Aqui estão os melhores métodos para o conseguir:

# OpenID Connect (OIDC)

O Azure DevOps pode ser integrado ao AWS por meio de um provedor de identidade OIDC, que permite que a VM do Azure assuma funções no AWS com segurança sem exigir chaves de acesso de longo prazo.

**Como funciona**:

- Configure um fornecedor de identidade OIDC no AWS para a sua organização Azure Active Diretory (AAD) ou Azure DevOps.
- Crie uma função IAM no AWS que confie no provedor OIDC e conceda as permissões necessárias.
- Use aws sts assume-role-with-web-identity dentro do pipeline do Azure DevOps para obter credenciais temporárias.

**Etapas**

Aqui está uma visão geral dos recursos a serem criados no Terraform:

a) **Provedor de Identidade OIDC**

```json
resource "aws_iam_openid_connect_provider" "oidc_provider" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["<THUMBPRINT_DO_OIDC>"] # Obtenha o thumbprint do emissor
  url             = "https://vstoken.azure.net/<ORGANIZATION>" # URL do emissor do Azure DevOps
}

```

b) **Função IAM**

```json
resource "aws_iam_role" "azure_devops_role" {
  name = "AzureDevOpsOIDCRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.oidc_provider.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "vstoken.azure.net/<ORGANIZATION>:sub" = "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>"
          }
        }
      }
    ]
  })
}

resource "aws_iam_policy" "devops_policy" {
  name        = "AzureDevOpsPolicy"
  description = "Policy for Azure DevOps integration"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket", "s3:GetObject"]
        Resource = ["arn:aws:s3:::<YOUR_BUCKET>/*"]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "attach_policy" {
  role       = aws_iam_role.azure_devops_role.name
  policy_arn = aws_iam_policy.devops_policy.arn
}

```

**c) Configuração do Pipeline**

No Azure DevOps, configure o pipeline para usar a função IAM com o seguinte comando:

```yml
steps:
- script: |
    export AWS_ROLE_ARN=arn:aws:iam::<AWS_ACCOUNT_ID>:role/AzureDevOpsOIDCRole
    export AWS_WEB_IDENTITY_TOKEN_FILE=/path/to/oidc/token
    aws sts assume-role-with-web-identity \
      --role-arn $AWS_ROLE_ARN \
      --role-session-name DevOpsSession \
      --web-identity-token file://$AWS_WEB_IDENTITY_TOKEN_FILE
  displayName: 'Assume AWS Role'

```

**Vantagens**

- Não é necessário gerir chaves de acesso AWS a longo prazo.
- Seguro e segue as práticas recomendadas, aproveitando tokens de curta duração.
- Mais fácil de auditar e revogar o acesso, se necessário.

**Quando usar**

Melhor quando os seus pipelines de DevOps do Azure precisam de aprovisionar recursos do AWS de forma intermitente


```bash
ROLE_ARN="arn:aws:iam::********:role/AzureDevOpsOIDCRole"
AUDIENCE="api://azure-awsoidcconnection"


access_token=$(curl "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=${AUDIENCE}" -H "Metadata:true" -s| jq -r '.access_token')


credentials=$(aws sts assume-role-with-web-identity --role-arn ${ROLE_ARN} --web-identity-token ${access_token} --role-session-name AWSAssumeRole | jq '.Credentials' | jq '.Version=1')

export AWS_ACCESS_KEY_ID=$(echo "$credentials" | jq -r '.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo "$credentials" | jq -r '.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo "$credentials" | jq -r '.SessionToken')

aws sts assume-role-with-web-identity --role-arn ${ROLE_ARN} --web-identity-token ${access_token} --role-session-name AWSAssumeRole

aws s3api create-bucket --bucket "polattovmtestcli" --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1

aws s3api delete-bucket --bucket "polattovmtestcli" --region us-east-1

```

# Contas de serviço (IRSA)

As VMs do Azure dão suporte à Identidade Gerenciada, que pode ser vinculada às funções do AWS IAM usando a integração entre nuvens.

**Como funciona**:

- Configure uma Identidade Gerenciada no Azure para sua VM.
- Estabeleça uma relação de confiança entre a identidade gerenciada do Azure e a função do AWS IAM.
- Use um serviço de federação como o Azure AD ou uma configuração de federação personalizada.
- Crie uma função de IAM no AWS com política de confiança para aceitar tokens de identidade do Azure AD.
- Configurar o pipeline para usar a identidade gerenciada para assumir a função do AWS IAM.

**Vantagens**:

- Gerenciamento de credenciais simplificado por meio do Azure Managed Identity.
- Credenciais de curta duração para segurança.
- Totalmente integrado aos ecossistemas do Azure e do AWS.

**Quando usar**:

Ideal para cenários em que a VM do Azure é uma entidade de longa duração e serve como um hub central para CI/CD.

# CLI do AWS com Azure Key Vault 

Usando a CLI do AWS com chaves de acesso armazenadas no Azure Key Vault (como uma solução híbrida ou de backup). Para situações em que o OIDC ou o Managed Identity não é viável, pode utilizar chaves de acesso de forma segura.

**Como funciona**:

- Armazene a chave de acesso e o segredo do AWS no Azure Key Vault.
- Use pipelines do Azure DevOps para recuperar essas credenciais em tempo de execução.
- Configure o AWS CLI ou SDK na VM do Azure com as credenciais recuperadas.

**Vantagens**:

- Fácil de implementar e funciona em várias ferramentas de CI/CD.
- Gerenciamento centralizado de segredos.

**Desvantagens**:

- As credenciais de longo prazo aumentam o risco de uso indevido.
- É necessário mais esforço para girar as chaves periodicamente.

**Quando usar**:

Como uma opção de recurso quando as funções de IAM com OIDC ou Identidade gerenciada não são viáveis.


# Usar uma função entre contas com tokens temporários

Se a VM do Azure tiver um IP fixo ou uma ligação de rede privada, pode ser utilizada a assunção de funções do AWS através de uma troca segura de tokens.

**Como funciona**:

- Use uma ID externa e uma relação de confiança entre o Azure DevOps e a função do AWS IAM.
- O pipeline solicita credenciais temporárias por meio do processo de assunção de função.

**Vantagens**:

- Forte controlo de acesso baseado em funções.
- Seguro contra uso indevido.

**Quando usar**:

Adequado para ambientes altamente restritos com rede fixa



# Recomendação

Para a maioria dos casos de utilização, as funções IAM do AWS com o fornecedor OIDC são a melhor opção devido à sua segurança, escalabilidade e facilidade de utilização. Se necessitar de uma integração mais estreita com o Azure, considere a utilização do Managed Identity com AWS IAM Roles.

Certifique-se de que a função do AWS IAM tem o princípio do menor privilégio, concedendo apenas as permissões necessárias para o provisionamento de recursos. Audite regularmente as políticas de acesso e monitorize os registos de pipeline para garantir a segurança e a conformidade.