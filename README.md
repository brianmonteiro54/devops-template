# DevOps Template
Este repositório contém templates de workflows para rodar pipelines de CI/CD no GitHub Actions.

### Terraform Pipelines

#### `terraform-ci.yml` — CI Completo

Pipeline de integração contínua com 6 jobs paralelos:

| Job | Função | Tipo |
|-----|--------|------|
| **Lint & Format** | `terraform fmt` + TFLint | 🔒 Bloqueante |
| **Validate** | `terraform validate` | 🔒 Bloqueante |
| **Security** | Checkov + Trivy (SARIF) | 🔒 Bloqueante |
| **Plan Preview** | `terraform plan` por ambiente (matrix) | Informativo |
| **Docs** | terraform-docs auto-generation | Auto-commit |
| **CI Gate** | Avaliação final bloqueante | Gate |

**Inputs disponíveis:**

| Input | Default | Descrição |
|-------|---------|-----------|
| `terraform-version` | `1.14.5` | Versão do Terraform |
| `tflint-version` | `v0.61.0` | Versão do TFLint |
| `checkov-version` | `3.2.354` | Versão do Checkov |
| `checkov-skip-checks` | `CKV_AWS_119,CKV_AWS_28` | Checks para ignorar |
| `trivy-severity` | `MEDIUM,HIGH,CRITICAL` | Severidades do Trivy |
| `aws-region` | `us-east-1` | Região AWS |
| `terraform-docs-working-dir` | `.` | Diretório do terraform-docs |
| `enable-plan-preview` | `true` | Habilitar plan preview em PRs |
| `environments` | `[{"branch":"main","env_name":"prod"},...]` | Matrix de environments para plan |

```yaml
jobs:
  ci:
    uses: brianmonteiro54/devops-template/.github/workflows/terraform-ci.yml@main
    with:
      terraform-docs-working-dir: '.'
      environments: '[{"branch":"main","env_name":"prod"},{"branch":"developer","env_name":"dev"}]'
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}
```

#### `terraform-cd.yml` — CD Completo

Pipeline de deploy com approval gates e verificação pós-apply:

| Job | Função |
|-----|--------|
| **Plan** | `terraform plan` com upload de artefato |
| **Drift Alert** | Alerta de drift (modo plan-only) |
| **Apply** | `terraform apply` com Environment Gate |
| **Verify** | Drift check pós-apply + smoke tests |
| **Notify Failure** | Summary de falha |

**Inputs disponíveis:**

| Input | Default | Descrição |
|-------|---------|-----------|
| `env-name` | _(required)_ | Ambiente alvo (dev, prod) |
| `tfvars-path` | _(required)_ | Caminho do terraform.tfvars |
| `backend-path` | _(required)_ | Caminho do backend.hcl |
| `is-drift-check` | `false` | Modo drift (só plan, sem apply) |
| `force-apply` | `false` | Forçar apply sem mudanças |
| `reason` | `''` | Justificativa para deploy manual |
| `checkout-ref` | `''` | Git ref para checkout |

```yaml
jobs:
  deploy:
    uses: brianmonteiro54/devops-template/.github/workflows/terraform-cd.yml@main
    with:
      env-name: prod
      tfvars-path: envs/prod/terraform.tfvars
      backend-path: envs/prod/backend.hcl
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}
```

**Drift Detection (modo plan-only):**

```yaml
jobs:
  drift:
    uses: brianmonteiro54/devops-template/.github/workflows/terraform-cd.yml@main
    with:
      env-name: prod
      tfvars-path: envs/prod/terraform.tfvars
      backend-path: envs/prod/backend.hcl
      is-drift-check: true
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}
```

#### `terraform-destroy.yml` — Destroy com Proteções

Pipeline de destroy com double-check e state backup:

| Job | Função |
|-----|--------|
| **Destroy Plan** | Preview do que será destruído |
| **Destroy** | `terraform destroy` com Environment Gate |
| **Verify** | Verifica se o state ficou limpo |

**Inputs disponíveis:**

| Input | Default | Descrição |
|-------|---------|-----------|
| `env-name` | _(required)_ | Ambiente alvo |
| `tfvars-path` | _(required)_ | Caminho do terraform.tfvars |
| `backend-path` | _(required)_ | Caminho do backend.hcl |
| `reason` | _(required)_ | Justificativa obrigatória |
| `target-resources` | `''` | Recursos específicos (comma-separated) |
| `skip-state-backup` | `false` | Pular backup do state |

```yaml
jobs:
  destroy:
    uses: brianmonteiro54/devops-template/.github/workflows/terraform-destroy.yml@main
    with:
      env-name: dev
      tfvars-path: envs/dev/terraform.tfvars
      backend-path: envs/dev/backend.hcl
      reason: 'Cleanup ambiente de teste'
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}
```

---

## Funcionalidades dos Terraform Pipelines

| Funcionalidade | Descrição |
|----------------|-----------|
| **PR Comments** | Comentários atualizáveis (edit, não duplica) com status de cada job |
| **SARIF** | Resultados do Trivy integrados na aba Security do GitHub |
| **Environment Gates** | Aprovação manual via GitHub Environments antes de apply/destroy |
| **State Backup** | Backup automático do state antes de apply/destroy (90 dias) |
| **Post-action Verify** | Drift check pós-apply e state check pós-destroy |
| **Drift Detection** | Modo plan-only para detecção agendada de mudanças manuais |
| **Smoke Tests** | Hook para `scripts/smoke-test.sh` customizável |
| **Step Summary** | Tabelas e detalhes no summary do GitHub Actions |

---

## Pré-requisitos

### Para workflows de Application (ECR/ECS)

| Secret | Descrição |
|--------|-----------|
| `AWS_ASSUME_ROLE_ARN` | ARN do role para OIDC federation |
| `AWS_REGION` | Região AWS |
| `PRIVATE_KEY` | Certificado PEM (opcional) |

### Para workflows de Terraform

| Secret | Descrição |
|--------|-----------|
| `AWS_ACCESS_KEY_ID` | Access key AWS |
| `AWS_SECRET_ACCESS_KEY` | Secret key AWS |
| `AWS_SESSION_TOKEN` | Session token (opcional — AWS Academy) |

### GitHub Environments (Terraform CD/Destroy)

Configure em **Settings → Environments** do repositório consumidor:

| Environment | Aprovadores | Uso |
|-------------|-------------|-----|
| `deploy-dev` | (opcional) | Apply em dev |
| `deploy-prod` | 1+ reviewer | Apply em prod |
| `destroy-dev` | 1 reviewer | Destroy em dev |
| `destroy-prod` | 2+ reviewers | Destroy em prod |

---

## Usando a Branch `togglemaster`

A branch `togglemaster` contém os callers prontos para o repositório [togglemaster-infrastructure](https://github.com/brianmonteiro54/togglemaster-infrastructure).

Para usar, copie os arquivos para o repositório de destino:

```bash
# Clone temporário da branch
git clone -b togglemaster https://github.com/brianmonteiro54/devops-template.git /tmp/devops-callers

# Copie os callers para o repo de destino
cp /tmp/devops-callers/.github/workflows/terraform-*.yml \
   <seu-repo>/.github/workflows/

rm -rf /tmp/devops-callers
```

---

## Versionamento

```yaml
# Desenvolvimento — sempre pega a última versão
uses: brianmonteiro54/devops-template/.github/workflows/terraform-ci.yml@main

# Produção — fixado em tag específica
uses: brianmonteiro54/devops-template/.github/workflows/terraform-ci.yml@v2.0.0
```
