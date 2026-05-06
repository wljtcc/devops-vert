# Documentação do Dashboard Grafana – Kubernetes (K8S Dashboard)

## 1. Introdução

Este documento descreve o dashboard **K8S Dashboard** do Grafana, que monitora o cluster da Plataforma Onda. O dashboard faz parte da hierarquia `Home > Dashboards > Kubernetes` e tem como objetivo fornecer uma visão abrangente do estado de um cluster Kubernetes, incluindo métricas de nós, pods, namespaces, rede e workloads.

A imagem evidencia uma interface típica do Grafana, com gráficos, indicadores numéricos e listas de recursos. Foram identificados diversos painéis que serão detalhados a seguir.

## 2. Visão geral do dashboard

- **Fonte de dados principal**: Prometheus (indicado na seção `Prometheus` do painel de estatísticas de namespaces).
- **Período analisado**: Gráficos com intervalo de tempo de 25 minutos.
- **Nós selecionados**: `[All]` (todos os nós do cluster).

## 3. Descrição detalhada dos painéis

### 3.1. Node Memory Ratio

 Geralmente representa o percentual de memória utilizada em relação à capacidade total dos nós, podendo incluir também o percentual de memória solicitada (requests) ou limite (limits).

### 3.2. Node CPU Ratio

Análogo à memória, tipicamente exibe o percentual de uso real de CPU e o percentual de CPU solicitada pelos pods.

### 3.3. Nodes with Pod

Provavelmente representa o número total de pods e o número total de nós.

### 3.4. Namespace Resource Statistics

A seção apresenta uma lista com namespaces e quantidades associadas:

Este painel exibe, por namespace, quantidades de pods, uso de memória, armazenamento.

### 3.5. Network Overview

Gráfico de linha com eixo Y em **MiB/s** e eixo X no horário.  
Apresenta múltiplas curvas, indicando tráfego de rede de entrada/saída ou taxa de transferência agregada por nós e namespaces.

### 3.6. Workload (Carga de trabalho)

**Gráficos**:
- **Memory Usage **: evolução do uso de memória ao longo do tempo.
- **CPU Used Cores **: uso de núcleos de CPU no mesmo período.
- **All: Nodes CPU Breakdown**: detalhamento do uso de CPU por nó.
- **All: Node Memory Breakdown**: detalhamento do uso de memória por nó.

Esses gráficos permitem identificar picos de consumo e possíveis desbalanceamentos entre nós.

### 3.7. Total Pod e Total Nodes

O gráfico **Pod Number and nodes** exibe a relação entre número de pods e limite superior de pods por nó (capacidade máxima suportada).

## 4. Observações sobre fontes de dados

- O Prometheus é a fonte de dados primária, configurado com o Helm Charts e exposto pela URL prometheus.plataformaonda.com.br, usando um ALB Interno.

## 5. Conclusão

O dashboard **K8S Dashboard** faz o monitoramento do cluster Kubernetes, abrangendo recursos de nós, workloads e rede. A documentação aqui fornecida serve como guia para administradores e engenheiros de plataforma que desejam compreender e observar o cluster de forma mais fácil.

---