# Documentação do Ambiente AWS – Plataforma ONDA com EKS

## 1. Visão Geral

Este documento descreve a arquitetura de infraestrutura do ambiente **Plataforma ONDA** na AWS, utilizando **Amazon EKS** (Elastic Kubernetes Service) como plataforma de orquestração de containers. O ambiente é hospedado na região **us-east-2** (Ohio) e segue as boas práticas de segurança, alta disponibilidade (parcial) e separação de camadas (pública/privada).

## 2. Componentes da Arquitetura

### 2.1. Região e Rede

| Componente | Configuração | Descrição |
| :--- | :--- | :--- |
| **Região** | `us-east-2` (Ohio) | Todos os recursos são provisionados nesta região. |
| **VPC** | `vpc-aleteia` (bloco: 10.0.0.0/16) | Rede virtual isolada que abriga os recursos do MVP. |
| **Zonas de Disponibilidade** | `us-east-2a`, `us-east-2b` (pelo menos uma AZ para o RDS) | Utilizada para alta disponibilidade dos componentes de rede. |

### 2.2. Sub-redes

| Tipo | CIDR (exemplo) | Associada a | Destino |
| :--- | :--- | :--- | :--- |
| **Public subnet** | `10.0.1.0/24` | Internet Gateway | Recursos que precisam de acesso direto à internet (ex: NAT Gateway, balanceadores públicos). |
| **Private subnet** | `10.0.2.0/24` | NAT Gateway (acesso à internet para updates) | Onde residem os nós do EKS, o banco de dados RDS e demais recursos internos. |

> *Obs: Os blocos CIDR são exemplos; podem ser ajustados conforme necessidade.*

### 2.3. Segurança – Security Groups

A arquitetura define três security groups principais:

| Security Group | Regras de entrada (inbound) | Regras de saída (outbound) | Destino |
| :--- | :--- | :--- | :--- |
| **sg-eks-cluster** | - Porta 443 (HTTPS) de qualquer lugar (0.0.0.0/0) <br>- Porta 10250 (kubelet) da VPC | Tudo liberado (0.0.0.0/0) | Controla o tráfego para o plano de controle do EKS e para os nós. |
| **sg-eks-nodes** | - Porta 443 do `sg-eks-cluster` <br>- Porta 10250 do `sg-eks-cluster` <br>- Porta 30000-32767 (NodePort) do balanceador | Tudo liberado (0.0.0.0/0) | Aplica-se aos worker nodes do EKS. |
| **sg-rds** | - Porta 5432 (PostgreSQL) do `sg-eks-nodes` <br>- Porta 5432 de qualquer IP autorizado (ex: jump-host) | Tudo liberado (0.0.0.0/0) | Protege a instância RDS, permitindo acesso apenas pelo EKS e administradores. |

### 2.4. Computação – EKS e Auto Scaling Group

| Componente | Descrição |
| :--- | :--- |
| **Cluster EKS** | Cluster Kubernetes gerenciado. O plano de controle é executado pela AWS em alta disponibilidade. |
| **Auto Scaling Group (ASG)** | Gerencia os worker nodes do EKS (EC2). Configurado para usar a **private subnet**, sem IP público. |
| **Tipo de instância** (sugestão) | `t3.medium` ou `t3.large`, dependendo da carga. |
| **Capacidade mín./máx.** | Mín: 2, Máx: 6, Desejado: 2 (distribuídos entre as AZs). |

### 2.5. Banco de Dados – RDS PostgreSQL

| Configuração | Valor |
| :--- | :--- |
| **Engine** | PostgreSQL 14.x |
| **Classe da instância** | `db.t3.micro` (MVP) ou `db.t3.small` |
| **Disponibilidade** | Single A-Z (`us-east-2a`) – adequado para MVP; para produção recomenda-se Multi-AZ. |
| **Armazenamento** | 20 GB GP2 (pode ser aumentado) |
| **Subnet group** | Private subnet group (apenas sub-redes privadas) |
| **Security group** | `sg-rds` (conforme descrito) |
| **Backup** | Snapshot automático diário, retenção de 7 dias. |

### 2.6. Armazenamento de Objetos – S3

| Bucket | Propósito |
| :--- | :--- |
| `aleteia-artifacts-<env>` | Armazenar artefatos de build (jar, docker context), logs, backups de configuração. |
| `aleteia-static-<env>` | (Opcional) Arquivos estáticos do frontend ou imagens públicas. |

- **Privacidade:** Privado por padrão, acessível apenas pela aplicação via IAM roles (ex: pelo EKS através de service accounts).

### 2.7. Distribuição e SSL/TLS – CloudFront

| Configuração | Valor |
| :--- | :--- |
| **Origem** | Pode ser configurada com duas origens: 1) S3 (conteúdo estático), 2) Application Load Balancer (ALB) do EKS para API dinâmica. |
| **Behaviors** | `/api/*` → ALB, `/*` → S3. |
| **SSL/TLS** | Certificado ACM (Amazon Certificate Manager) para domínio personalizado (ex: `app.aleteia.com`). |
| **Acesso** | Restrito via HTTPS apenas. |
| **CloudFront** também pode ser usado **apenas como CDN** do frontend S3, enquanto a API é acessada diretamente via ALB (com SSL). Para simplificar o MVP, a imagem indica CloudFront provavelmente na frente de toda a aplicação. |

### 2.8. Balanceamento de Carga e SSL/TLS

| Componente | Descrição |
| :--- | :--- |
| **Application Load Balancer (ALB)** | Criado pelo **AWS Load Balancer Controller** dentro do EKS. Escuta na porta 443 (HTTPS). |
| **Certificado SSL/TLS** | Certificado ACM provisionado na região `us-east-2` (ou global para CloudFront). O ALB utiliza este certificado. |
| **Acesso público** | O ALB é do tipo **internet-facing**, associado à **public subnet**. O tráfego vindo da internet (0.0.0.0/0) na porta 443 é permitido. |
| **Target group** | Aponta para os pods da aplicação `aleteia` (NodePort ou IP). O Security Group do ALB libera entrada do mundo. |

### 2.9. Aplicação – Aleteia (no EKS)

| Componente | Descrição |
| :--- | :--- |
| **Deployment** | `aleteia-core` (Java/Spring Boot) rodando no cluster EKS. |
| **Service** | Do tipo `ClusterIP`, exposto internamente para o ALB (ingress). |
| **ConfigMap / Secrets** | Variáveis de ambiente: conexão com RDS (host, senha), buckets S3, etc. |
| **Horizontal Pod Autoscaler (HPA)** | Escala automaticamente réplicas com base em CPU/memória. |

## 3. Fluxo de Requisição (Exemplo)

1. **Usuário** acessa `https://app.aleteia.com`.
2. **CloudFront** consulta o comportamento da rota:
   - Se for arquivo estático (`.js`, `.css`, `.png`), busca no bucket S3.
   - Se for `/api/*`, encaminha para o **ALB** (origem configurada).
3. **ALB** (com SSL/TLS) recebe a requisição e a envia para o **target group** que contém os pods da aplicação no EKS.
4. O pod do EKS processa a requisição, consulta o **RDS PostgreSQL** (via sg-rds) e, se necessário, grava/obtém objetos do **S3**.
5. A resposta retorna pelo mesmo caminho até o usuário.

## 4. Considerações de Segurança

| Aspecto | Implementação |
| :--- | :--- |
| **Isolamento de rede** | Nós do EKS e RDS em sub-redes privadas, sem IP público. |
| **Acesso SSH** | Não permitido diretamente; usa-se **Systems Manager Session Manager** ou um bastion na subnet pública. |
| **IAM Roles** | Uso de **IRSA** (IAM Roles for Service Accounts) no EKS para acessar S3 e outros serviços sem chaves estáticas. |
| **Encryption** | RDS e S3 com criptografia em repouso (AWS KMS). |
| **WAF** | (Recomendado) Associar CloudFront ou ALB a um Web Application Firewall. |

## 5. Resumo da Arquitetura (MVP)

```mermaid
graph TB
    Internet[Internet] --> CloudFront[CloudFront CDN + SSL]
    CloudFront --> ALB[Application Load Balancer (SSL)]
    CloudFront --> S3_Static[S3 - Static Files]
    ALB --> EKS[EKS Cluster - Private Subnet]
    EKS --> RDS[(RDS PostgreSQL single-AZ)]
    EKS --> S3_Artifacts[S3 - Artifacts/Logs]
    subgraph AWS Cloud
        ALB
        CloudFront
        S3_Static
        EKS
        RDS
        S3_Artifacts
    end
```
