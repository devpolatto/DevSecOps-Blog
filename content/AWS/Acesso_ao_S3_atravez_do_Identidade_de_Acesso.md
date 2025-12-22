---
title: Acesso ao S3 atravez do Identidade de Acesso de Origem OIA
tags:
  - AWS
  - S3
  - OIA
  - IAM
enableToc: true
---
# Cenário

Uma empresa está desenvolvendo um aplicativo de compartilhamento de arquivos que usará um bucket do Amazon S3 para armazenamento. A empresa quer atender a todos os arquivos por meio de uma distribuição do Amazon CloudFront. A empresa não quer que os arquivos sejam acessíveis por meio de navegação direta para a URL do S3.

# Solução

Crie uma identidade de acesso de origem (OAI). Atribua o OAI à distribuição do CloudFront. Configure as permissões de bucket do S3 para que somente o OAI tenha permissão de leitura.

Criar o bucket do S3

```json
resource "aws_s3_bucket" "file_sharing_bucket" {
  bucket = "file-sharing-app-bucket"
  acl    = "private"
}
```

Configurar a política do bucket para restringir o acesso ao OAI

```json
resource "aws_s3_bucket_policy" "bucket_policy" {
  bucket = aws_s3_bucket.file_sharing_bucket.id

  policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "${aws_cloudfront_origin_access_identity.oai.iam_arn}"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${aws_s3_bucket.file_sharing_bucket.id}/*"
    }
  ]
}
POLICY
}
```

Criar a Identidade de Acesso de Origem (OAI)

```json
resource "aws_cloudfront_origin_access_identity" "oai" {
  comment = "OAI for file sharing application"
}
```

Criar a distribuição do CloudFront

```json
resource "aws_cloudfront_distribution" "cloudfront_distribution" {
  origin {
    domain_name = aws_s3_bucket.file_sharing_bucket.bucket_regional_domain_name
    origin_id   = "S3-origin"
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }
	enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    target_origin_id       = "S3-origin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods = ["GET", "HEAD"]
    cached_methods = ["GET", "HEAD"]
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400
  }
  price_class = "PriceClass_100"
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```