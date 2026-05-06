# Helm Chart - Onda Keycloak Auth Service

## Visão Geral

Este Helm Chart é responsável pela implantação do **Keycloak** (serviço de autenticação) na Plataforma Onda, utilizando o repositório de imagens da AWS ECR e configurações específicas para Kubernetes (probes, ingress com ALB, banco de dados externo, etc.).

---

## Pré‑requisitos

- Kubernetes cluster v1.33+
- Helm 3.x
- AWS Load Balancer Controller configurado (para o `ingress.class: alb`)
- Banco de dados PostgreSQL externo (RDS ou similar) – o chart **não** cria um banco interno (postgresql.enabled = false)
- Acesso ao ECR para as imagens do Keycloak

---

## Parâmetros Gerais

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `imagePullSecrets` | list | `[]` | Nomes de secrets para pull de imagens privadas |
| `nameOverride` | string | `''` | Substitui o nome do chart |
| `fullnameOverride` | string | `''` | Substitui o nome completo do release |

---

## Service Account

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `serviceAccount.create` | bool | `true` | Cria uma ServiceAccount |
| `serviceAccount.annotations` | object | `{}` | Anotações adicionais à ServiceAccount |
| `serviceAccount.name` | string | `'auth'` | Nome da ServiceAccount a ser usada |

---

## Anotações e Contexto de Segurança

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `podAnnotations` | object | `{}` | Anotações para os Pods |
| `podSecurityContext` | object | `{}` | Contexto de segurança do Pod (ex: `fsGroup`) |
| `securityContext` | object | `{}` | Contexto de segurança do container (ex: `runAsNonRoot`) |

---

## Autoscaling (Horizontal Pod Autoscaler)

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `autoscaling.enabled` | bool | `false` | Habilita o HPA (desabilitado por padrão) |

---

## Ingress (AWS ALB)

O ingress está configurado para o **AWS Load Balancer Controller**:

| Parâmetro | Valor |
|-----------|-------|
| `ingress.enabled` | `true` |
| `ingress.annotations` | Configuração ALB especificando subnets, scheme, certificado, group name e tags |
| `ingress.hosts` | `login.plataformaonda.com.br` → paths: `/*` → serviço `auth-keycloak:80` |
| `ingress.tls` | hosts: `login.plataformaonda.com.br` (certificado ARN fornecido) |

> ⚠️ **Importante**: As subnets e o ARN do certificado referem-se à conta AWS `682033503689` na região `us-east-2`.

---

## Configurações Globais (`global.onda`)

### Repositório de Imagens

| Parâmetro | Valor |
|-----------|-------|
| `global.onda.image.repository` | `682033503689.dkr.ecr.us-east-2.amazonaws.com` |

### Serviços Internos

| Parâmetro | Valor | Descrição |
|-----------|-------|------------|
| `global.onda.services.keycloak.url` | `${AUTH_KEYCLOAK_SERVICE_HOST}` | URL do serviço Keycloak injetada via variável de ambiente |

### Probes de Saúde (Padrão para todos os serviços)

| Probe | Habilitado | initialDelaySeconds | periodSeconds | timeoutSeconds | failureThreshold | successThreshold |
|-------|------------|---------------------|---------------|----------------|------------------|--------------------|
| `livenessProbe` | `true` | 60 | 1 | 5 | 3 | 1 |
| `readinessProbe` | `true` | 30 | 10 | 1 | 3 | 1 |
| `startupProbe` | `false` | 30 | 5 | 1 | 60 | 1 |

> As probes globais podem ser sobrescritas por configurações específicas do serviço Keycloak (veja abaixo).

---

## Parâmetros Específicos do Keycloak

### Habilitar / Desabilitar

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `keycloak.enabled` | bool | `true` | Implanta o Keycloak |

### Réplicas e Imagem

| Parâmetro | Valor |
|-----------|-------|
| `keycloak.replicaCount` | `1` |
| `keycloak.image.registry` | `682033503689.dkr.ecr.us-east-2.amazonaws.com` |
| `keycloak.image.repository` | `onda/keycloak` |
| `keycloak.image.tag` | `23.0.6` |
| `keycloak.image.pullPolicy` | `IfNotPresent` |

### Credenciais de Administrador (⚠️ Sensível)

| Parâmetro | Valor |
|-----------|-------|
| `keycloak.auth.adminUser` | `plataformaondakc` |
| `keycloak.auth.adminPassword` | `SENHA_POSTGRES` |

### TLS e Proxy

| Parâmetro | Valor | Descrição |
|-----------|-------|------------|
| `keycloak.tls.enabled` | `false` | TLS não configurado internamente (o ALB já termina o TLS) |
| `keycloak.production` | `true` | Modo produção (recomendado) |
| `keycloak.proxy` | `edge` | Proxy edge (terminação TLS no load balancer) |

### Recursos (Resource Limits/Requests)

| Parâmetro | Tipo | Padrão |
|-----------|------|--------|
| `keycloak.resources.limits` | object | `{}` |
| `keycloak.resources.requests` | object | `{}` |

> Defina limites e requisições de CPU/memória conforme a necessidade do ambiente.

### Probes de Saúde (Específicas do Keycloak)

| Probe | Habilitado | initialDelaySeconds | periodSeconds | timeoutSeconds | failureThreshold | successThreshold |
|-------|------------|---------------------|---------------|----------------|------------------|--------------------|
| `livenessProbe` | `true` | 300 | 1 | 5 | 3 | 1 |
| `readinessProbe` | `true` | 30 | 10 | 1 | 3 | 1 |
| `startupProbe` | `false` | 30 | 5 | 1 | 60 | 1 |

> Os valores do livenessProbe (`initialDelaySeconds: 300`) são maiores que os padrões globais, adequados para a inicialização mais lenta do Keycloak.

### Init Containers

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|------------|
| `keycloak.initContainers` | list | `[]` | Containers de inicialização opcionais |

### Service

| Parâmetro | Valor | Descrição |
|-----------|-------|------------|
| `keycloak.service.type` | `NodePort` | Tipo do serviço Kubernetes (exposto internamente como NodePort) |

### Autoscaling (Keycloak)

| Parâmetro | Valor |
|-----------|-------|
| `keycloak.autoscaling.enabled` | `false` |
| `keycloak.autoscaling.minReplicas` | `1` |
| `keycloak.autoscaling.maxReplicas` | `1` |
| `keycloak.autoscaling.targetCPU` | `''` |
| `keycloak.autoscaling.targetMemory` | `''` |

### Banco de Dados

Por padrão, o banco de dados interno do Keycloak está desabilitado (`postgresql.enabled: false`). Utiliza-se um banco externo.

| Parâmetro | Valor |
|-----------|-------|
| `keycloak.externalDatabase.host` | `plataformaonda.cn8o6aau4xqf.us-east-2.rds.amazonaws.com` |
| `keycloak.externalDatabase.port` | `5432` |
| `keycloak.externalDatabase.user` | `plataformaondakeycloak` |
| `keycloak.externalDatabase.password` | `SENHA_POSTGRES` |
| `keycloak.externalDatabase.database` | `onda_auth` |

### Cache

| Parâmetro | Valor | Descrição |
|-----------|-------|------------|
| `keycloak.cache.enabled` | `false` | Cache desabilitado (configuração básica) |

### Logging

| Parâmetro | Valor |
|-----------|-------|
| `keycloak.logging.output` | `default` |
| `keycloak.logging.level` | `INFO` |

---

## Exemplo de Instalação

```bash
# Instalar/Atualizar release
helm upgrade --install auth-keycloak -f values.yaml \
  --namespace auth \
  --create-namespace
```
