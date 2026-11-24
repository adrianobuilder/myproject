# Engenharia de Dados | Pipeline de Produtos com DuckDB

Pipeline local de engenharia de dados que demonstra a construção de um fluxo analítico em três camadas — **Bronze, Silver e Gold** — a partir de arquivos CSV de produtos e estoque. Desenvolvido com Python, Pandas, SQL e DuckDB, o projeto cobre ingestão, rastreabilidade de origem, deduplicação por atualização mais recente, padronização de nomes e tipos e preparação de uma dimensão de produtos.

> **Resumo para portfólio:** as camadas são implementadas como etapas lógicas do fluxo e tabelas DuckDB (`bronze_z0019`, `produtos` e `dim_produtos`). O projeto é executado manualmente em notebooks e está em evolução; não representa uma plataforma de produção orquestrada.

## Visão geral

Os arquivos de entrada contêm registros de produtos com atualizações entre cargas. O pipeline preserva os registros recebidos na Bronze, aplica regras de seleção e padronização para gerar uma visão Silver e projeta os atributos necessários à dimensão Gold.

```text
CSV de origem
   |
   v
BRONZE  bronze_z0019
   Dados recebidos + nome do arquivo + horário de ingestão
   |
   v
SILVER  produtos
   Último registro por produto + nomes e tipos padronizados
   |
   v
GOLD    dim_produtos
   Dimensão de produto com os atributos selecionados para consumo
```

O armazenamento é feito em `scripts/dados_duckdb.db`, um arquivo DuckDB local.

## Camadas do pipeline

### Bronze — ingestão e rastreabilidade

Implementada em [`scripts/ingestao.ipynb`](./scripts/ingestao.ipynb). O notebook lê os CSVs delimitados por ponto e vírgula com Pandas e grava os registros no DuckDB. Antes da persistência, acrescenta:

- `nome_arquivo`: identifica o arquivo que originou cada registro;
- `data_ingestao`: registra o momento em que a carga foi executada.

A tabela de destino usada pelo restante do fluxo é `bronze_z0019`. Essa camada conserva os campos de origem para permitir rastrear a procedência dos dados e comparar diferentes ingestões.

### Silver — seleção, deduplicação e padronização

Implementada em [`scripts/enriquecimento.ipynb`](./scripts/enriquecimento.ipynb). A transformação consulta a Bronze e:

1. Filtra registros a partir da data de referência definida na consulta.
2. Numera as versões de cada produto com `ROW_NUMBER()`, particionando por `NATBR` e ordenando por `data_ingestao` decrescente.
3. Mantém a versão mais recente de cada identificador.
4. Remove colunas técnicas de ingestão do conjunto destinado ao consumo.
5. Renomeia os campos para uma nomenclatura mais amigável e converte os tipos no Pandas.
6. Persiste o resultado na tabela `produtos`.

| Origem | Campo Silver | Tipo definido no destino |
| --- | --- | --- |
| `NATBR` | `id` | `BIGINT` |
| `MAKTX` | `nm_produto` | `TEXT` |
| `WERKS` | `id_categoria` | `TEXT` |
| `MAINS` | `id_fornecedor` | `BIGINT` |
| `LABST` | `vl_preco` | `FLOAT` |

### Gold — dimensão para consumo analítico

Implementada em [`scripts/refinamento.ipynb`](./scripts/refinamento.ipynb). A etapa consulta `produtos`, seleciona os atributos usados na dimensão e cria `dim_produtos`:

| Campo Gold | Tipo |
| --- | --- |
| `id_produto` | `BIGINT` |
| `nm_produto` | `TEXT` |
| `vl_produto` | `FLOAT` |

Essa projeção demonstra a preparação de uma estrutura dimensional enxuta para consultas analíticas. A tabela atual não inclui métricas agregadas nem dimensões relacionadas de categoria ou fornecedor.

> **Semântica dos dados:** os nomes `id_categoria`, `id_fornecedor`, `vl_preco` e `vl_produto` refletem os mapeamentos implementados nos notebooks. A semântica dos campos de origem deve ser validada com o sistema fonte antes de qualquer uso de negócio.

## Decisões técnicas demonstradas

- **Formato colunar analítico local:** DuckDB armazena as camadas em um banco embutido, sem exigir um servidor de banco separado.
- **SQL para transformação:** a deduplicação usa função de janela (`ROW_NUMBER`) para selecionar a versão mais recente por produto.
- **Pandas para preparação:** leitura dos CSVs, manipulação tabular, renomeação de colunas e conversão de tipos.
- **Rastreabilidade:** metadados de arquivo e horário acompanham os registros na camada Bronze.
- **Separação por responsabilidades:** ingestão, enriquecimento e refinamento estão organizados em notebooks distintos.
- **Modelagem analítica inicial:** a saída Gold é projetada como uma dimensão de produtos com esquema explícito.

## Tecnologias

| Tecnologia | Aplicação |
| --- | --- |
| Python | Linguagem de implementação |
| Pandas | Leitura e transformação tabular |
| DuckDB | Persistência local e execução de SQL analítico |
| Jupyter | Desenvolvimento e execução interativa do fluxo |

## Estrutura do repositório

```text
.
├── landing/
│   ├── z0019_1.csv
│   └── z0019_2.csv
├── scripts/
│   ├── ingestao.ipynb
│   ├── enriquecimento.ipynb
│   └── refinamento.ipynb
└── README.md
```

O banco `scripts/dados_duckdb.db` é um artefato local usado/criado durante a execução e pode não estar disponível em um clone novo.

## Dados de entrada

Os arquivos de exemplo têm separador `;` e contêm:

| Coluna | Uso na transformação |
| --- | --- |
| `NATBR` | Identificador usado para particionar as versões do produto |
| `MAKTX` | Descrição do produto |
| `WERKS` | Mapeado para `id_categoria` |
| `MAINS` | Mapeado para `id_fornecedor` |
| `LABST` | Mapeado para `vl_preco` |

Os CSVs são amostras pequenas para demonstrar cargas com registros de produtos repetidos entre arquivos e valores atualizados.

## Como executar

### 1. Instalar dependências

Use um ambiente Python que também esteja selecionado como kernel do Jupyter:

```bash
python -m pip install pandas duckdb jupyter
```

### 2. Iniciar Jupyter no diretório esperado

Os notebooks usam caminhos relativos: leem os CSVs em `../landing/` e abrem o banco como `dados_duckdb.db`. Inicie o Jupyter a partir de `scripts/`:

```bash
cd scripts
python -m jupyter lab
```

No VS Code, abra a pasta do projeto, selecione o mesmo ambiente Python em que instalou as dependências e abra os notebooks em `scripts/`. Se o diretório de trabalho do kernel for diferente, ajuste os caminhos relativos.

### 3. Executar as etapas

Execute os notebooks nesta ordem:

1. `ingestao.ipynb`
2. `enriquecimento.ipynb`
3. `refinamento.ipynb`

No notebook de ingestão, a variável `arquivo` escolhe qual CSV será carregado. Para processar o segundo arquivo, selecione-o e execute novamente as células correspondentes de leitura e gravação.

## Consultas de exploração

As consultas a seguir podem ser executadas em uma conexão DuckDB com o banco do projeto:

```sql
SHOW TABLES;

SELECT nome_arquivo, COUNT(*) AS registros
FROM bronze_z0019
GROUP BY nome_arquivo
ORDER BY nome_arquivo;

SELECT *
FROM produtos
ORDER BY id;

SELECT *
FROM dim_produtos
ORDER BY id_produto;
```

## Estado atual e pontos de atenção

Este repositório demonstra as transformações centrais das três camadas em um ambiente de estudo. Alguns aspectos ainda precisam ser tratados antes de considerar o fluxo robusto para execução recorrente:

- A execução é manual; não há orquestrador, agendamento, monitoramento ou gestão de tentativas.
- Os `INSERT` são append-only. Reexecutar as células pode duplicar registros nas tabelas de destino; ainda não há estratégia idempotente de `MERGE`/upsert.
- A ingestão tenta renomear `bronze_produtos` para `bronze_z0019`; a célula pode falhar se a tabela de destino já existir.
- A deduplicação usa uma data fixa (`2026-11-12`) que deve se tornar um parâmetro de execução.
- Os caminhos dependem do diretório de trabalho do kernel Jupyter.
- Não há testes automatizados nem validações formais para esquema, nulos, integridade referencial ou qualidade dos dados.
- A semântica de alguns campos de origem precisa de confirmação com a fonte de dados.

## Próximas evoluções

- Tornar caminhos, arquivos e data de referência configuráveis.
- Implementar cargas idempotentes e transacionais, com estratégia explícita de atualização.
- Adicionar validações de esquema e qualidade de dados em cada camada.
- Extrair a lógica dos notebooks para módulos Python reutilizáveis.
- Criar testes automatizados para as regras de deduplicação e transformação.
- Automatizar o fluxo com um orquestrador e incorporar logs, monitoramento e tratamento de falhas.
- Formalizar o modelo dimensional e a semântica dos atributos Gold com regras de negócio.

## Sobre o projeto

Projeto pessoal de portfólio voltado à prática de fundamentos de engenharia de dados: ingestão em lote, persistência analítica, rastreabilidade, transformação SQL/Pandas, deduplicação e organização de dados em camadas Bronze, Silver e Gold.
