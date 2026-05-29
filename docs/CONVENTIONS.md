# Convenções do projeto

Este documento define as convenções de nomes, idioma e estrutura usadas em todo o projeto. O objetivo é manter consistência e facilitar a manutenção/extensão por qualquer engenheiro de dados.

## Idioma

| Onde | Idioma |
|---|---|
| Código (Python, SQL, DAX, M) | **Inglês** |
| Nomes de pastas, arquivos, tabelas, colunas | **Inglês** |
| Nomes de medidas e variáveis DAX | **Inglês** |
| Commits Git | **Inglês** |
| Documentação (`docs/`, `README.md`, comentários funcionais) | **Português** durante desenvolvimento, **inglês** na versão final |
| Slides da apresentação | **Português** durante desenvolvimento, **inglês** se solicitado |

> A versão final do projeto terá toda a documentação traduzida para inglês.

## Naming conventions

### Pastas e arquivos

- Pastas: `snake_case`, plural quando coleção (`notebooks/`, `images/`).
- Arquivos Python: `snake_case.py`.
- Arquivos SQL: `snake_case.sql`.
- Arquivos Markdown: `UPPER_SNAKE.md` para docs (`ARCHITECTURE.md`), `snake_case.md` para conteúdo (`measures_reference.md`).
- Arquivos PowerBI: `snake_case.pbix`.

### Tabelas

- **Bronze**: prefixo `bronze_` + entidade no singular ou plural conforme contexto.
  - `bronze_yellow_trips_raw`
  - `bronze_taxi_zone_lookup`
  - `bronze_acs_demographics`
  - `bronze_noaa_weather`
- **Silver**: prefixo `silver_` + entidade + sufixo de estado.
  - `silver_yellow_trips_clean`
- **Gold**: prefixo `fact_` ou `dim_`.
  - `fact_trips`
  - `dim_date`, `dim_time`, `dim_zone`, `dim_vendor`, `dim_ratecode`, `dim_payment`

### Colunas

- Sempre `snake_case`.
- Sufixos de unidade obrigatórios para campos numéricos com unidade física:
  - `_mi` (miles)
  - `_min` (minutes)
  - `_sec` (seconds)
  - `_mph` (miles per hour)
  - `_usd` (dollars)
- Chaves estrangeiras:
  - Surrogate keys terminam em `_key` (ex.: `date_key`, `zone_key`).
  - Business IDs terminam em `_id` (ex.: `location_id`, `vendor_id`).
- Booleans/flags começam com `is_`:
  - `is_weekend`, `is_holiday`, `is_anomalous`.
- Datetimes terminam em `_datetime`:
  - `pickup_datetime`, `dropoff_datetime`.
- Datas terminam em `_date`:
  - `full_date`.

### Medidas DAX

- **Title Case** com palavras separadas por espaço:
  - `Total Trips`, `Avg Fare`, `Revenue per Mile`.
- Prefixos opcionais por categoria:
  - `_` para medidas técnicas/auxiliares: `_Selected Metric`.
- Medidas dentro de Calculation Group: nome curto do item:
  - `Current`, `PY`, `YoY %`, `YTD`, `MAT`.

### UDFs (DAX User-Defined Functions)

- **PascalCase**:
  - `ClassifyTripDuration`, `ClassifyFareTier`, `ComputeRevenuePerMile`.

### Field Parameters

- Prefixo `FP_`:
  - `FP_Metric`, `FP_Dimension`.

### Roles (RLS)

- `<Borough> Manager`:
  - `Manhattan Manager`, `Brooklyn Manager`, etc.
- Role global sem filtro: `Global Manager`.

### Variáveis Python

- `snake_case`.
- Constantes em `UPPER_SNAKE`.
- Funções em `snake_case` com prefixo de ação: `download_`, `load_`, `clean_`, `validate_`.

### Variáveis Spark SQL / CTEs

- `snake_case`.
- CTEs nomeadas como substantivos curtos: `cleaned`, `enriched`, `aggregated`.

## Git

### Branches

- `main` — sempre verde (código que roda).
- Não usamos feature branches neste projeto (escopo pessoal e curto).

### Commits

Padrão **Conventional Commits**:

```
<type>: <short description>

<optional body>
```

Tipos:

- `feat:` — nova feature ou camada de dados
- `fix:` — correção de bug
- `docs:` — apenas documentação
- `refactor:` — refatoração sem mudança funcional
- `chore:` — setup, infraestrutura, deps
- `data:` — atualização de dados ou pipeline

Exemplos:

```
chore: initial project scaffolding
feat: implement bronze ingestion notebook
feat: add silver cleaning with quality flags
docs: add data dictionary for gold layer
fix: handle null passenger_count in silver
```

### Tags

- `v0.1` — fim da camada Silver.
- `v0.5` — fim do star schema Gold.
- `v0.9` — relatório completo, antes de polimento.
- `v1.0` — versão final entregue.

## Estrutura de pastas

```
3m-techcase-nyc-tlc/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── CONVENTIONS.md
│   ├── DATA_SOURCES.md
│   └── DATA_DICTIONARY.md
├── notebooks/
├── sql/
├── powerbi/
├── python_fallback/
├── images/
├── .env.example
├── .gitignore
├── requirements.txt
└── LICENSE
```

## Segurança

- Tokens e credenciais **sempre** em `.env`.
- `.env` está no `.gitignore`.
- `.env.example` documenta as variáveis (sem valores reais).
- Nenhum dado pessoal/sensível no repo (TLC já é anonimizado, ACS é agregado).
