# Helm Chart - Onda Platform

## Visão Geral

Este Helm Chart é responsável pela implantação dos microsserviços da plataforma **Onda** no Kubernetes.  
O arquivo `values.yaml` fornecido contém todas as configurações padrão para os componentes, incluindo ajustes de rede, banco de dados, storage, probes de saúde e ativação de serviços.

---

## Pré‑requisitos

- Kubernetes cluster (versão 1.33+)
- Helm 3.x
- AWS ECR para imagens
- AWS Load Balancer Controller configurado (para uso do `ingress.class: alb`)
- Keycloak disponível no namespace `auth`
- Acesso ao RDS PostgreSQL

---

## Parâmetros de Configuração

### Gerais

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `imagePullSecrets` | list | `[]` | Nomes de secrets para pull de imagens privadas |
| `nameOverride` | string | `""` | Substitui o nome do chart |
| `fullnameOverride` | string | `""` | Substitui o nome completo do release |

---

### Service Account

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `serviceAccount.create` | bool | `false` | Se `true`, cria uma ServiceAccount |
| `serviceAccount.annotations` | object | `{}` | Anotações para a ServiceAccount |
| `serviceAccount.name` | string | `""` | Nome da ServiceAccount a usar (se vazio e `create=true`, usa template) |

---

### Anotações e Contexto de Segurança

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `podAnnotations` | object | `{}` | Anotações para os Pods |
| `podSecurityContext` | object | `{}` | Contexto de segurança do Pod (ex: `fsGroup`) |
| `securityContext` | object | `{}` | Contexto de segurança do container (ex: `runAsNonRoot`, `capabilities`) |

---

### Autoscaling

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `autoscaling.enabled` | bool | `false` | Habilita o Horizontal Pod Autoscaler |

---

### Ingress (AWS ALB)

O Ingress está configurado para usar o **AWS Load Balancer Controller** com as seguintes anotações:

| Parâmetro | Valor |
|-----------|-------|
| `ingress.enabled` | `true` |
| `ingress.annotations` | Configuração ALB (target‑type, group name, subnets, scheme, certificado, tags) |
| `ingress.hosts` | `api-vert.plataformaonda.com.br`<br>paths: `/*` → serviço `onda-vert-gateway-service:80` |
| `ingress.tls` | hosts: `api-vert.plataformaonda.com.br` (certificado ARN fornecido) |

> ⚠️ **Atenção**: As subnets e o ARN do certificado são específicos da conta AWS. Ajuste conforme seu ambiente.

---

### Configurações Globais (`global.onda`)

#### Repositório de Imagens

| Parâmetro | Valor |
|-----------|-------|
| `global.onda.image.repository` | `682033503689.dkr.ecr.us-east-2.amazonaws.com` |

Todas as imagens dos microsserviços serão buscadas nesse registry.

#### Banco de Dados (PostgreSQL RDS)

| Parâmetro | Valor |
|-----------|-------|
| `spring.datasource.postgresql.url` | `jdbc:postgresql://.../onda_vert?currentSchema=public` |
| `spring.datasource.postgresql.username` | `plataformaonda` |
| `spring.datasource.postgresql.password` | `"Xiq4#Hiq4$Kev0&Vur2"` |

> 🔒 **Segurança**: A senha está em texto plano. Considere usar `Secret` do Kubernetes.

#### Configurações JPA / Hibernate

| Parâmetro | Valor |
|-----------|-------|
| `spring.jpa.hibernate.dialect` | `PostgreSQLDialect` |
| `spring.jpa.generateDDL` | `false` |
| `spring.jpa.hibernateDDLAuto` | `update` |
| `spring.jpa.showSQL` | `false` |

#### URLs dos Serviços Internos

| Parâmetro | Valor |
|-----------|-------|
| `services.gatewayService.url` | `${ONDA_VERT_GATEWAY_SERVICE_SERVICE_HOST}` (injetado via variável de ambiente) |
| `services.keycloak.url` | `auth-keycloak.auth` (DNS interno) |

#### Storage S3

| Parâmetro | Valor |
|-----------|-------|
| `storage.s3.endpoint` | `https://s3.amazonaws.com` |
| `storage.s3.accessKey` | `ID_KEY` |
| `storage.s3.bucket` | `vert-plataformaonda-storage` |
| `storage.s3.secretKey` | `ID_Secret` |
| `storage.s3.signingRegion` | `us-east-2` |

> 🔒 **Segurança**: As credenciais da AWS não devem ser expostas no values. Use `IRSA` ou `Secrets`.

#### Probes de Saúde (Kubernetes)

| Probe | Habilitado | initialDelaySeconds | periodSeconds | timeoutSeconds | failureThreshold | successThreshold |
|-------|------------|---------------------|---------------|----------------|------------------|--------------------|
| `livenessProbe` | `true` | 60 | 1 | 5 | 3 | 1 |
| `readinessProbe` | `true` | 30 | 10 | 1 | 3 | 1 |
| `startupProbe` | `false` | 30 | 5 | 1 | 60 | 1 |

#### Mapeamento de Hosts dos Serviços (`servicesHost`)

| Nome do serviço | Valor (nome interno no Kubernetes) |
|-----------------|--------------------------------------|
| `coreService` | `onda-vert-core-service` |
| `debitoDiretoService` | `onda-vert-debito-direto-service` |
| `checkwebService` | `onda-vert-checkweb-service` |
| `messagingService` | `onda-vert-messaging-service` |
| `messagingV2Service` | `onda-vert-messaging-v2-service` |
| `reportService` | `onda-vert-report-service` |
| `apiService` | `onda-vert-api-service` |

---

### Ativação de Serviços (Subcharts)

O chart raiz pode ativar/desativar serviços filhos:

| Serviço | Parâmetro | Padrão |
|---------|-----------|--------|
| Gateway | `gateway-service.enabled` | `true` |
| Core | `core-service.enabled` | `true` |

Outros serviços (débito direto, checkweb, messaging, report, api) **não** possuem chave explícita neste values, mas podem ser ativados via seus próprios valores ou pelo uso de dependências.

---

## Exemplo de Instalação

```bash
# Adicionar repositório (se aplicável)
helm repo add onda https://seu-registry/helm-charts

# Instalar/Atualizar release
helm upgrade --install plataforma-onda ./helm-charts-onda \
  --namespace plataforma-onda \
  --create-namespace \
  --values custom-values.yaml \
  --set global.onda.spring.datasource.postgresql.password=NEW_PASSWORD