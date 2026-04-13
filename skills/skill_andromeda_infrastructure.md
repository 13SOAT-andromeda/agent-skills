---
name: andromeda-infrastructure
description: Padrão de infraestrutura como código (Terraform) para projetos Go. Define a estrutura de módulos, gestão de estado remoto, segredos e automação de deploy.
---

# Andromeda Infrastructure Skill (Terraform)

Este documento define o padrão de infraestrutura como código (IaC) para os projetos da Andromeda, baseado no modelo do `tech-challenge-s1`. Todos os novos ambientes e evoluções de infraestrutura devem seguir estas diretrizes.

---

## 🏗️ Estrutura de Diretórios (`infra/`)

A infraestrutura deve ser estritamente modularizada para garantir reuso e isolamento de recursos:

- `infra/main.tf`: Ponto de entrada que configura o provider AWS, o backend remoto e instancia os módulos.
- `infra/variables.tf`: Declaração de todas as variáveis necessárias (região, CIDRs, nomes de cluster).
- `infra/outputs.tf`: Exportação de dados críticos (VPC IDs, RDS Endpoints, EKS Cluster Name).
- `infra/modules/`:
    - `network/`: VPC, Subnets públicas e privadas, Internet Gateway e NAT Gateway.
    - `eks/`: Cluster EKS, Node Groups, IAM Roles (incluindo suporte a `LabRole` para AWS Academy) e Security Groups do cluster.
    - `rds/`: Instância PostgreSQL, Subnet Groups e regras de firewall permitindo acesso apenas pelo cluster EKS.

---

## 💾 Gestão de Estado (Remote State)

O estado do Terraform (`tfstate`) nunca deve ser armazenado localmente para evitar conflitos e perda de dados.

- **Backend:** S3 Bucket com versionamento habilitado.
- **Configuração Padrão:**
```hcl
terraform {
  backend "s3" {
    bucket = "tech-challenge-13-soat-tfstate"
    key    = "tfstate"
    region = "us-east-1"
  }
}
```

- **Bucket Creation:** Utilize o alvo `make create-tfstate-bucket` antes do primeiro deploy para garantir que o bucket exista com as permissões de bloqueio de acesso público.

---

## 🔐 Gestão de Variáveis e Segredos

- **Zero Hardcoding:** Nenhuma credencial ou ARN específico de ambiente deve estar no código `.tf`.
- **Injeção via TF_VAR:** Use o prefixo `TF_VAR_` para mapear variáveis do sistema para o Terraform sem expor dados sensíveis.
- **Workflow de Segredos:**
    1. Defina no `.env` (ex: `DB_PASSWORD=...`).
    2. O script `apply-terraform.sh` exporta como `export TF_VAR_db_password="$DB_PASSWORD"`.
    3. O Terraform consome via `variable "db_password" {}`.

---

## 🚀 Automação e Deploy

O ciclo de vida da infraestrutura é gerido por scripts que garantem consistência entre execuções locais e CI/CD.

### Script `apply-terraform.sh`
- Automatiza a sequência `init` -> `plan` -> `apply`.
- Carrega automaticamente o arquivo `.env` da raiz do projeto.
- Suporta a flag `--auto-approve` para uso em pipelines automatizados.

### Makefile (Interface Unificada)
- `make apply-terraform`: Atalho para o script de aplicação.
- `make down-terraform`: Atalho para o `terraform destroy`.
- `make create-tfstate-bucket`: Provisiona o bucket de estado via AWS CLI.

---

## 🔌 Integração com Kubernetes (K8s)

A infraestrutura deve facilitar o deploy da aplicação injetando dados dinamicamente:

1. **Recuperação de Endpoints:** O deploy deve recuperar o endereço do RDS via AWS CLI.
2. **Patch de ConfigMaps:** O workflow deve atualizar o ConfigMap do K8s com o endpoint provisionado.

---

## ✅ Padrões de Qualidade

- **Tags Obrigatórias:** Todos os recursos devem ser taggeados com o nome do projeto e ambiente.
- **Segurança de Rede:** O RDS deve sempre residir em subnets privadas sem IP público.
- **Idempotência:** O código deve ser executável múltiplas vezes sem causar duplicidade ou erros.

---
*Este modelo de infraestrutura é o padrão ouro para todos os projetos da Andromeda.*
