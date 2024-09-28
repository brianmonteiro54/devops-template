# DevOps Template - AWS ECR & ECS Deployment

Este repositório contém templates de workflows para rodar pipelines de CI/CD no GitHub Actions, com foco em integração e entrega contínua para **Amazon Elastic Container Registry (ECR)** e **Amazon Elastic Container Service (ECS)**.

## Estrutura

O repositório está estruturado para realizar **build**, **push** da imagem Docker para o ECR, e fazer o **deploy** da imagem no ECS. Ele utiliza três workflows principais:

1. **build.yml**: Responsável por construir e fazer o push da imagem Docker no ECR.
2. **bundle.yml**: Chamado internamente para conectar os workflows de build e deploy.
3. **deploy.yml**: Gerencia o deploy da imagem Docker para um cluster e serviço ECS.

## Repositório de Destino

O repositório de destino utiliza este repositório como um **template** para que o workflow de CI/CD possa ser reutilizado em diversos projetos. Isso significa que, em vez de duplicar os arquivos de workflow para cada repositório, basta referenciar o `bundle.yml` deste repositório template, o que torna o processo mais eficiente e centralizado.

Se você tiver vários repositórios que compartilham a mesma estrutura de CI/CD (build e deploy para AWS ECR e ECS), o uso deste template facilita a atualização e manutenção dos pipelines. Caso seja necessário fazer alterações nos workflows, como a adição de novos passos ou melhorias no processo de build, basta modificar o repositório template, e todos os repositórios que fazem referência ao `bundle.yml` serão automaticamente atualizados.

Dessa forma, você só precisa alterar as informações específicas de cada repositório de destino, como:

### Variáveis de Entrada

No repositório de destino, você só precisa alterar as variáveis de entrada para personalizar o pipeline:

- **`IMAGE_NAME`**: Nome da imagem Docker.
- **`TASK_NAME`**: Nome da tarefa ECS.
- **`CONTAINER_NAME`**: Nome do container no ECS.
- **`ECS_SERVICE`**: Nome do serviço ECS.
- **`ECS_CLUSTER`**: Nome do cluster ECS.

Esses parâmetros podem variar entre os repositórios de destino, enquanto o pipeline base permanece o mesmo.

### Uso do `PRIVATE_KEY`

O `PRIVATE_KEY` é opcional, mas pode ser usado para armazenar um certificado necessário para acessar recursos protegidos, como servidores ou serviços externos. Caso o segredo `PRIVATE_KEY` não seja configurado no repositório de destino, o workflow ainda funcionará normalmente.

## Pré-requisitos

Antes de rodar este pipeline, é necessário ter configurado:

1. **Amazon Web Services (AWS)** com permissões adequadas para:
   - Acessar o **ECR** para armazenar imagens Docker.
   - Fazer deploy no **ECS**.
2. **GitHub Secrets** para armazenar credenciais seguras:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_REGION`
   - `PRIVATE_KEY` (opcional).

Para configurar essas credenciais, acesse o repositório no GitHub e vá em **Settings > Secrets and variables > Actions**. Adicione as variáveis de ambiente necessárias.

A configuração dos pré-requisitos, **deve ser feita no repositório de destino**.

### Exemplo de configuração no repositório de destino:

```yaml
name: Build & Deploy - Production

on:
  repository_dispatch:
    types: [deploy-production]
  push:
    branches:
      - main  

jobs:
  workflow:
    uses: brianmonteiro54/devops-template/.github/workflows/bundle.yml@main

    with:
      IMAGE_NAME: api-production
      TASK_NAME: api-production
      CONTAINER_NAME: api-production
      ECS_SERVICE: api-production
      ECS_CLUSTER: production
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
      PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}
