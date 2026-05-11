# Questão 06 — Módulo Terraform no padrão interno

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** Sonnet 4.6
**Framework:** C-A-R-E

---

## Contexto

> A empresa possui um padrão interno de IaC em Terraform com requisitos obrigatórios de segurança e compliance para todos os times: tags obrigatórias, prefixo `hvt-` em nomes de recursos e configurações mandatórias em buckets S3 (encryption, versioning, block public access e logging).

---

## Prompt usado

```
# Context
Você é um engenheiro sênior de infraestrutura especializado em Terraform e AWS, responsável por criar módulos reutilizáveis que serão consumidos por todos os times de uma empresa. A empresa possui um padrão interno de IaC com os seguintes requisitos obrigatórios de segurança e compliance:

- Tags obrigatórias em todo recurso: `Owner`, `CostCenter`, `Environment`
- Prefixo `hvt-` em todos os nomes de recursos
- Todo bucket S3 deve ter:
  - Encryption habilitada (SSE-S3 como mínimo)
  - Versioning ativo
  - Block public access total (todos os 4 flags)
  - Logging configurado
- Todas as variáveis de entrada devem estar em `variables.tf` com os
  atributos `description` e `type` obrigatoriamente preenchidos

# Action
Crie um módulo Terraform reutilizável para provisionar buckets S3 totalmente aderente ao padrão interno descrito acima. O módulo deve:

1. Ser estruturado nos arquivos: `main.tf`, `variables.tf` e `outputs.tf`
2. Usar `locals` para consolidar as tags comuns (Owner, CostCenter, Environment)
3. Implementar recursos separados para cada configuração do bucket:
   - `aws_s3_bucket`
   - `aws_s3_bucket_server_side_encryption_configuration`
   - `aws_s3_bucket_versioning`
   - `aws_s3_bucket_public_access_block`
   - `aws_s3_bucket_logging`
4. Aplicar merge das common_tags com tags específicas de cada recurso
5. Garantir que todos os nomes sigam o padrão `hvt-` como prefixo
6. Incluir um arquivo `examples/basic/main.tf` com exemplo real de uso do módulo

# Result
O resultado esperado é um módulo Terraform production-ready, que:

- Passe em um `terraform validate` sem erros
- Seja imediatamente consumível por qualquer time via bloco `module {}`
- Garanta 100% de aderência ao padrão de segurança e compliance da empresa
- Seja legível e autodocumentado pelas descriptions das variáveis
- Entregue o código organizado por arquivo, com cabeçalho identificando
  cada um (ex: `# variables.tf`, `# main.tf`, `# outputs.tf`)

# Example
Siga rigorosamente este estilo de referência, extraído do módulo de VPC já existente na empresa:

# variables.tf
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

# main.tf
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

O módulo S3 deve adotar o mesmo padrão de nomenclatura de variáveis (snake_case), a mesma estrutura de locals para tags, e o mesmo uso de merge() para composição de tags nos recursos.
```

---

## Output completo da IA

```
# variables.tf

variable "bucket_name" {
  description = "Nome base do bucket S3 (o prefixo hvt- será adicionado automaticamente)"
  type        = string
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

variable "owner" {
  description = "Time ou pessoa responsável pelo recurso"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo associado ao recurso para fins de billing"
  type        = string
}

variable "logging_target_bucket" {
  description = "Nome do bucket S3 de destino para os logs de acesso"
  type        = string
}

variable "logging_target_prefix" {
  description = "Prefixo dos objetos de log no bucket de destino"
  type        = string
  default     = "s3-access-logs/"
}

variable "tags" {
  description = "Tags adicionais a serem aplicadas nos recursos além das tags obrigatórias"
  type        = map(string)
  default     = {}
}

---

# main.tf

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  bucket_name = "hvt-${var.bucket_name}-${var.environment}"
}

resource "aws_s3_bucket" "this" {
  bucket = local.bucket_name

  tags = merge(local.common_tags, var.tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
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
  bucket        = aws_s3_bucket.this.id
  target_bucket = var.logging_target_bucket
  target_prefix = var.logging_target_prefix
}

---

# outputs.tf

output "bucket_id" {
  description = "ID (nome) do bucket S3 criado"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN do bucket S3 criado"
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "Nome de domínio do bucket S3"
  value       = aws_s3_bucket.this.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "Nome de domínio regional do bucket S3"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}

---

# examples/basic/main.tf

terraform {
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

module "s3_artifacts" {
  source = "../../"

  bucket_name   = "artifacts"
  environment   = "production"
  owner         = "platform-team"
  cost_center   = "CC-1234"

  logging_target_bucket = "hvt-logs-central-production"
  logging_target_prefix = "s3-access-logs/artifacts/"

  tags = {
    Project = "internal-platform"
    Tier    = "storage"
  }
}

output "artifacts_bucket_arn" {
  value = module.s3_artifacts.bucket_arn
}
```

---

## Análise do framework

**Justificativa:** O prompt segue o framework **C-A-R-E**, que organiza a instrução em quatro camadas: primeiro estabelece quem é o agente e quais são as restrições do domínio (Context), depois define a tarefa com itens numerados e tecnicamente precisos (Action), em seguida declara os critérios de aceitação do entregável (Result) e finalmente ancora o estilo esperado em código real já existente na empresa (Example). Essa estrutura elimina ambiguidade em cada dimensão: papel/restrições, escopo/tarefas, critérios/qualidade e estilo/convenção.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| C — Context | `"Você é um engenheiro sênior de infraestrutura especializado em Terraform e AWS... A empresa possui um padrão interno de IaC com os seguintes requisitos obrigatórios de segurança e compliance: Tags obrigatórias... Prefixo hvt-... Todo bucket S3 deve ter..."` |
| A — Action | `"Crie um módulo Terraform reutilizável para provisionar buckets S3... O módulo deve: 1. Ser estruturado nos arquivos: main.tf, variables.tf e outputs.tf; 2. Usar locals para consolidar as tags comuns..."` |
| R — Result | `"O resultado esperado é um módulo Terraform production-ready, que: Passe em um terraform validate sem erros; Seja imediatamente consumível por qualquer time via bloco module {}..."` |
| E — Example | `"Siga rigorosamente este estilo de referência, extraído do módulo de VPC já existente na empresa: [código do módulo de VPC com locals, merge() e convenção de nomes]"` |
