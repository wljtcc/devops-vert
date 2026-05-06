# Documentação do Pipeline Azure DevOps - Deploy SPA

## 1. Visão Geral

Este pipeline é responsável pelo **deploy manual de aplicações SPA (Single Page Application)** em ambiente de produção. Ele é acionado exclusivamente de forma manual e permite selecionar uma **tag de release** (ex.: `v1.9.1`) e os **clientes** que receberão o deploy (ou todos os clientes de uma vez). O pipeline utiliza um template externo (`templates/manual-build-master.yaml`) para executar as etapas de build, upload para S3 e invalidação de CloudFront.

**Características principais:**
- Execução **100% manual** (sem triggers automáticos)
- Deploy baseado em **tags de release** (não na branch `main` ou `master`)
- Suporte a **dois modos**: `Todos os clientes` ou `Apenas selecionados`
- Clientes atualmente disponíveis: 9 (com um cliente comentado – IONIQ)

## 2. Gatilhos (Triggers)

| Configuração | Valor | Descrição |
| :--- | :--- | :--- |
| `trigger` | `none` | Nenhum commit ou push inicia o pipeline. |
| `pr` | `none` | Nenhum Pull Request inicia o pipeline. |
| `name` | `'Deploy Produção - Tag ${{ parameters.gitTag }} - ${{ parameters.deployMode }}'` | Nome dinâmico da execução, mostrando a tag e o modo escolhido. |

## 3. Parâmetros de Execução

No momento da execução manual, o operador deve informar os seguintes parâmetros:

### 3.1. `gitTag` (string)
- **Display Name:** `Tag de Release para Deploy (Produção)`
- **Valores disponíveis:** lista fixa de versões (de `v1.0.0` até `v1.9.1`)
- **Valor padrão:** `v1.9.1`
- **Descrição:** Tag do repositório que será usada para fazer checkout do código fonte e realizar o build.

### 3.2. `deployMode` (string)
- **Display Name:** `Modo de Deploy`
- **Valores:**
  - `Todos os clientes` – faz deploy para **todos** os clientes listados.
  - `Apenas selecionados` – faz deploy **somente** para os clientes cujo checkbox for marcado.
- **Valor padrão:** `Apenas selecionados`

### 3.3. Clientes (boolean)

Para o modo `Apenas selecionados`, o operador pode escolher um ou mais dos seguintes clientes:

| Parâmetro | Display Name | Padrão | Observação |
| :--- | :--- | :--- | :--- |
| `deployONDAARQDIGITAL` | ONDA-ARQDIGITAL | `false` | |
| `deployBGC` | BGC | `false` | |
| `deployMONTECASSINO` | MONTECASSINO | `false` | |
| `deploySIMPI` | SIMPI | `false` | |
| `deployVOXDATATI` | VOXDATATI | `false` | |
| `deployWIZCO` | WIZCO | `false` | |
| `deployVERT` | VERT | `false` | |
| `deployNSTECH` | NSTECH | `false` | |
| `deployDEMO` | DEMO | `false` | |
| ~~`deployIONIQ`~~ | ~~IONIQ~~ | (comentado) | Cliente atualmente desabilitado no código. |

> **Nota:** O cliente IONIQ está comentado no YAML, portanto não aparece na interface de execução.

## 4. Estágios

O pipeline possui **apenas um estágio**, que utiliza um template externo. A definição é:

```yaml
stages:
  - template: templates/manual-build-master.yaml
    parameters:
      gitTag: ${{ parameters.gitTag }}
      deployMode: ${{ parameters.deployMode }}
      allClients: [ ...lista de todos os clientes... ]
      selectedClients: [ ...mapeamento booleano... ]
```

## 4.1. Template `manual-build-master.yaml`

O template é responsável por:

- Fazer o **checkout** do código correspondente à tag informada (`gitTag`).
- Executar o **build** da aplicação SPA (geralmente `npm install && npm run build`).
- Realizar o **upload** dos artefatos estáticos para o bucket S3 específico de cada cliente.
- **Invalidar o cache** do CloudFront de cada cliente atendido.

**Comportamento conforme `deployMode`:**

- `Todos os clientes`: itera sobre a lista `allClients` e executa as etapas para cada um.
- `Apenas selecionados`: filtra a lista `allClients` usando os valores booleanos de `selectedClients` (apenas aqueles com `true`).

**Estrutura de cada cliente no parâmetro `allClients`:**

| Campo        | Descrição                                                   | Exemplo                     |
| :----------- | :---------------------------------------------------------- | :-------------------------- |
| `setup`      | Nome lógico do cliente (usado em diretórios, variáveis)     | `onda-arqdigital`           |
| `display`    | Nome amigável para logs e exibição                          | `ONDA-ARQDIGITAL`           |
| `s3Bucket`   | Nome do bucket S3 onde os arquivos serão enviados           | `arqdigital-fe-storage`     |
| `cloudfrontId` | ID da distribuição CloudFront para invalidação de cache   | `EQ5OEX60DL0I1`             |

> **Atenção:** O conteúdo exato do template não foi fornecido, mas esta documentação reflete o comportamento esperado baseado nas práticas comuns de deploy de SPA no Azure DevOps com AWS.

---

## 5. Fluxo de Trabalho (Workflow)

### Passo a passo detalhado:

1. Operador acessa o Azure DevOps e executa o pipeline manualmente.
2. Operador seleciona:
   - A tag de release desejada (ex.: `v1.9.1`)
   - O modo de deploy (`Todos os clientes` ou `Apenas selecionados`)
   - Se `Apenas selecionados`, marca os clientes alvo.
3. O pipeline inicia e o template faz checkout do código da tag escolhida.
4. O build da aplicação é executado (geralmente gerando uma pasta `dist/` ou `build/`).
5. Para cada cliente no conjunto final (todos ou os selecionados):
   - Os arquivos estáticos são enviados para o bucket S3 correspondente (normalmente com `aws s3 sync`).
   - Uma invalidação de cache é disparada no CloudFront associado (para garantir que os novos arquivos sejam entregues imediatamente).
6. O pipeline reporta sucesso ou falha para cada cliente.

---

## 6. Exemplo de Execução

**Entrada do usuário:**
- `gitTag`: `v1.9.0`
- `deployMode`: `Apenas selecionados`
- Clientes marcados: `ONDA-ARQDIGITAL`, `SIMPI`, `DEMO`

**Resultado esperado:**
1. Checkout do código na tag `v1.9.0`.
2. Build da aplicação.
3. Deploy para os buckets:
   - `arqdigital-fe-storage`
   - `simpi-fe-storage`
   - `demo-fe-storage`
4. Invalidação dos CloudFronts: `EQ5OEX60DL0I1`, `E34YOVHF59TUNG`, `E350K5AJOP4MOR`.
5. Os clientes `BGC`, `MONTECASSINO`, `VOXDATATI`, `WIZCO`, `VERT`, `NSTECH` **não** recebem o deploy.

---

## 7. Notas e Boas Práticas

| Item              | Descrição                                                                 | Recomendação                                                                                              |
| :---------------- | :------------------------------------------------------------------------ | :-------------------------------------------------------------------------------------------------------- |
| **Tags de release** | A lista de tags é fixa no código YAML.                                   | Mantenha a lista atualizada sempre que uma nova tag for criada, ou use um parâmetro dinâmico que busque as tags diretamente do repositório (ex.: `git tag -l`). |
| **Cliente IONIQ**   | Está comentado no YAML.                                                  | Se o cliente voltar a ser utilizado, descomente as linhas correspondentes nos parâmetros e na lista `allClients`. |
| **Template externo** | O template `manual-build-master.yaml` não está visível neste documento. | Certifique-se de que o template esteja versionado no mesmo repositório, no caminho `templates/`.         |
| **Rollback**        | Não há mecanismo de rollback automático.                                 | Para reverter, execute o pipeline novamente com a tag anterior e invalide os CloudFronts.                |
| **Segurança**       | As credenciais AWS (S3, CloudFront) devem estar configuradas como AWS Service Connection no Azure DevOps. | Jamais armazene chaves de acesso diretamente no pipeline.                                                |
| **Testes**          | O pipeline não executa testes automatizados antes do deploy.             | Considere adicionar um estágio opcional de testes (ex.: `npm test`) antes do upload para S3.             |

---

## 8. Possíveis Melhorias

- **Obter tags automaticamente:** usar um script para listar tags do repositório e tornar o parâmetro `gitTag` dinâmico.
- **Pré-visualização do deploy:** adicionar uma etapa que exiba os clientes que serão afetados antes de prosseguir.
- **Aprovação manual:** incluir um approval check antes do envio para produção.
- **Centralizar clientes em um arquivo JSON:** evitar duplicação da lista entre `allClients` e `selectedClients`.
- **Suporte a múltiplos ambientes:** o pipeline é fixo para produção; poderia ser parametrizado para staging, demo, etc.

---

## 9. Conclusão

Este pipeline oferece um controle granular e seguro para deploys de front-end em produção, permitindo que o time de operações selecione exatamente qual versão (tag) e quais clientes serão atualizados. O uso de template facilita a manutenção das etapas comuns. A principal limitação é a lista de tags fixa, que requer atualização manual no arquivo YAML sempre que uma nova versão é lançada.

---
