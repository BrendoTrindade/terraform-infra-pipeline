# 🚀 Infra Terraform + GitHub Actions — S3 Pipeline

> Automação de infraestrutura AWS com Terraform e GitHub Actions, com deploy separado por ambiente (DEV e PROD) usando Reusable Workflows e OIDC Federation.

![Terraform](https://img.shields.io/badge/Terraform-1.8.3-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20IAM%20%7C%20DynamoDB-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)

---

## 📐 Arquitetura

```
Dev → Push (develop branch) → GitHub Actions → Terraform → AWS DEV (S3)
                ↓ Merge
            main branch  → GitHub Actions → Terraform → AWS PROD (S3)
```

O pipeline segue o fluxo **GitOps**:
- Push na branch `develop` → deploy automático no ambiente **DEV**
- Merge para `main` → deploy automático no ambiente **PROD**
- Um único **Reusable Workflow** (`terraform.yml`) centraliza toda a lógica de CI/CD
- O `destroy_config.json` controla se os recursos serão destruídos ou provisionados

---

## 🗂️ Estrutura do Repositório

```
├── .github/
│   └── workflows/
│       ├── terraform.yml       # 🔁 Reusable Workflow (lógica central)
│       ├── main.yml            # 🟢 Gatilho PROD (branch main)
│       └── develop.yml         # 🔵 Gatilho DEV (branch develop)
│
└── infra/
    ├── envs/
    │   ├── dev/
    │   │   └── terraform.tfvars    # Variáveis do ambiente DEV
    │   └── prod/
    │       └── terraform.tfvars    # Variáveis do ambiente PROD
    ├── main.tf             # Definição dos recursos (S3 Bucket)
    ├── provider.tf         # Provider AWS — região us-east-1
    ├── variables.tf        # Declaração das variáveis
    ├── backend.tf          # Remote State no S3 + Lock DynamoDB
    └── destroy_config.json # Flag booleana para Terraform Destroy
```

---

## ⚙️ Pré-requisitos

Antes de usar este projeto, certifique-se de ter:

- [ ] Conta AWS com permissões para criar S3, IAM Role, DynamoDB
- [ ] Bucket S3 para armazenar o **Terraform State** (remote backend)
- [ ] Tabela DynamoDB para **State Locking**
- [ ] IAM Role configurada com **OIDC Federation** para o GitHub Actions
- [ ] Terraform `>= 1.8.3` instalado localmente (para testes)

---

## 🔐 Autenticação AWS — OIDC (Keyless)

Este projeto usa **OpenID Connect (OIDC)** para autenticação entre GitHub Actions e AWS, **sem necessidade de armazenar AWS Access Keys como secrets**.

### Como configurar a IAM Role:

1. No console da AWS, crie um **Identity Provider** do tipo OpenID Connect:
   - **URL:** `https://token.actions.githubusercontent.com`
   - **Audience:** `sts.amazonaws.com`

2. Crie uma **IAM Role** com trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::SEU_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:SEU_USUARIO/SEU_REPO:*"
        },
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

3. Anexe as políticas necessárias (ex: `AmazonS3FullAccess` para este projeto)

---

## 🚀 Como usar

### 1. Clone o repositório

```bash
git clone https://github.com/SEU_USUARIO/SEU_REPO.git
cd SEU_REPO
```

### 2. Configure o Remote Backend

Edite o `backend.tf` ou passe as variáveis via `terraform init`:

```bash
cd infra
terraform init \
  -backend-config="bucket=SEU_BUCKET_STATE" \
  -backend-config="key=nome-do-projeto" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=SUA_TABELA_LOCK"
```

### 3. Ajuste as variáveis de ambiente

**`infra/envs/dev/terraform.tfvars`**
```hcl
bucket_name = "dev-us-east-1-buildrun-video-pipeline-proj1"
```

**`infra/envs/prod/terraform.tfvars`**
```hcl
bucket_name = "prod-us-east-1-buildrun-video-pipeline-proj2"
```

### 4. Configure o destroy_config.json

```json
{
  "dev": false,
  "prod": false
}
```

> 💡 Defina `true` para destruir os recursos do ambiente correspondente no próximo pipeline.

### 5. Configure os inputs nos workflows

Em `.github/workflows/develop.yml` e `main.yml`, atualize:

```yaml
aws-assume-role-arn: "arn:aws:iam::SEU_ACCOUNT_ID:role/SUA_ROLE"
aws-statefile-s3-bucket: "SEU_BUCKET_STATE"
aws-lock-dynamodb-table: "SUA_TABELA_LOCK"
```

### 6. Faça push e veja a mágica acontecer!

```bash
# Deploy em DEV
git checkout develop
git add .
git commit -m "feat: add infra config"
git push origin develop

# Deploy em PROD
git checkout main
git merge develop
git push origin main
```

---

## 🔄 Pipeline — Fluxo detalhado

```
Push/Merge
    │
    ▼
[develop.yml / main.yml]
    │  Trigger: branch push
    │  Chama: terraform.yml (Reusable Workflow)
    │
    ▼
[terraform.yml]
    ├── ✅ Checkout do código
    ├── 🔧 Setup Terraform 1.8.3
    ├── 🔐 Configure AWS Credentials (OIDC)
    ├── 📖 Lê destroy_config.json
    ├── 🏗️  Terraform Init (Remote Backend S3)
    ├── ✔️  Terraform Validate
    │
    ├── [Se destroy = true]
    │       └── 💥 Terraform Destroy
    │
    └── [Se destroy = false]
            ├── 📋 Terraform Plan
            └── 🚀 Terraform Apply
```

---

## 🏗️ Recursos provisionados

| Recurso | Ambiente | Nome |
|---------|----------|------|
| S3 Bucket | DEV | `dev-us-east-1-buildrun-video-pipeline-proj1` |
| S3 Bucket | PROD | `prod-us-east-1-buildrun-video-pipeline-proj2` |

---

## 🛡️ Boas práticas implementadas

- ✅ **Keyless Authentication** — sem AWS secrets expostos no repositório
- ✅ **Remote State** — Terraform state armazenado no S3
- ✅ **State Locking** — DynamoDB previne execuções concorrentes
- ✅ **Workspaces** — ambientes DEV e PROD isolados no mesmo backend
- ✅ **Reusable Workflow** — lógica de CI/CD centralizada e reutilizável
- ✅ **Destroy controlado** — flag booleana evita destruições acidentais
- ✅ **GitOps** — infraestrutura versionada e auditável via Git

---

## 📚 Tecnologias

| Tecnologia | Uso |
|-----------|-----|
| **Terraform** | IaC — provisionamento dos recursos AWS |
| **GitHub Actions** | CI/CD — automação do pipeline |
| **AWS S3** | Recurso provisionado + Remote State |
| **AWS DynamoDB** | State Locking do Terraform |
| **AWS IAM + OIDC** | Autenticação keyless |

---

## 👤 Autor

**Brendo Trindade**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/SEU_PERFIL)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/SEU_USUARIO)

---

## 📄 Licença

Este projeto está sob a licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.
