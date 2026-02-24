# Contexto

A equipe comercial identificou a necessidade de disponibilizar aos clientes do setor de alimentação informações estratégicas sobre seus concorrentes, incluindo:

Faixa de preço praticada

Fluxo médio de pessoas por dia da semana e período do dia

População e densidade demográfica dos bairros

O objetivo do desafio é estruturar um serviço de dados escalável, otimizado para armazenamento e consulta analítica.
---
Estrutura do Projeto

Organização baseada em camadas analíticas:

````
geofusion-case/
│
├── data/
│   ├── raw_bronze/                         # Camada de dados brutos (fonte original)
│   │   ├── bairros.csv
│   │   ├── concorrentes.csv
│   │   ├── eventos_de_fluxo.csv
│   │   └── populacao.json
│   │
│   ├── trusted_silver/                     # Dados tratados, limpos e tipados (Parquet)
│   │   ├── bairros.parquet
│   │   ├── concorrentes.parquet
│   │   ├── eventos.parquet
│   │   └── populacao.parquet
│   │
│   └── refined_gold/                       # Dados analíticos agregados
│       ├── concorrente_geografia.parquet
│       └── fluxo_concorrente_analitico.parquet
│
├── notebooks/                              # Desenvolvimento analítico
│   ├── 01_exploracao.ipynb                 # Análise exploratória
│   ├── 02_tratamento.ipynb                 # Limpeza e padronização
│   ├── 03_metricas.ipynb                   # Cálculo das métricas
│   └── 04_dataviz.ipynb                    # Visualizações
│
├── reports/                                # Relatórios finais do case
│
├── src/                                    # Espaço para futura evolução (pipelines, scripts)
│
├── README.md                               # Documentação do projeto
└── requirements.txt                        # Dependências do ambiente
````
------------------------

# Arquitetura de Dados

O projeto adota o modelo **Bronze / Silver / Gold**, amplamente utilizado em arquiteturas analíticas para garantir organização, rastreabilidade e escalabilidade.

- **Bronze/Raw:** ingestão e preservação do dado original  
- **Silver/Trusted:** padronização, limpeza e validação  
- **Gold/Refined:** consumo analítico e agregações  

---

## Bronze (Raw)

Camada responsável pelo armazenamento dos dados brutos.

- Dados originais  
- Sem transformação  
- Mantém o formato da fonte (CSV / JSON)  
- Garante rastreabilidade  
- Permite reprocessamento completo  

---

## Silver (Trusted)

Camada de tratamento e padronização.

- Tipagem corrigida  
- Tratamento de valores nulos  
- Padronização estrutural  
- Conversão para formato **Parquet**  
- Validação de integridade entre tabelas  

**Objetivo:** disponibilizar dados confiáveis e consistentes para análise.

---

## Gold (Refined)

Camada analítica final.

- Métricas calculadas  
- Joins entre tabelas  
- Agregações consolidadas  
- Dados prontos para consumo analítico  

Essa arquitetura permite:

- Escalabilidade  
- Organização clara das responsabilidades  
- Evolução futura para ambiente em nuvem  
---
## Qual a vantagem de separar Raw e Trusted?

Separar essas camadas permite:

- Preservação do dado original para auditoria
- Reprocessamento completo em caso de erro
- Comparação entre dado bruto e tratado
- Evolução de regras de negócio sem perda histórica

Em termos práticos, isso reduz risco operacional e aumenta confiabilidade do pipeline.

---

# Uso do DuckDB

O **DuckDB** foi utilizado como engine SQL analítica local para consultas sobre arquivos Parquet.

## Principais Vantagens

- Executa SQL diretamente sobre arquivos Parquet  
- Não requer servidor  
- Alta performance para agregações  
- Ideal para validação e análise local  

---

## Exemplo de Consulta

```python
import duckdb

con = duckdb.connect()

df = con.execute("""
    SELECT codigo_concorrente,
           dia_semana,
           periodo,
           AVG(media_fluxo) AS fluxo_medio
    FROM 'data/refined_gold/fluxo_concorrente_analitico.parquet'
    GROUP BY codigo_concorrente, dia_semana, periodo
""").df()
```
---
## Visão de Engenharia

A arquitetura foi pensada para:

- Minimizar retrabalho
- Permitir auditoria
- Facilitar manutenção
- Preparar o projeto para crescimento de volume
- Tornar a solução migrável para ambiente distribuído

---

# Respostas às Perguntas do Negócio
## Faixa de Preço dos Concorrentes

A faixa de preço foi calculada utilizando agregações:

- Preço mínimo

- Preço máximo

- Preço médio

Exemplo em Pandas:
````python
df_preco = concorrentes.groupby("codigo_concorrente").agg(
    preco_min=("preco_medio", "min"),
    preco_max=("preco_medio", "max"),
    preco_medio=("preco_medio", "mean")
).reset_index()
````
---
# Fluxo de Pessoas por Dia e Período

## A segmentação foi realizada da seguinte forma:

1- Extração do dia da semana da data

2- Classificação do período:

- Manhã (06h–12h)

- Tarde (12h–18h)

- Noite (18h–06h)

Exemplo:
````python
eventos["dia_semana"] = eventos["data_visita"].dt.day_name()
eventos["hora"] = eventos["data_visita"].dt.hour
````
Agregações realizadas:

- Média de visitas

- Máximo por dia

- Mínimo por dia
---
# População e Densidade Demográfica

A densidade foi calculada como:

densidade_demografica = populacao / area_km2

Join realizado entre:

- Concorrentes

- Bairros

- População
---
# Schema de Saída Proposto
Tabela analítica final: `fato_concorrente_analitico`
| Coluna                | Tipo   | Descrição           |
| --------------------- | ------ | ------------------- |
| codigo_concorrente    | INT    | Identificador       |
| codigo_bairro         | INT    | Bairro              |
| preco_min             | FLOAT  | Menor preço         |
| preco_max             | FLOAT  | Maior preço         |
| preco_medio           | FLOAT  | Ticket médio        |
| dia_semana            | STRING | Dia da semana       |
| periodo               | STRING | Manhã/Tarde/Noite   |
| media_fluxo           | INT    | Média de visitas    |
| max_fluxo             | INT    | Máximo de visitas   |
| min_fluxo             | INT    | Mínimo de visitas   |
| populacao             | INT    | População do bairro |
| area_km2              | FLOAT  | Área do bairro      |
| densidade_demografica | FLOAT  | Habitantes por km²  |


## Formato recomendado: Parquet

Justificativa:

- Formato colunar

- Alta compressão

- Melhor performance de leitura

- Compatível com engines distribuídas

---

# Exemplos de Queries SQL
## Faixa de Preço
```SQL
SELECT 
    codigo_concorrente,
    MIN(preco_medio) AS preco_min,
    MAX(preco_medio) AS preco_max,
    AVG(preco_medio) AS preco_medio
FROM fato_concorrente_analitico
GROUP BY codigo_concorrente;
```
## Fluxo por Dia e Período
```SQL
SELECT 
    codigo_concorrente,
    dia_semana,
    periodo,
    AVG(media_fluxo) AS fluxo_medio
FROM fato_concorrente_analitico
GROUP BY codigo_concorrente, dia_semana, periodo;
```

## Densidade Demográfica
```SQL
SELECT 
    codigo_bairro,
    populacao,
    area_km2,
    densidade_demografica
FROM fato_concorrente_analitico
ORDER BY densidade_demografica DESC;
```

----

# Uso do DuckDB

O DuckDB foi utilizado como engine SQL analítica local para consultas sobre arquivos Parquet.

Vantagens:

- Execução direta sobre Parquet

- Alta performance

- Não requer servidor

- Ideal para análise local

# Escalabilidade e Evolução
## Como a escalabilidade foi considerada

- Separação em camadas

- Possibilidade de processamento incremental

- Agregações pré-computadas

- Redução de joins em tempo de consulta

## Melhorias para produção

- Orquestração com Airflow

- Versionamento de dados

- Testes automatizados

- Monitoramento

- Particionamento por data

## Cenário de crescimento 100x

- Migração para Spark

- Armazenamento em Data Lake

- Modelo dimensional (estrela)

- Particionamento por data e concorrente