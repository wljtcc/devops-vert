## Documentação do Pipeline Azure DevOps - Aleteia Core Service

## Pipeline
- **Nome do Pipeline**: `aleteia-core`
- **URL do Pipeline**: `https://dev.azure.com/comercialvertanalytics/PLATAFORMA%20ONDA/_build?definitionId=10`

# Documentaçãoo do Pipeline Azure DevOps - Aleteia Core Service

## 1. Visão Geral

Este pipeline automatiza o processo de **Build, Push e Deploy** do serviço `core-service` (Aleteia) em diferentes ambientes Kubernetes (EKS) na AWS. Ele foi projetado para ser dinâmico, utilizando parâmetros para selecionar o ambiente alvo, e realiza o restart dos deployments ao invés de uma atualização tradicional de imagem.

**Autor:** Wellington Terrão  
**Data:** 29/01/2026

## 2. Gatilhos (Triggers)

| Configuração | Valor | Descrição |
| :--- | :--- | :--- |
| `trigger` | `none` | O pipeline **não** é iniciado automaticamente por pushes no repositório. |
| `pr` | `none` | O pipeline **não** é iniciado automaticamente por Pull Requests. |

> **Nota:** A execução é **manual**, exigindo que um usuário escolha o ambiente no momento da execução.

## 3. Parâmetros

| Nome | Tipo | Valores Permitidos | Valor Padrão | Descrição |
| :--- | :--- | :--- | :--- | :--- |
| `pEnvironment` | `string` | `develop`, `staging`, `demo`, `production` | `develop` | Define o ambiente Kubernetes que será alvo do deploy. |

## 4. Variáveis Globais

| Variável | Valor | Descrição |
| :------- | :---- | :---------- |
| `ALETEIA_SERVICE` | `aleteia-core` | Nome base do serviço. |
| `APP_SERVICE` | `core-service` | Nome do deployment/ serviço no Kubernetes. |
| `AWS_REGION` | `us-east-2` | Região AWS onde os recursos estão hospedados. |

### 4.1. Mapeamento Dinâmico de Infraestrutura

Com base no parâmetro `pEnvironment`, as seguintes variáveis são definidas:

| Ambiente | CLUSTER | NAMESPACE |
| :--- | :--- | :--- |
| `develop` | `onda-demo-eks` | `aleteia` |
| `staging` | `onda-demo-eks` | `aleteia-stg` |
| `demo` | `plataformaonda-eks` | `demo` |
| `production` | `plataformaonda-eks` | `prod` |

## 5. Arquitetura do Pipeline

O pipeline é dividido em dois estágios principais:

- **Estágio Build**: Compilação, criação da imagem Docker e push para o ECR.
- **Estágio Deploy**: Conexão ao cluster EKS e restart dos deployments.

```
Início Manual → Estágio Build → (se sucesso) → Estágio Deploy → Fim
```

## 6. Estágio 1: Build e Push

**Nome:** `Build`  
**Display Name:** `Build e Push - Ambiente: <ambiente_selecionado>`

### 6.1. Job: `BuildAndPush`

**Pool:** `ubuntu-latest`


#### Passos:

1. **Instalar JDK 21**
   - Task: `JavaToolInstaller@0`
   - Configuração: Instala o JDK 21 x64.

2. **Build com Gradle**
   - Task: `Gradle@3`
   - Comando: `build`
   - JDK: 1.21

3. **Definir Nomes e Versões Dinâmicos**
   - Script: Extrai a versão do `build.gradle` e monta o nome final da aplicação.
   - **Lógica:**
     - `APP_NAME_FINAL = "aleteia-core"` (fixo)
     - `VERSION_APP = "aleteia-core-<versão_do_gradle>"`
   - **Saídas (Outputs):**
     - `VERSION_BUILD`: Versão extraída do Gradle.
     - `VERSION_APP`: Nome completo da aplicação com versão.
     - `APP_NAME_FINAL`: Nome base da aplicação.

4. **Criar Dockerfile Dinâmico**
   - Gera um `Dockerfile` usando a imagem base `amazoncorretto:21.0.9-al2023`.
   - Copia o JAR gerado (`build/libs/aleteia-core-<versão>.jar`) para `/app/<nome_app_completo>.jar`.

5. **Build da Imagem Docker**
   - Task: `Docker@2`
   - Tags: `latest` e `<versão_do_gradle>`.

6. **Push para ECR - Ambientes `develop` e `staging`**
   - **Condição:** `pEnvironment` = `develop` ou `staging`.
   - **Credenciais AWS:** `ALETEIAECR`
   - **Repositório:** `aleteia/<ambiente>/aleteia-core`
   - **Ações:** Push da tag `latest` e da tag com a versão.

7. **Push para ECR - Ambientes `demo` e `production`**
   - **Condição:** `pEnvironment` = `demo` ou `production`.
   - **Credenciais AWS:** `PlataformaOndaAzureAccess`
   - **Repositório:** `aleteia/<ambiente>/aleteia-core`
   - **Ações:** Push da tag `latest` e da tag com a versão.

> **Nota sobre Push:** O pipeline **não está atualizando a imagem** nos deployments do Kubernetes. Ele apenas constrói e envia a imagem para o ECR. O deploy é feito via `kubectl rollout restart`, que recria os pods e puxa a imagem `latest` (assumindo uma política de pull como `Always`).

---
## 7. Estágio 2: Deploy e Restart

**Nome:** `Deploy`  
**Display Name:** `Deploy e Restart`  
**Dependência:** `Build` (executa apenas se o Build for bem-sucedido)  
**Variável Recebida:** `VERSION_BUILD` do estágio anterior.

### 7.1. Job: `ExecuteDeployDEVELOP`

- **Condição:** `pEnvironment` = `develop`
- **Pool:** `ubuntu-latest`
- **Credenciais AWS:** `ALETEIAEKS`
- **Ações:**
  1. Configura o `kubeconfig` para o cluster `onda-demo-eks`.
  2. Executa `kubectl rollout restart deployment/aleteia-core-service -n aleteia`.
  3. Aguarda o sucesso do rollout.

### 7.2. Job: `ExecuteDeploySTAGING`

- **Condição:** `pEnvironment` = `staging`
- **Pool:** `ubuntu-latest`
- **Credenciais AWS:** `ALETEIAEKS`
- **Ações:**
  1. Configura o `kubeconfig` para o cluster `onda-demo-eks`.
  2. Executa `kubectl rollout restart deployment/aleteia-stg-core-service -n aleteia-stg`.
  3. Aguarda o sucesso do rollout.

### 7.3. Job: `ExecuteDeployDEMO`

- **Condição:** `pEnvironment` = `demo`
- **Credenciais AWS:** `PlataformaOndaAzureAccess`
- **Ações:**
  1. Configura o `kubeconfig` para o cluster `plataformaonda-eks`.
  2. Executa `kubectl rollout restart deployment/onda-demo-core-service -n demo`.
  3. Aguarda o sucesso do rollout.

### 7.4. Job: `ExecuteDeployPRD`

- **Condição:** `pEnvironment` = `production`
- **Credenciais AWS:** `PlataformaOndaAzureAccess`
- **Ações (Especial para Produção):**
  1. Configura o `kubeconfig` para o cluster `plataformaonda-eks`.
  2. Reinicia os deployments em **múltiplos namespaces**:
     - `arqdigital`: `onda-arqdigital-core-service`
     - `bgc`: `onda-bgc-core-service`
     - `nstech`: `onda-nstech-core-service`
     - `simpi`: `onda-simpi-core-service`
     - `vert`: `onda-vert-core-service`
     - `wizco`: `onda-wizco-core-service`
  3. Aguarda 10 segundos entre cada restart.

---

## 8. Fluxo de Trabalho (Workflow)

1. Um desenvolvedor/operador inicia o pipeline **manualmente**.
2. No momento da execução, escolhe o ambiente (`develop`, `staging`, `demo`, `production`).
3. O pipeline compila o código Java com Gradle.
4. O pipeline constrói uma imagem Docker.
5. A imagem é enviada para o Amazon ECR, em um repositório específico do ambiente (ex: `aleteia/develop/aleteia-core`).
6. O pipeline se conecta ao cluster EKS correto.
7. O pipeline executa `kubectl rollout restart` para o(s) deployment(s) alvo, forçando os pods a recriarem e baixarem a imagem `latest`.
8. O pipeline aguarda a confirmação de que o restart foi bem-sucedido.

---

## 9. Pontos de Atenção e Melhorias

| Categoria | Descrição | Recomendação |
| :--- | :--- | :--- |
| **Estratégia de Deploy** | O pipeline usa `rollout restart`, que depende da política `imagePullPolicy: Always` para baixar a nova imagem. Não há garantia explícita da versão que está rodando. | Substituir por `kubectl set image deployment/... <container>=<imagem:versão>` para ter um controle explícito da versão em execução. |
| **Segurança** | As credenciais AWS (`ALETEIAECR`, `PlataformaOndaAzureAccess`, `ALETEIAEKS`) estão nomeadas, mas não é visível se usam Service Connections ou Variáveis de Ambiente. | Garantir que todas as credenciais estejam configuradas como **AWS Service Connection** no Azure DevOps para maior segurança. |
| **Condições Duplicadas** | As condições para Push ao ECR estão duplicadas. | Extrair a lógica de condições para uma variável de template ou usar um `${{ if }}` para incluir o passo correto. |
| **Scripts Estáticos** | No ambiente `production`, os namespaces e deployments são estáticos. | Tornar esta lista configurável via variável de pipeline para facilitar manutenção. |
| **Rollback** | Não há estratégia de rollback implementada. | Adicionar um estágio de rollback que faça o deploy de uma tag anterior conhecida. |
| **Testes** | O pipeline não possui estágio de testes automatizados. | Incluir estágios de testes unitários e de integração antes do push da imagem. |

---

## 10. Exemplo de Execução

**Entrada do Usuário:**
- Ambiente selecionado: `demo`

**Resultado:**
1. Build da aplicação.
2. Imagem enviada para: `[AWS_ACCOUNT].dkr.ecr.us-east-2.amazonaws.com/aleteia/demo/aleteia-core:latest`
3. Conexão com cluster `plataformaonda-eks`.
4. Restart do deployment `onda-demo-core-service` no namespace `demo`.
5. Pipeline bem-sucedido.