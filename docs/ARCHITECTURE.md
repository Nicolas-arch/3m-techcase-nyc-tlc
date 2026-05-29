# Arquitetura da Solução

## Visão geral

Pipeline em arquitetura **medallion** (Bronze / Silver / Gold) totalmente dentro do **Microsoft Fabric Lakehouse**, com Power BI conectado ao modelo semântico via **DirectLake** (zero-copy).

A escolha do Fabric é estratégica: é a stack que a 3M usa no time de Enterprise Supply Chain Analytics. Demonstrar fluência nessa stack pesa mais do que entregar o mesmo resultado em uma ferramenta isolada.

## Diagrama (lógico)

```
                          ┌──────────────────────────┐
                          │  TLC CloudFront (Parquet)│
                          │  ACS Open Data (SODA API)│
                          │  NOAA CDO (REST API)     │
                          └────────────┬─────────────┘
                                       │
                                       ▼
                          ┌──────────────────────────┐
                          │  01_bronze_ingest        │
                          │  (PySpark notebook)      │
                          └────────────┬─────────────┘
                                       │
                                       ▼
                          ┌──────────────────────────┐
                          │  Lakehouse / Files       │
                          │  bronze/yellow_taxi/...  │
                          │  bronze/lookups/...      │
                          │  bronze/enrichment/...   │
                          └────────────┬─────────────┘
                                       │
                                       ▼
                          ┌──────────────────────────┐
                          │  02_silver_clean         │
                          │  (Spark SQL)             │
                          │  → silver_yellow_trips   │
                          │    _clean (Delta)        │
                          └────────────┬─────────────┘
                                       │
                                       ▼
                          ┌──────────────────────────┐
                          │  03_gold_model           │
                          │  (Spark SQL)             │
                          │  → fact_trips +          │
                          │    dim_date / dim_time / │
                          │    dim_zone / dim_vendor │
                          │    dim_ratecode /        │
                          │    dim_payment           │
                          └────────────┬─────────────┘
                                       │
                                       ▼
                          ┌──────────────────────────┐
                          │  Semantic Model          │
                          │  sm_yellow_taxi_3m       │
                          │  (DirectLake)            │
                          └────────────┬─────────────┘
                                       │
                                       ▼
                          ┌──────────────────────────┐
                          │  Power BI Desktop (.pbix)│
                          │  + Calc Groups + UDFs    │
                          │  + Field Parameters + RLS│
                          └──────────────────────────┘
```

## Camadas

### Bronze — dados crus, intactos

| Item | Local | Observação |
|---|---|---|
| Parquets TLC | `Files/bronze/yellow_taxi/year=YYYY/month=MM/data.parquet` | Hive-style partitioning |
| Taxi zone lookup | `Files/bronze/lookups/taxi_zone_lookup.csv` | Lookup oficial TLC |
| ACS demographics | `Files/bronze/enrichment/acs_demographics.csv` | Baixado via SODA API |
| NOAA weather | `Files/bronze/enrichment/noaa_weather.csv` | Baixado via CDO API |
| Delta consolidado | `Tables/bronze_yellow_trips_raw` | União de todos os parquets |

**Princípio**: nada é descartado nem alterado nesta camada. Idempotência: o notebook pode ser re-executado sem duplicar.

### Silver — limpo, tipado, enriquecido com derivações

Tabela: `Tables/silver_yellow_trips_clean` (Delta, particionada por `year`, `month`).

Transformações aplicadas:

- Renomeação para snake_case com sufixos de unidade (`trip_distance_mi`, `duration_min`).
- Filtro temporal: `pickup_datetime` entre 2022-01-01 e 2024-12-31.
- Filtros de qualidade (descarte): `trip_distance > 0`, `fare_amount >= 0`, `dropoff > pickup`.
- Campos derivados: `duration_min`, `avg_speed_mph`, `pickup_hour`, `pickup_dow`.
- Flag `is_anomalous` para outliers extremos (não descartados, sinalizados).

### Gold — modelo dimensional (star schema)

| Tabela | Tipo | Granularidade |
|---|---|---|
| `fact_trips` | Fato | 1 linha por viagem (~110M) |
| `dim_date` | Dimensão | 1 linha por dia (1.096 linhas) |
| `dim_time` | Dimensão | 24 linhas (horas do dia) |
| `dim_zone` | Dimensão | 265 zonas TLC + ACS demographics |
| `dim_vendor` | Dimensão | 2 vendors |
| `dim_ratecode` | Dimensão | 6 códigos |
| `dim_payment` | Dimensão | 6 tipos de pagamento |

Relacionamentos single direction, 1:N. `dim_zone` usada duas vezes via role-playing (pickup / dropoff).

### Semantic Model — DirectLake

- Criado sobre as tabelas Gold do Lakehouse.
- Modo DirectLake: lê Delta diretamente do OneLake sem importar dados.
- Conectado ao Power BI Desktop via Live connection.

### Front-end — Power BI Desktop

- Modelo conectado via Live → DAX local + visuais + governança (RLS, UDFs).
- 5 páginas: Executive Overview, Demand Deep Dive, Operations & Cost, Demand vs Demographics, Data Quality.

## Decisões técnicas e justificativas

### Por que Microsoft Fabric e não DuckDB / Snowflake / Databricks?

- **Fabric é a stack da 3M** (citada no processo seletivo). Demonstrar fluência > entregar em ferramenta isolada.
- Lakehouse + Delta + DirectLake formam uma arquitetura moderna e zero-copy, padrão atual do mercado.
- Trial gratuito de 60 dias cobre o período do teste.

### Por que medallion (Bronze/Silver/Gold)?

- Padrão amplamente reconhecido em data engineering.
- Separa concerns: raw / cleaned / modeled, com responsabilidade clara em cada camada.
- Permite reprocessar qualquer camada sem tocar nas anteriores.

### Por que star schema (não snowflake / flat)?

- Star schema é o padrão para modelos semânticos em ferramentas de BI.
- Power BI/Tabular otimiza queries assumindo star schema.
- Field Parameters e Calculation Groups funcionam melhor com dimensões denormalizadas.

### Por que DirectLake (não Import / DirectQuery)?

- **Import** duplica dados (Lakehouse + memória do dataset) — desperdício.
- **DirectQuery** sofre com latência em fato de 110M linhas.
- **DirectLake** lê Delta nativamente do OneLake, performance de Import sem cópia.

### Por que role-playing dimension para Zone (não USERELATIONSHIP)?

- Mais legível no painel de modelo (duas tabelas visíveis: pickup_zone e dropoff_zone).
- Evita medidas DAX condicionais que confundem usuários finais.
- Pequeno overhead de memória aceitável.

## Stack alternativa (slide "como escalaria na 3M")

Para showcase de visão arquitetural, sem construir:

- **Ingestão**: Azure Data Factory ou Fabric Data Pipeline schedulado.
- **Storage**: Lakehouse no Microsoft Fabric (OneLake) ou Snowflake.
- **Transformação**: Notebooks PySpark no Fabric ou dbt sobre Snowflake.
- **Modelagem**: Power BI via DirectLake ou Snowflake DirectQuery.
- **Orquestração**: Fabric Pipelines + Azure DevOps (XMLA endpoint para CI/CD do .pbix).
- **Governança**: Microsoft Purview para data catalog e linhagem.

## Plano B (contingência)

Se Fabric Trial tiver problema:

- Ingestão e ETL em **Python + Polars** local (`python_fallback/`).
- Saída em Parquet no disco local.
- Power BI Desktop conecta direto aos Parquets em modo Import.
- Mesmo modelo, mesma lógica — só muda o ambiente de execução.

Scripts mantidos atualizados em paralelo aos notebooks Fabric para reduzir risco.
