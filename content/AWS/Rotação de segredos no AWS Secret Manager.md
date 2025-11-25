---
title: Rotação de segredos no AWS Secret Manager
tags:
  - AWS
  - SSM
enableToc: true
---
O AWS Secrets Manager requer uma função Lambda personalizada para lidar com a rotação de segredos se você quiser implementar uma estratégia de rotação **personalizada**. Abaixo está uma explicação expandida e um exemplo de função Lambda que pode ser implantado para gerenciar a rotação de segredos.
# Criando o segredo
Crie um segredo do AWS Secrets Manager

```json
# 
resource "aws_secretsmanager_secret" "example_secret" {
  name        = "example-secret"
  description = "Example secret managed by Terraform"

  rotation_lambda_arn = aws_lambda_function.secret_rotation_function.arn
  rotation_rules {
    automatically_after_days = 7
  }
}
```

Definir o valor secreto

```json
resource "aws_secretsmanager_secret_version" "example_secret_value" {
  secret_id     = aws_secretsmanager_secret.example_secret.id
  secret_string = jsonencode({
    username = "example_user"
    password = "example_password"
  })
}
```

Política de IAM para permitir que os servidores Web acedam ao segredo

```json
resource "aws_iam_policy" "web_server_secrets_access" {
  name        = "web-server-secrets-access"
  description = "Allows web servers to access secrets in AWS Secrets Manager"

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ],
        Resource = aws_secretsmanager_secret.example_secret.arn
      }
    ]
  })
}
```

Anexar a política de IAM a uma função de IAM para servidores Web

```json
resource "aws_iam_role" "web_server_role" {
  name               = "web-server-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "attach_web_server_policy" {
  role       = aws_iam_role.web_server_role.name
  policy_arn = aws_iam_policy.web_server_secrets_access.arn
}
```

## Função lambda para rotação segredos

Crie uma função Lambda de espaço reservado para lidar com a rotação secreta.

Etapas para implantar:
- Empacote o código Lambda: Zipar o arquivo Python como rotation.zip.
- Implantar usando o Terraform: Certifique-se de que o nome do arquivo no manifesto do Terraform corresponda ao caminho do arquivo zip.
- Validar a rotação: O AWS Secrets Manager agora usará a função Lambda para fazer a rotação de segredos na programação definida.

```json
resource "aws_lambda_function" "secret_rotation_function" {
  filename         = "rotation.zip" # Replace with the zip file path of your Lambda function
  function_name    = "secret-rotation-function"
  role             = aws_iam_role.secret_rotation_role.arn
  handler          = "lambda_function.lambda_handler"
  runtime          = "python3.9"
  source_code_hash = filebase64sha256("rotation.zip")
}
```

Crie uma Função IAM para a função de rotação Lambda

```json
resource "aws_iam_role" "secret_rotation_role" {
  name               = "secret-rotation-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "lambda.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

Anexar políticas à função de rotação

```json
resource "aws_iam_role_policy_attachment" "attach_rotation_policy" {
  role       = aws_iam_role.secret_rotation_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "attach_secretsmanager_policy" {
  role       = aws_iam_role.secret_rotation_role.name
  policy_arn = "arn:aws:iam::aws:policy/SecretsManagerReadWrite"
}
```

## Exemplo de código de função Lambda

```python
import boto3
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

secretsmanager = boto3.client('secretsmanager')

def lambda_handler(event, context):
    try:
        # Step 1: Parse the rotation event
        step = event['Step']
        secret_arn = event['SecretId']
        
        if step == "createSecret":
            create_secret(secret_arn)
        elif step == "setSecret":
            set_secret(secret_arn)
        elif step == "testSecret":
            test_secret(secret_arn)
        elif step == "finishSecret":
            finish_secret(secret_arn)
        else:
            raise ValueError(f"Invalid step: {step}")
    except Exception as e:
        logger.error(f"Error handling rotation event: {e}")
        raise

def create_secret(secret_arn):
    logger.info(f"Creating a new secret for ARN: {secret_arn}")
    # Retrieve the current secret value
    secret = secretsmanager.get_secret_value(SecretId=secret_arn)
    current_secret = json.loads(secret['SecretString'])

    # Generate a new secret value
    new_secret = {
        "username": current_secret['username'],
        "password": "NewSecurePassword123!"  # Replace with your logic to generate a secure password
    }

    # Store the new secret version
    secretsmanager.put_secret_value(
        SecretId=secret_arn,
        SecretString=json.dumps(new_secret),
        VersionStages=['AWSPENDING']
    )
    logger.info(f"New secret version created for ARN: {secret_arn}")

def set_secret(secret_arn):
    logger.info(f"Setting the pending secret for ARN: {secret_arn}")
    # Implementation depends on how the secret is applied in the system (e.g., database credentials)
    # Example: Apply the secret in the system
    pass

def test_secret(secret_arn):
    logger.info(f"Testing the pending secret for ARN: {secret_arn}")
    # Validate the new secret
    # Example: Test new database credentials
    pass

def finish_secret(secret_arn):
    logger.info(f"Finishing the rotation for ARN: {secret_arn}")
    # Mark the pending version as the current version
    secretsmanager.update_secret_version_stage(
        SecretId=secret_arn,
        VersionStage='AWSCURRENT',
        MoveToVersionId=secretsmanager.describe_secret(SecretId=secret_arn)['VersionId'],
        RemoveFromVersionId='AWSPENDING'
    )
    logger.info(f"Rotation complete for ARN: {secret_arn}")

```
a