# 🚀 Infra Terraform + GitHub Actions — S3 Pipeline

> Automação de infraestrutura AWS com Terraform e GitHub Actions, com deploy separado por ambiente (DEV e PROD) usando Reusable Workflows e autenticação OIDC (keyless).

![Terraform](https://img.shields.io/badge/Terraform-1.8.3-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20IAM%20%7C%20DynamoDB-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)

---

## 📐 Visão Geral

```
Dev → push (develop) → GitHub Actions → Terraform → AWS DEV  (S3)
                ↓ merge
            branch main  → GitHub Actions → Terraform → AWS PROD (S3)
```

| Branch | Trigger | Workflow chamado | Ambiente |
|--------|---------|-----------------|----------|
| `develop` | `git push` | `develop.yml` → `terraform.yml` | DEV |
| `main` | `git merge` | `main.yml` → `terraform.yml` | PROD |

---

## 🗂️ Estrutura do Repositório

```
terraform-infra-pipeline/
├── .github/
│   └── workflows/
│       ├── terraform.yml       # Reusable Workflow — lógica central do pipeline
│       ├── develop.yml         # Gatilho DEV (branch develop)
│       └── main.yml            # Gatilho PROD (branch main)
├── infra/
│   ├── envs/
│   │   ├── dev/
│   │   │   └── terraform.tfvars   # Variáveis do ambiente DEV
│   │   └── prod/
│   │       └── terraform.tfvars   # Variáveis do ambiente PROD
│   ├── backend.tf              # Remote State (S3) + State Locking (DynamoDB)
│   ├── destroy_config.json     # Flag booleana para Terraform Destroy
│   ├── main.tf                 # Definição dos recursos (S3 Bucket)
│   ├── provider.tf             # Provider AWS e região
│   └── variables.tf            # Declaração das variáveis
├── .gitignore
└── README.md
```

---

## 📄 Arquivos — Passo a Passo

---

### PASSO 01 — `infra/variables.tf`

> Declara todas as variáveis utilizadas nos recursos Terraform. Mantém o código flexível e reutilizável entre ambientes.

```hcl
variable "bucket_name" {
  type = string
}
```

> 💡 A variável `bucket_name` é sobrescrita pelo `terraform.tfvars` de cada ambiente (dev ou prod), passado com `-var-file` no pipeline.

---

### PASSO 02 — `infra/provider.tf`

> Configura o provider AWS e define a região padrão onde os recursos serão provisionados.

```hcl
provider "aws" {
  region = "us-east-1"
}
```

> 💡 O provider é configurado uma única vez aqui. A autenticação com a AWS é feita pelo GitHub Actions via OIDC — sem Access Keys no código.

---

### PASSO 03 — `infra/backend.tf`

> Define onde o Terraform State será armazenado (S3) e como será travado durante execuções concorrentes (DynamoDB).

```hcl
terraform {
  backend "s3" {}
}
```

> ⚠️ O backend vazio `{}` é intencional. Os valores reais (bucket, região, tabela DynamoDB) são injetados pelo GitHub Actions via `-backend-config` no passo de `terraform init`, mantendo o código limpo e sem dados sensíveis hardcoded.

---

### PASSO 04 — `infra/main.tf`

> Define o recurso principal do projeto: um bucket S3 cujo nome é parametrizado por ambiente.

```hcl
resource "aws_s3_bucket" "bucket" {
  bucket = var.bucket_name
}
```

> 💡 O Terraform Workspace garante que DEV e PROD sejam completamente isolados no mesmo backend. Em projetos reais, este arquivo cresce com mais recursos como políticas de bucket, versionamento e replicação.

---

### PASSO 05 — `infra/destroy_config.json`

> Controle booleano que determina se o Terraform deve destruir ou provisionar os recursos no próximo pipeline.

```json
{
  "dev":  false,
  "prod": false
}
```

> ⚠️ Com ambos em `false`, o pipeline sempre executa Plan + Apply. Mude `dev` para `true` para destruir o ambiente de desenvolvimento sem afetar o PROD. Esta abordagem evita destruições acidentais por parâmetro errado na linha de comando.

---

### PASSO 06 — `infra/envs/dev/terraform.tfvars`

> Variáveis específicas do ambiente de desenvolvimento.

```hcl
bucket_name = "dev-us-east-1-buildrun-video-pipeline-proj1"
```

---

### PASSO 07 — `infra/envs/prod/terraform.tfvars`

> Variáveis específicas do ambiente de produção.

```hcl
bucket_name = "prod-us-east-1-buildrun-video-pipeline-proj2"
```

> 💡 O prefixo `dev-` ou `prod-` no nome do bucket garante isolamento visual e facilita políticas IAM por ambiente. Ambos os arquivos são passados ao Terraform com `-var-file=./envs/{environment}/terraform.tfvars`.

---

### PASSO 08 — `.github/workflows/terraform.yml`

> O coração do pipeline. Este é o **Reusable Workflow** que centraliza toda a lógica de CI/CD. É chamado pelos outros dois workflows via `uses:`.

```yaml
name: "Terraform Workflow"

on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
      aws-assume-role-arn:
        type: string
        required: true
      aws-region:
        type: string
        required: true
      aws-statefile-s3-bucket:
        type: string
        required: true
      aws-lock-dynamodb-table:
        type: string
        required: true

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.8.3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ inputs.aws-assume-role-arn }}
          role-session-name: GitHub_to_AWS_via_FederatedOIDC
          aws-region: ${{ inputs.aws-region }}

      - name: Read destroy configuration
        id: read-destroy-config
        run: |
          DESTROY="$(jq -r '.${{ inputs.environment }}' ./infra/destroy_config.json)"
          echo "destroy=$(echo $DESTROY)" >> $GITHUB_OUTPUT

      - name: Terraform Init
        run: |
          cd infra && terraform init \
            -backend-config="bucket=${{ inputs.aws-statefile-s3-bucket }}" \
            -backend-config="key=${{ github.event.repository.name }}" \
            -backend-config="region=${{ inputs.aws-region }}" \
            -backend-config="dynamodb_table=${{ inputs.aws-lock-dynamodb-table }}"

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Destroy
        if: steps.read-destroy-config.outputs.destroy == 'true'
        id: terraform-destroy
        run: |
          cd infra &&
          terraform workspace select ${{ inputs.environment }} || \
          terraform workspace new ${{ inputs.environment }} && \
          terraform destroy \
            -var-file="./envs/${{ inputs.environment }}/terraform.tfvars" \
            -auto-approve

      - name: Terraform Plan
        if: steps.read-destroy-config.outputs.destroy != 'true'
        id: terraform-plan
        run: |
          cd infra &&
          terraform workspace select ${{ inputs.environment }} || \
          terraform workspace new ${{ inputs.environment }} && \
          terraform plan \
            -var-file="./envs/${{ inputs.environment }}/terraform.tfvars" \
            -out="${{ inputs.environment }}.plan"

      - name: Terraform Apply
        if: steps.read-destroy-config.outputs.destroy != 'true'
        id: terraform-apply
        run: |
          cd infra &&
          terraform workspace select ${{ inputs.environment }} || \
          terraform workspace new ${{ inputs.environment }} && \
          terraform apply "${{ inputs.environment }}.plan"
```

> ℹ️ O `if` condicional nos steps de Destroy/Plan/Apply lê a saída do step `read-destroy-config`. Isso permite que o mesmo workflow gerencie tanto provisionamento quanto destruição sem duplicação de código.

---

### PASSO 09 — `.github/workflows/develop.yml`

> Gatilho do ambiente DEV. Monitora a branch `develop` e aciona o Reusable Workflow com os parâmetros de desenvolvimento.

```yaml
name: "DEV DEPLOY"

on:
  push:
    branches:
      - develop

permissions:
  id-token: write   # Necessário para autenticação OIDC com a AWS
  contents: read

jobs:
  terraform:
    uses: ./.github/workflows/terraform.yml
    with:
      environment:             dev
      aws-assume-role-arn:     "arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>"
      aws-region:              "us-east-1"
      aws-statefile-s3-bucket: "<seu-bucket-state>"
      aws-lock-dynamodb-table: "<sua-tabela-lock>"
```

> ⚠️ A permissão `id-token: write` é obrigatória para que o GitHub Actions possa solicitar um token OIDC temporário da AWS. Sem ela, a autenticação keyless não funciona.

---

### PASSO 10 — `.github/workflows/main.yml`

> Gatilho do ambiente PROD. Idêntico ao `develop.yml`, mas monitora a branch `main` e usa `environment: prod`.

```yaml
name: "PROD DEPLOY"

on:
  push:
    branches:
      - main

permissions:
  id-token: write   # Necessário para autenticação OIDC com a AWS
  contents: read

jobs:
  terraform:
    uses: ./.github/workflows/terraform.yml
    with:
      environment:             prod
      aws-assume-role-arn:     "arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>"
      aws-region:              "us-east-1"
      aws-statefile-s3-bucket: "<seu-bucket-state>"
      aws-lock-dynamodb-table: "<sua-tabela-lock>"
```

> 💡 A única diferença entre `develop.yml` e `main.yml` é a branch monitorada e o `environment` passado. Toda a lógica de execução vive no Reusable Workflow `terraform.yml`.

---

### PASSO 11 — `.gitignore`

> Garante que arquivos sensíveis e gerados pelo Terraform nunca sejam versionados no repositório.

```gitignore
# Terraform state e cache — nunca commitar!
*.tfstate
*.tfstate.backup
.terraform/
*.plan
**/.terraform.lock.hcl

# Variáveis locais sensíveis
terraform.tfvars.local

# Sistema operacional
.DS_Store
Thumbs.db
```

> 💡 O `.terraform.lock.hcl` pode ser versionado em projetos de equipe para garantir consistência nas versões de providers. Neste projeto ele é ignorado pois o pipeline sempre executa `terraform init` do zero.

---

## 🔐 Configuração OIDC na AWS

Este projeto usa **OpenID Connect (OIDC)** para autenticação, sem armazenar AWS Access Keys como secrets.

**1. Criar o Identity Provider na AWS:**
- URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

**2. Criar a IAM Role com trust policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<SEU_USUARIO>/<SEU_REPO>:*"
        },
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

**3.** Anexe as políticas necessárias (ex: `AmazonS3FullAccess` para este projeto).

---

## 🔄 Fluxo completo do Pipeline

```
Push / Merge
     │
     ▼
[develop.yml ou main.yml]
     │  Define: environment, role ARN, região, bucket state, tabela DynamoDB
     │  Chama: terraform.yml via uses:
     │
     ▼
[terraform.yml — Reusable Workflow]
     ├── ✅ Checkout do código
     ├── 🔧 Setup Terraform 1.8.3
     ├── 🔐 Configure AWS Credentials (OIDC — sem Access Keys)
     ├── 📖 Lê destroy_config.json → define flag destroy
     ├── 🏗️  Terraform Init (injeta backend config via -backend-config)
     ├── ✔️  Terraform Validate
     │
     ├── [Se destroy = true]
     │       └── 💥 Terraform Destroy
     │
     └── [Se destroy = false]
             ├── 📋 Terraform Plan  → gera {environment}.plan
             └── 🚀 Terraform Apply → aplica o .plan gerado
```

---

## 🛡️ Boas Práticas Implementadas

| Prática | Descrição |
|---------|-----------|
| ✅ **Autenticação Keyless (OIDC)** | Nenhuma AWS Access Key armazenada. A IAM Role é assumida temporariamente via token OIDC. |
| ✅ **Remote State (S3)** | O `terraform.tfstate` nunca existe localmente. Lido e escrito diretamente no S3. |
| ✅ **State Locking (DynamoDB)** | Impede que dois pipelines modifiquem a infraestrutura simultaneamente. |
| ✅ **Workspaces por Ambiente** | DEV e PROD isolados no mesmo backend — sem risco de um apply afetar o outro. |
| ✅ **Reusable Workflow** | Lógica de CI/CD centralizada. Novo ambiente = novo arquivo de gatilho, sem duplicar código. |
| ✅ **Destroy Controlado** | Flag booleana explícita evita destruições acidentais por erro de digitação. |

---

## 🏗️ Recursos Provisionados

| Recurso | Ambiente | Nome |
|---------|----------|------|
| S3 Bucket | DEV | `dev-us-east-1-buildrun-video-pipeline-proj1` |
| S3 Bucket | PROD | `prod-us-east-1-buildrun-video-pipeline-proj2` |

---

## 📚 Tecnologias

| Tecnologia | Uso |
|-----------|-----|
| **Terraform 1.8.3** | IaC — provisionamento dos recursos AWS |
| **GitHub Actions** | CI/CD — automação do pipeline |
| **AWS S3** | Recurso provisionado + armazenamento do Remote State |
| **AWS DynamoDB** | State Locking do Terraform |
| **AWS IAM + OIDC** | Autenticação keyless entre GitHub e AWS |

---

## 👤 Autor

**Brendo Trindade**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/SEU_PERFIL)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/SEU_USUARIO)

---

## 📄 Licença

Este projeto está sob a licença MIT.
