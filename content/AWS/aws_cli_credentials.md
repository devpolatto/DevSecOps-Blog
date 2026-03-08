---
title: Configurando ambiente para usar o AWS CLI com credenciais
tags:
  - AWS
enableToc: true
---

# Estrutura de credentials e config do AWS CLI:

- Naming convention: empresa-ambiente-permissao

- `.aws/credentials:`

     Somente credenciais reais ficam aqui.

     ```shell
     [default]
     aws_access_key_id = ****
     aws_secret_access_key = ****
     [account-root]
     aws_access_key_id = ****
     aws_secret_access_key = ****
     ```

- `.aws/config:`

     Aqui ficam região, roles e organização dos perfis.

     ```shell
     [default]
     region = us-west-2
     output = json

     # Base profile
     [profile account-root]
     region = us-west-2

     [profile account-prod-admin]
     role_arn = arn:aws:iam::<AWS_ACCOUNT_ID>:role/AdministratorRole
     source_profile = account-root
     region = us-west-2

     [profile account-dev-admin]
     role_arn = arn:aws:iam::<AWS_ACCOUNT_ID>:role/AdministratorRole
     source_profile = account-root
     region = us-west-2
     ```

Valide como comando `aws sts get-caller-identity` para garantir que as credenciais estão funcionando corretamente.

```shell
aws sts get-caller-identity --profile account-prod-admin
```