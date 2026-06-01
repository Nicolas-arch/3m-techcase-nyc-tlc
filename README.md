# 3M Technical Case — NYC TLC Yellow Taxi Analytics

[![Status](https://img.shields.io/badge/status-in_development-yellow)]()
[![Stack](https://img.shields.io/badge/stack-Microsoft_Fabric-blue)]()
[![Power BI](https://img.shields.io/badge/Power_BI-Desktop-F2C811)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

> Teste técnico para vaga de **Data / Analytics / BI** no time global de **Enterprise Supply Chain Analytics** da 3M.

## Visão geral

Solução end-to-end de engenharia analítica sobre os dados públicos do **NYC TLC Yellow Taxi (2022–2024)**, traduzindo a operação de táxis para uma narrativa de **cadeia de suprimentos urbana** (demanda, lead time, lanes O-D, eficiência operacional, custos e exception management).

A entrega cobre todo o fluxo:

1. **Data Intake** — ingestão programática de 36 arquivos Parquet (3 anos × 12 meses) da CDN pública da TLC.
2. **Harmonization & Cleansing** — limpeza, tipagem, derivação de campos e flags de qualidade em Spark SQL.
3. **Enrichment** — join com fonte externa: **NYC ACS Demographics** (renda, população, densidade por bairro) e **NOAA Weather** (clima histórico).
4. **Data Model** — star schema em Delta Lake (1 fato + 6 dimensões) materializado no Lakehouse.
5. **Front-end** — relatório Power BI conectado via **DirectLake** com Time Intelligence, Field Parameters, Calculation Groups, Conditional Formatting, RLS e UDFs DAX.

## Stack

| Camada | Ferramenta |
|---|---|
| Storage & Compute | Microsoft Fabric Lakehouse (OneLake + Delta) |
| Ingestão | Fabric Notebook (PySpark) |
| Transformação | Fabric Notebook (Spark SQL) |
| Modelo Semântico | Power BI semantic model em DirectLake |
| Front-end | Power BI Desktop |
| DAX Avançado | Tabular Editor 2 |
| Versionamento | GitHub |
| Plano B local | Python + Polars |

## Estrutura do repositório

```
3m-techcase-nyc-tlc/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md         # arquitetura técnica
│   ├── CONVENTIONS.md          # convenções de nomes e código
│   ├── DATA_SOURCES.md         # fontes de dados e como acessá-las
│   ├── DATA_DICTIONARY.md      # dicionário das tabelas Gold (será preenchido)
│   ├── PRESENTATION_NOTES.md   # decisões técnicas + storytelling p/ apresentação
│   └── CODE_EXPLAINED.md       # material de apoio (didático) do código
├── notebooks/                  # exportados do Fabric
│   ├── 01_bronze_ingest.ipynb     # ✅ ingestão + persistência Delta
│   ├── 02_silver_clean.ipynb      # ✅ filtros de qualidade + derivações
│   └── 03_gold_model.ipynb        # ✅ star schema + ACS + NOAA enrichment
├── sql/                        # scripts SQL auxiliares
├── powerbi/
│   ├── 3m_techcase.pbix
│   ├── theme.json
│   └── measures_reference.md
├── python_fallback/            # ingestão e ETL local (plano B)
├── images/                     # diagramas e screenshots
├── .env.example                # template das variáveis de ambiente
├── .gitignore
├── requirements.txt
└── LICENSE
```

## Pré-requisitos

- Microsoft 365 Developer Program tenant
- Microsoft Fabric Trial ativo
- Power BI Desktop ≥ 2.140 (com preview features: UDFs, Calculation Groups, DirectLake)
- Tabular Editor 2
- Python 3.10+

## Configuração de ambiente

```bash
git clone https://github.com/Nicolas-arch/3m-techcase-nyc-tlc.git
cd 3m-techcase-nyc-tlc

# venv para o plano B local
python -m venv .venv
.venv\Scripts\activate         # Windows
pip install -r requirements.txt

# variáveis de ambiente
copy .env.example .env         # preencher tokens
```

Tokens necessários (cadastro gratuito):

- **NOAA Climate Data Online**: https://www.ncdc.noaa.gov/cdo-web/token
- **NYC Open Data App Token**: https://data.cityofnewyork.us/profile/app_tokens

## Status do projeto

| Etapa | Status |
|---|---|
| Setup de infraestrutura (Fabric Workspace, Lakehouse, GitHub, venv) | ✅ concluído (28/05) |
| Ingestão Bronze (36 parquets + lookup zones) | ✅ concluído (29/05) |
| Limpeza Silver (filtros, derivações, flag de anomalia) | ✅ concluído (30/05) |
| Modelagem Gold + Enrichment (ACS + NOAA) | ✅ concluído (31/05) |
| Modelo semântico Power BI (DirectLake) | ✅ concluído (01/06) |
| Features DAX (Calc Groups + Field Parameters + UDFs) | ✅ concluído (02/06) |
| RLS (5 roles por Borough) + Conditional Formatting | 🟡 em andamento (03/06) |
| Dashboard (5 páginas) | ⏳ |
| Apresentação (.pptx) | ⏳ |

## Highlights do desenvolvimento

### Bronze
- **119.136.044 trips** ingeridas em Delta a partir de 36 arquivos parquet (~1.85 GB).
- **Schema canônico explícito** para resolver schema drift entre arquivos da TLC (INT vs BIGINT, `airport_fee` vs `Airport_fee`).
- **Idempotência** na ingestão: re-execução não duplica arquivos nem dados.
- **663 registros descobertos com data fora do escopo** (2001 a 2026) — descoberta de qualidade de dados.

### Silver
- **115.756.175 trips** após limpeza (98,16% do volume Bronze).
- **3.379.869 linhas descartadas (2,84%)** com breakdown documentado:
  - 2.123.821 com distância ≤ 0 (cancelamentos / erros de meter)
  - 1.365.574 com tarifa negativa (estornos / correções)
  - 61.114 com duração inválida (dropoff ≤ pickup)
  - 663 com vazamento temporal (2001-2026)
- **180.145 anomalias flagged** (0,16%) — mantidas no dataset para análise no dashboard de Data Quality.
- **Top anomalia detectada**: trip de **US$ 401.092** em 10 minutos (erro óbvio de meter) — vira caso de exception management na apresentação.
- **Padrão YoY**: -3,4% em 2023, +7,5% em 2024 — narrativa de demand recovery pós-pandemia.

### Gold (Star Schema)
- **1 fato** (`fact_trips` — 115,7M trips particionado por year/month) + **6 dimensões**.
- **dim_date enriquecida com clima NOAA** (1.096 dias × TMAX, TMIN, PRCP, SNOW, weather_category, is_extreme_weather).
- **dim_zone enriquecida com ACS Borough demographics** (population, median income, density).
- **Integridade referencial validada**: zero foreign keys orfãs no fato (padrão COALESCE → Unknown).
- **Padrão híbrido de fontes externas**: NOAA via REST API com paginação anual (operational data), ACS via snapshot Borough-level (reference data).
- **Gotcha documentado**: NOAA CDO API limita range a 1 ano por request — fix via loop por ano.

### DAX Advanced Features (dia 02/06)
- **Calculation Group "Time Analytics"** criado via Tabular Editor 2 com 7 calculation items (Current, PY, YoY, YoY%, YTD, PYTD, MAT). Em vez de duplicar 10 medidas × 7 transformações = 70 medidas, mantém 10 medidas + 7 items = 17 objetos.
- **2 Field Parameters** (FP_Metric com 5 medidas + FP_Dimension com 5 dimensões) — 25 combinações analíticas em 1 visual.
- **3 UDFs DAX** (User-Defined Functions, preview feature 2025): `ClassifyTripDuration`, `ClassifyFareTier`, `ComputeRevenuePerMile`. Encapsulam lógica reutilizável.
- **Padrão YoY validado pelo Calc Group**: +27,4% em 2023 (recovery pós-pandemia) e +5,7% em 2024 (estabilização).
- **Insight Field Parameters**: Afternoon + Evening = 70% do volume diário — padrão clássico de demanda urbana.
- **Insight UDFs**: EWR (Newark Airport) tem Revenue per Mile de $24,05 — 5x a média de Manhattan, padrão nicho premium.
- **Gotchas documentados**: sintaxe DAX UDF sem aspas no DEFINE, preview feature exige restart do PBI Desktop, BLANK→0 coercion em SWITCH.

### Power BI Semantic Model (DirectLake / Composite)
- **`sm_yellow_taxi_3m`** criado em DirectLake sobre o Lakehouse `lh_yellow_taxi`.
- **Power BI Desktop conectado via composite model** (DirectQuery + remote semantic model) — permite adicionar medidas, hierarquias e RLS locais sem perder zero-copy.
- **7 relacionamentos** configurados no semantic model (1 fato + 6 dimensões + 1 role-playing para Dropoff Zone).
- **4 hierarquias** criadas: Date (Year → Quarter → Month → Day), Time (Day Part → Hour), Pickup Zone (Borough → Service Zone → Zone Name), Dropoff Zone.
- **`dim_date` marcada como Date Table** — habilita Time Intelligence DAX.
- **10 medidas base** criadas em `_Measures` (Total Trips, Trips per Day, Total Revenue, Avg Fare, Total Tip, Tip Rate %, Total Distance, Avg Distance, Avg Duration, Anomaly Rate %).
- **17 colunas técnicas ocultas** (surrogate keys + business IDs) — padrão Kimball.
- **Validação cruzada**: matriz Power BI bate 100% com o business sanity query rodado no Gold (Manhattan 2024: 35,27M trips, US$ 844M; total 2024: 39,7M trips, US$ 1,14B).

Decisões técnicas e respostas-padrão para apresentação em [`docs/PRESENTATION_NOTES.md`](docs/PRESENTATION_NOTES.md).
Material didático do código em [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## Critérios do teste

- ✅ Escopo: Yellow Taxi 2022–2024
- ✅ Fonte externa correlacionada (ACS Demographics + NOAA Weather)
- ✅ Features obrigatórias planejadas: Time Intelligence, Field Parameters, Calculation Groups, Conditional Formatting, RLS, UDFs

## Autor

Nicolas Augusto — Campinas/SP — [LinkedIn](#)

## Licença

[MIT](LICENSE)
