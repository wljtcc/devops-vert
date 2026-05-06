# Documentação de Alertas no Grafana com Integração ao Microsoft Teams

## 1. Visão Geral

Esta documentação descreve a configuração e o gerenciamento de **alertas no Grafana**, com foco no envio de notificações para um **canal do Microsoft Teams**. As informações são baseadas na tela de **Alert rules** do Grafana.

A interface exibe regras de alerta agrupadas, seus estados (paused, normal, firing, pending), saúde (health) e opções de exportação. Todas as regras estão no estado **Paused** (pausadas), mas com saúde **Ok** e classificação **Normal** (ou seja, não estão em firing, mas estão desabilitadas temporariamente).

## 2. Estrutura da Página "Alert rules"

### 2.1. Filtros e Organização

| Filtro/Campo          | Descrição                                                                 |
|-----------------------|---------------------------------------------------------------------------|
| **Search by data sources** | Permite filtrar regras por fonte de dados (ex: Prometheus, Loki, etc.)   |
| **All data sources**       | Seleciona todas as fontes de dados (padrão)                              |
| **Dashboard**              | Filtra regras associadas a um dashboard específico                       |
| **State**                  | Estados possíveis: `Firing` (disparando), `Normal` (ok), `Pending` (pendente) |
| **Rule type**              | `Alert` (regra de alerta) ou `Recording` (regra de gravação de métricas) |
| **Health**                 | `Ok`, `No Data`, `Error` – indica a saúde da execução da regra           |
| **View as**                | `Grouped` (agrupado por pasta/namespace), `List` (lista simples), `State` (por estado) |

### 2.2. Indicadores de Estado

São listadas 4 regras:

| Rule Name                     | State   | Health | Summary | Next evaluation | Actions |
|-------------------------------|---------|--------|---------|-----------------|---------|
| BGC - Status Analise          | Paused  | ok     | (vazio) |                 | More →  |
| WIZCO - Status Analise        | Paused  | ok     | (vazio) |                 | More →  |
| SIMPI - Status Analise        | Paused  | ok     | (vazio) |                 | More →  |
| VERT - Status Analise         | Paused  | ok     | (vazio) |                 | More →  |

- **State = Paused**: A regra está desabilitada temporariamente. Não será avaliada e não pode disparar alertas.
- **Health = Ok**: A expressão da regra é válida e a avaliação (se estivesse ativa) não produziria erro.
- **Normal**: Agrupamento inferior indica "4 normal" – ou seja, todas as 4 regras estão no estado "Normal" (não firing/pending), mas com pause individual.

> **Importante**: Uma regra pausada não envia notificações, independentemente da condição dos dados.

## 3. Configuração de Alertas para o Microsoft Teams

Para que os alertas do Grafana sejam enviados a um canal do Teams, é necessário criar um **ponto de contato** (contact point) do tipo **Power Automate** e associá-lo a uma política de notificação.

### 3.1. Obter URL do Webhook do Teams

1. No Microsoft Teams, acesse o canal desejado.
2. Clique nos `...` (Mais opções) → **Conectores**.
3. Procure por **Power Automate** e adicione.
4. Configure um nome (ex: "Grafana Alerts") e opcionalmente um avatar.
5. Copie a URL gerada (ex: `https://yourdomain.webhook.office.com/...`).

### 3.2. Criar Contact Point no Grafana

1. No Grafana, vá para **Alerting → Contact points**.
2. Clique em **+ Add contact point**.
3. **Name**: `Teams - Status Analise`
4. **Integration**: escolha **Webhook**.
5. **URL**: cole a URL criada pelo Power Automate do Teams.
6. **HTTP Method**: `POST`
7. **Message format**: `JSON` (padrão)
8. **Optional settings**:
   - Defina cabeçalhos se necessário (geralmente vazio).
   - Configure **Text** para formatar a mensagem (pode usar template Go).
9. Clique em **Test** para enviar uma mensagem de teste ao Teams.
10. Salve o contact point.

> **Dica**: Você também pode usar a integração nativa **Microsoft Teams** (disponível em versões recentes do Grafana Enterprise/Cloud) – ela oferece melhor formatação adaptada.

### 3.3. Associar o Contact Point a uma Política de Notificação

1. Em **Alerting → Notification policies**.
2. Na política raiz (ou crie uma nova política específica).
3. Em **Contact point**, selecione o contact point criado (`Teams - Status Analise`).
4. Opcional: adicione **matching labels** para direcionar apenas alertas com certas labels (ex: `status="analise"`).
5. Configure **timing options** (tempo de espera para repetir, silenciamento, etc.).
6. Salve a política.

## 4. Fluxo Completo de um Alerta para Teams

1. **Regra ativa** (não pausada) com condição: ex: `BGC - Status Analise` → consulta métrica > limite.
2. **Grafana avalia** periodicamente (ex: a cada 60s).
3. Se condição verdadeira por **for how long** (duração configurada), o estado muda `Pending` → `Firing`.
4. O **Alertmanager** do Grafana recebe o alerta e aplica as políticas de notificação.
5. A política encaminha o alerta ao **contact point** `Teams - Status Analise`.
6. O Grafana envia um POST para o webhook do Teams com payload JSON.
7. O Teams exibe um card com título, resumo, labels, valores e link de gráfico.

### 4.1. Exemplo de Payload (simplificado)

```json
{
  "title": "[Firing] BGC - Status Analise",
  "text": "Valor atual: 95% (limite: 80%)\nLabels: namespace=bgc, severity=critical",
  "sections": [{
    "title": "Grafana Alert",
    "facts": [
      {"name": "Rule", "value": "BGC - Status Analise"},
      {"name": "State", "value": "Firing"},
      {"name": "Dashboard URL", "value": "https://grafana/d/abc"}
    ]
  }]
}
```

## 5. Gerenciamento Prático das Regras (Baseado na Imagem)

| Ação | Como executar |
|------|---------------|
| Pausar uma regra | Na linha da regra → `More` → **Pause** |
| Reativar (Unpause) | Na linha da regra → `More` → **Unpause** |
| Editar a condição/expressão | `More` → **Edit** |
| Silenciar alertas sem pausar a regra | Usar **Silences** (Alerting → Silences) |
| Testar o Teams webhook | No contact point → **Test** → escolha uma regra fake ou envio manual |
| Ver histórico de alertas disparados | **Alerting → Alert list** (filtrar por nome da regra) |

## 7. Troubleshooting Comum

| Problema | Possível causa e solução |
|----------|--------------------------|
| Nenhum alerta chega ao Teams | Verifique se a regra está **Paused** (como na imagem). Solução: **Unpause**. |
| Teste do webhook falha (erro HTTP 4xx/5xx) | URL inválida ou conector do Teams expirado (recrie o webhook). Teste com `curl` ou Postman. |
| Alerta dispara mas mensagem não formatada | Configure **Message format** como `JSON` e ajuste o template. Para Teams, prefira o conector nativo. |
| Regra com `Health = Error` | A expressão da regra tem erro de sintaxe ou referência a métrica inexistente. Edite e corrija. |
| Estado `No Data` na saúde | A consulta da regra retornou "no data" durante avaliação. Ajuste o comportamento em `Alert rule` → `No data evaluation`. |

## 8. Recomendações de Boas Práticas

- **Nunca deixe regras de produção pausadas sem motivo**. Use pausa apenas durante manutenção ou testes.
- **Utilize labels consistentes** (ex: `service="status-analise"`) para rotear alertas para os times corretos.
- **Mantenha um contact point de fallback** (ex: e-mail) caso o Teams esteja indisponível.
- **Exporte as regras regularmente** (`Export rules`) e versione-as em repositório Git.
- **Defina tempo de avaliação adequado** (ex: `for: 5m`) para evitar falsos positivos.
- **Monitore a saúde das regras** usando o próprio Grafana (ex: métrica `grafana_alerting_rule_evaluations_total`).

## 9. Conclusão

A tela "Alert rules" do Grafana é o centro de gerenciamento de todas as condições de alerta. Na imagem analisada, todas as quatro regras estão **Paused** (pausadas) e em estado **Normal** – ou seja, estão desabilitadas, não enviarão notificações ao Teams mesmo que a integração esteja totalmente configurada.

Para ativar o fluxo completo, é necessário:

1. **Unpause** as regras desejadas.
2. Garantir que o contact point do Teams exista e esteja correto.
3. Associar o contact point a uma política de notificação (padrão ou customizada).
