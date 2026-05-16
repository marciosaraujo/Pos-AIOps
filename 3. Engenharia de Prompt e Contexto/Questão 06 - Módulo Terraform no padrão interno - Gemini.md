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
Terraform
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
---
## Modelo
```
Gemini 3.1 Pro
```
---
## Output
```
variables.tf

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

variable "owner" {
  description = "Dono do recurso (Squad ou Departamento)"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo para billing"
  type        = string
}

variable "bucket_name" {
  description = "Nome base do bucket S3 (o prefixo hvt- será adicionado automaticamente)"
  type        = string
}

variable "log_bucket_id" {
  description = "ID do bucket S3 de destino para os logs de acesso"
  type        = string
}
```
```
main.tf

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
  
  full_bucket_name = "hvt-${var.bucket_name}-${var.environment}"
}

resource "aws_s3_bucket" "this" {
  bucket = local.full_bucket_name

  tags = merge(local.common_tags, {
    Name = local.full_bucket_name
  })
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.log_bucket_id
  target_prefix = "log/${local.full_bucket_name}/"
}
````

```
examples/main.tf (Exemplo de Consumo)  

module "secure_s3_bucket" {
  source = "../"

  # Nomenclatura e Compliance
  bucket_name = "app-assets"
  environment = "production"
  owner       = "squad-frontend"
  cost_center = "cc-10293"

  # Configuração de Segurança (Bucket de logs pré-existente)
  log_bucket_id = "hvt-central-logs-production"
}
```
---
## Justificativa
O framework C-A-R-E estrutura o prompt definindo a autoridade corporativa e o escopo do módulo S3 (Context), seguido do comando técnico direto para a criação do recurso (Action). Em seguida, lista os requisitos técnicos e de segurança obrigatórios (Result) e utiliza um código de referência da empresa para garantir a padronização exata de estilo, nomenclatura e formatação de tags (Example).