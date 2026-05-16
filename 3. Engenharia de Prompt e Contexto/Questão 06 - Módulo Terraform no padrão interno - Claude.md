## Prompt
```
[Context] A equipe de segurança e compliance definiu um novo padrão interno de IaC na AWS que todos os módulos Terraform devem seguir rigorosamente. O módulo será consumido por todos os times de engenharia da empresa, exigindo alta clareza e padronização.

[Action] Escreva um módulo Terraform reutilizável focado na criação de buckets S3.

[Result] O código gerado deve ser dividido em arquivos (variables.tf, main.tf e um exemplo de uso). O módulo deve cumprir os seguintes requisitos de compliance:

1. Todas as variáveis em variables.tf devem ter description e type declarados.
2. O nome do bucket criado deve utilizar o prefixo hvt-.
3. Todos os recursos criados devem conter as tags obrigatórias: Owner, CostCenter e Environment.
4. O bucket S3 deve ter recursos separados configurando: criptografia habilitada (padrão SSE-S3 AES256), versionamento ativo, bloqueio total de acesso público (Block Public Access) e logging de acessos habilitado (apontando para um bucket de logs recebido via variável).

[Example] Utilize exatamente o mesmo padrão de nomenclatura, declaração de locals e fusão de tags (função merge) mostrado neste módulo de VPC de referência:
```
`Terraform`
```
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}
variable "owner" {
  description = "Dono do recurso"
  type        = string
}
variable "cost_center" {
  description = "Centro de custo"
  type        = string
}
variable "cidr_block" {
  description = "CIDR block da VPC"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```

```
---
## Modelo
```
Claude Opus 4.7 Adaptativo
```
---
## Output

## `variables.tf`

```hcl
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

variable "owner" {
  description = "Dono do recurso"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo"
  type        = string
}

variable "bucket_name" {
  description = "Nome base do bucket S3 (será prefixado com 'hvt-')"
  type        = string
}

variable "log_bucket_id" {
  description = "ID do bucket S3 de destino para armazenamento dos logs de acesso"
  type        = string
}

variable "log_prefix" {
  description = "Prefixo (path) dentro do bucket de logs onde os access logs serão gravados"
  type        = string
  default     = "s3-access-logs/"
}
```

## `main.tf`

```hcl
locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  bucket_full_name = "hvt-${var.bucket_name}-${var.environment}"
}

# -----------------------------------------------------------------------------
# Bucket principal
# -----------------------------------------------------------------------------
resource "aws_s3_bucket" "this" {
  bucket = local.bucket_full_name

  tags = merge(local.common_tags, {
    Name = local.bucket_full_name
  })
}

# -----------------------------------------------------------------------------
# Criptografia em repouso - SSE-S3 (AES256)
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# -----------------------------------------------------------------------------
# Versionamento ativo
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

# -----------------------------------------------------------------------------
# Bloqueio total de acesso público (Block Public Access)
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# -----------------------------------------------------------------------------
# Logging de acessos - apontando para bucket de logs recebido via variável
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.log_bucket_id
  target_prefix = var.log_prefix
}
```

## `examples/usage/main.tf` (Exemplo de Uso)

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

module "s3_app_data" {
  source = "../../"

  environment   = "production"
  owner         = "sre-team"
  cost_center   = "CC-1042"
  bucket_name   = "app-data"
  log_bucket_id = "hvt-central-access-logs-production"
  log_prefix    = "s3-access-logs/app-data/"
}
```

O bucket resultante deste exemplo será criado com o nome `hvt-app-data-production`, satisfazendo a regra de prefixo obrigatório.

## Checklist de Compliance

| Requisito | Status | Implementação |
|-----------|--------|---------------|
| `description` e `type` em todas as variáveis | ✅ | `variables.tf` |
| Prefixo `hvt-` no nome do bucket | ✅ | `local.bucket_full_name` |
| Tags obrigatórias (Owner, CostCenter, Environment) | ✅ | `local.common_tags` + `merge()` |
| Criptografia SSE-S3 AES256 | ✅ | `aws_s3_bucket_server_side_encryption_configuration` |
| Versionamento ativo | ✅ | `aws_s3_bucket_versioning` |
| Block Public Access total | ✅ | `aws_s3_bucket_public_access_block` (4 flags `true`) |
| Logging em bucket externo | ✅ | `aws_s3_bucket_logging` com `var.log_bucket_id` |


---
## Justificativa
O framework CARE foi escolhido porque transforma um pedido subjetivo em uma especificação verificável: o Context ancora o porquê (compliance), a Action delimita o escopo, o Result serve como checklist auditável de aceitação e o Example impõe o padrão estilístico do time — eliminando ambiguidade em uma tarefa de IaC que não admite interpretação livre.