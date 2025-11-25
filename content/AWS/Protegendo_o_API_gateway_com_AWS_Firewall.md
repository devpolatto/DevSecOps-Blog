---
title: Protegendo o API gateway com AWS Firewall
tags:
  - AWS
  - Segurança
  - Firewall
enableToc: true
---

# Cenário

Uma empresa global está usando o Amazon API Gateway para projetar APIs REST para seus usuários de clubes de fidelidade na região us-east-1 e na região ap-southeast-2. Um arquiteto de soluções deve projetar uma solução para proteger essas APIs REST gerenciadas do API Gateway em várias contas contra injeção de SQL e ataques de script entre sites
Qual solução atenderá a esses requisitos com o MENOR esforço administrativo?
# Solução:
Configure o AWS WAF em ambas as regiões. Associe ACLs da Web regionais a um estágio de API.
## **AWS WAF ACL**:
Define Web ACLs para cada região.

```json
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_wafv2_web_acl" "api_gateway_acl_us_east_1" {
  provider     = aws.us_east_1
  name         = "api-gateway-waf-us-east-1"
  scope        = "REGIONAL"
  description  = "WAF ACL for protecting API Gateway in us-east-1"
  default_action {
    allow {}
  }
  
  rule {
    name     = "BlockSQLInjection"
    priority = 1
    statement {
      sqli_match_statement {
        field_to_match {
          body {}
        }
        text_transformation {
          priority = 0
          type     = "URL_DECODE"
        }
      }
    }
    action {
      block {}
    }
    visibility_config {
      sampled_requests_enabled = true
      cloudwatch_metrics_enabled = true
      metric_name = "block-sql-injection"
    }
  }

  rule {
    name     = "BlockXSS"
    priority = 2
    statement {
      xss_match_statement {
        field_to_match {
          body {}
        }
        text_transformation {
          priority = 0
          type     = "HTML_ENTITY_DECODE"
        }
      }
    }
    action {
      block {}
    }
    visibility_config {
      sampled_requests_enabled = true
      cloudwatch_metrics_enabled = true
      metric_name = "block-xss"
    }
  }

  visibility_config {
    sampled_requests_enabled = true
    cloudwatch_metrics_enabled = true
    metric_name = "api-gateway-waf-us-east-1"
  }
}
```


```json
provider "aws" {
  alias  = "ap_southeast_2"
  region = "ap-southeast-2"
}

resource "aws_wafv2_web_acl" "api_gateway_acl_ap_southeast_2" {
  provider     = aws.ap_southeast_2
  name         = "api-gateway-waf-ap-southeast-2"
  scope        = "REGIONAL"
  description  = "WAF ACL for protecting API Gateway in ap-southeast-2"
  default_action {
    allow {}
  }

  rule {
    name     = "BlockSQLInjection"
    priority = 1
    statement {
      sqli_match_statement {
        field_to_match {
          body {}
        }
        text_transformation {
          priority = 0
          type     = "URL_DECODE"
        }
      }
    }
    action {
      block {}
    }
    visibility_config {
      sampled_requests_enabled = true
      cloudwatch_metrics_enabled = true
      metric_name = "block-sql-injection"
    }
  }

  rule {
    name     = "BlockXSS"
    priority = 2
    statement {
      xss_match_statement {
        field_to_match {
          body {}
        }
        text_transformation {
          priority = 0
          type     = "HTML_ENTITY_DECODE"
        }
      }
    }
    action {
      block {}
    }
    visibility_config {
      sampled_requests_enabled = true
      cloudwatch_metrics_enabled = true
      metric_name = "block-xss"
    }
  }

  visibility_config {
    sampled_requests_enabled = true
    cloudwatch_metrics_enabled = true
    metric_name = "api-gateway-waf-ap-southeast-2"
  }
}

```

## **Associação ao API Gateway**:

Vincula cada Web ACL ao estágio de produção (`prod`) das APIs REST no API Gateway, usando os ARNs das ACLs.

```json
resource "aws_api_gateway_stage" "api_stage_us_east_1" {
  provider       = aws.us_east_1
  stage_name     = "prod"
  rest_api_id    = "<your-api-id>"
  deployment_id  = "<your-deployment-id>"
  
  wafv2_web_acl_arn = aws_wafv2_web_acl.api_gateway_acl_us_east_1.arn
}

resource "aws_api_gateway_stage" "api_stage_ap_southeast_2" {
  provider       = aws.ap_southeast_2
  stage_name     = "prod"
  rest_api_id    = "<your-api-id>"
  deployment_id  = "<your-deployment-id>"

  wafv2_web_acl_arn = aws_wafv2_web_acl.api_gateway_acl_ap_southeast_2.arn
}
```