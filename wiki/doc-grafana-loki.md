# Documentação da Interface de Consultas do Loki no Grafana

## 1. Introdução

Esta documentação descreve os principais componentes da interface de consultas do **Loki** no Grafana. O Loki é um sistema de agregação de logs horizontalmente escalável, e sua integração com o Grafana permite explorar, filtrar e visualizar logs de forma eficiente.

## 2. Estrutura Geral da Interface

### 2.1. Seleção da Fonte de Dados

- **Indicador**: `loki`
- **Função**: Permite escolher a fonte de dados Loki configurada no Grafana. É possível alternar entre diferentes instâncias ou tenants.

### 2.2. Aba "Queries"

Área principal onde as consultas LogQL são construídas. Inclui:

- **Kick start your query**: Botão ou link que oferece modelos iniciais de consulta (ex.: `{namespace="default"} |="error"`).
- **Label browser**: Ferramenta visual para navegar pelos labels disponíveis no Loki e construir filtros interativamente.
- **Explain query**: Exibe explicação detalhada da consulta LogQL atual, mostrando como cada parte será interpretada pelo Loki.
- **Label filters**: Seção para adicionar filtros baseados em labels no formato `{label_name="value"}`.
  - `Select label`: Menu suspenso com todos os labels descobertos.
  - `Select value`: Valores possíveis para o label selecionado.
- **Line contains**: Campo de texto para filtrar linhas que contenham determinada substring (equivalente ao operador `|= "texto"`).
- **Operations**: Lista de operações/transformadores que podem ser aplicados aos logs (ex.: `json`, `regexp`, `unpack`, `line_format`).
- **Text to find**: Similar ao "Line contains", provavelmente para busca livre ou expressão regular.
- **Options**: Configurações adicionais da consulta (limite de linhas, direção, intervalo de tempo).
- **Type: Range**: Define o tipo de consulta – `Range` (consulta por intervalo de tempo, padrão para logs) ou `Instant` (apenas um ponto no tempo).
- **Add query**: Permite adicionar outra consulta na mesma visualização (útil para comparar resultados lado a lado).
- **Query inspector**: Ferramenta para inspecionar a requisição enviada ao Loki, os cabeçalhos, o tempo de resposta e os dados brutos.

## 3. Fluxo de Trabalho Típico para uma Consulta no Loki

1. **Selecione a fonte de dados `loki`**.
2. **Defina o intervalo de tempo** (no topo do Explore) – por exemplo, últimas 1 hora.
3. **Construa a consulta**:
   - Use o `Label browser` para escolher labels iniciais, ex.: `{namespace="kube-system"}`.
   - Adicione um filtro de linha (`Line contains`) para buscar uma palavra-chave, ex.: `error`.
   - Se necessário, aplique uma `Operation` como `json` para extrair campos estruturados.
4. **Clique em `Run query`** (ou pressione Enter).
5. **Inspecione os resultados** – se houver dados, eles aparecerão em uma lista colorida por label.
6. **Se não houver dados**, utilize o `Query inspector` para verificar a resposta do Loki.

## 4. Explicação Detalhada dos Componentes

### 4.1. Label Browser

- Permite navegar pelos labels disponíveis em todas as séries de logs.
- Ajuda a construir filtros sem ter que digitar manualmente a sintaxe LogQL.
- Exibe a contagem de séries (streams) para cada combinação de labels.

### 4.2. Label Filters

- Equivalente à parte do seletor de streams em LogQL: `{label="value"}`.
- Multiplos filtros podem ser adicionados, todos combinados com `AND`.
- Exemplo: `{namespace="default", pod=~"frontend-.*"}`.

### 4.3. Line Contains / Text to find

- **Line contains**: Gera automaticamente o filtro `|= "texto"` (contém, case-sensitive).
- Para expressões regulares, pode-se usar o operador `|~ "regex"` (não mostrado na imagem, mas suportado).
- **Text to find** pode ser um campo alternativo para busca livre ou expressão complexa.

### 4.4. Operations

Transformações aplicadas ao conteúdo da linha de log antes da exibição. Exemplos comuns:

| Operação | Descrição | Exemplo de uso |
|----------|-----------|----------------|
| `json`   | Extrai campos de um log JSON em colunas separadas | `\| json` |
| `regexp` | Extrai grupos nomeados usando expressão regular | `\| regexp "(?P<ip>\d+\.\d+\.\d+\.\d+)"` |
| `unpack` | Desempacota dados de logs do sistema (ex.: criados por pipelines) | `\| unpack` |
| `line_format` | Reformata a linha usando sintaxe de template | `\| line_format "{{.message}}"` |
| `label_format` | Renomeia ou adiciona labels com base em dados extraídos | `\| label_format host="{{.hostname}}"` |

### 4.5. Type: Range

- **Range query**: Retorna logs dentro de um intervalo de tempo. É o tipo mais usado.
- **Instant query**: Retorna apenas um único ponto no tempo (útil para métricas derivadas de logs, como contagens agregadas).

### 4.6. Query Inspector

Ferramenta indispensável para depuração. Fornece:

- **Request**: O URL e o corpo da requisição enviada ao Loki.
- **Response**: A resposta bruta (JSON) retornada pelo Loki.
- **Stats**: Estatísticas de execução (bytes processados, linhas lidas, tempo de execução).
- **Error**: Em caso de erro, a mensagem de retorno.

## 5. Práticas Recomendadas

- **Use filtros de label primeiro** – eles são muito mais eficientes do que filtrar por linha.
- **Prefira `|= "palavra"` a `|~ ".*palavra.*"`** – o primeiro é mais rápido.
- **Combine operações** – ex.: `{namespace="prod"} | json | line_format "{{.level}} {{.message}}"`.
- **Salve consultas frequentes** – utilize os favoritos do Explore ou crie dashboards com painéis de logs.
- **Monitore a performance** – use `Query inspector` para ver o custo em bytes processados.

## 7. Conclusão

A interface de consultas do Loki no Grafana é rica e oferece desde navegação visual por labels até transformações avançadas com operações. Para maior produtividade, recomenda-se praticar a escrita de consultas LogQL diretamente, combinada com o uso do `Label browser` e `Query inspector` para depuração.

---

