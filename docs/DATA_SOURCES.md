# Fontes de Dados

Este documento descreve todas as fontes de dados usadas no projeto, com URLs, métodos de acesso e schemas.

## 1. NYC TLC Yellow Taxi Trip Records (fonte primária)

### Sobre

Registros oficiais de viagens de táxis amarelos em NYC, publicados mensalmente pela TLC (Taxi and Limousine Commission) desde 2009.

### Escopo do projeto

- **Apenas** Yellow Taxi (Green, FHV, HVFHV fora do escopo).
- **Apenas** anos 2022, 2023 e 2024.
- Total estimado: **~110 milhões de viagens** em 36 arquivos Parquet.

### Acesso

- **Página oficial**: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
- **CDN direta** (CloudFront): `https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_YYYY-MM.parquet`
- **Sem autenticação**.

### Lookups

- **Taxi zone lookup (CSV)**: `https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv`
- **Taxi zone shapefile (ZIP)**: `https://d37ci6vzurychx.cloudfront.net/misc/taxi_zones.zip`

### Schema principal

| Campo (original) | Tipo | Renomeado para | Observação |
|---|---|---|---|
| `VendorID` | int | `vendor_id` | 1 = Creative Mobile, 2 = Curb |
| `tpep_pickup_datetime` | timestamp | `pickup_datetime` | |
| `tpep_dropoff_datetime` | timestamp | `dropoff_datetime` | |
| `passenger_count` | int | `passenger_count` | Pode ter null |
| `trip_distance` | double | `trip_distance_mi` | Milhas |
| `PULocationID` | int | `pu_location_id` | FK taxi_zone_lookup |
| `DOLocationID` | int | `do_location_id` | FK taxi_zone_lookup |
| `RatecodeID` | int | `ratecode_id` | 1=Standard, 2=JFK, 3=Newark, 4=Nassau, 5=Negotiated, 6=Group |
| `payment_type` | int | `payment_type_id` | 1=CC, 2=Cash, 3=No charge, 4=Dispute, 5=Unknown, 6=Voided |
| `fare_amount` | double | `fare_amount` | Tarifa base |
| `total_amount` | double | `total_amount` | Total cobrado |
| `tip_amount` | double | `tip_amount` | |
| `congestion_surcharge` | double | `congestion_surcharge` | Manhattan abaixo da 96th St |
| `airport_fee` | double | `airport_fee` | LGA/JFK |

### Dicionário oficial

PDF da TLC: https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf

---

## 2. NYC ACS Demographics (fonte externa — enrichment primário)

### Sobre

Dados do **American Community Survey (ACS)** do U.S. Census Bureau, agregados por **NTA (Neighborhood Tabulation Areas)** de NYC. Indicadores socioeconômicos: população, renda, ocupação.

### Por que essa fonte

Permite correlacionar **demanda de táxi** com **densidade populacional**, **renda média** e **% de ocupação** por bairro — equivalente direto a uma análise de **demand drivers** em supply chain (densidade de clientes × ticket médio × poder de compra).

### Acesso

- **Portal NYC Open Data**: https://data.cityofnewyork.us/
- **Dataset alvo**: ACS Demographic and Housing Estimates by NTA (procurar "ACS demographic" no portal).
- **Método de acesso**:
  - **Opção A (recomendada)**: API SODA via `sodapy` (Python) — permite query e paginação.
  - **Opção B**: Download CSV direto pelo botão "Export" do portal.

### Autenticação (API)

- **App Token** (opcional, mas aumenta rate limit):
  - Cadastro: https://data.cityofnewyork.us/profile/app_tokens
  - Variável de ambiente: `NYC_APP_TOKEN`, `NYC_APP_SECRET`

### Indicadores trazidos

| Campo | Descrição |
|---|---|
| `nta_code` | Código do NTA |
| `nta_name` | Nome do bairro |
| `total_population` | População residente |
| `median_household_income` | Renda mediana domiciliar (USD) |
| `employment_rate` | % população ocupada |
| `population_density` | Habitantes por km² (calculado) |

### Crosswalk NTA ↔ Taxi Zone

NTAs e Taxi Zones têm geometrias diferentes. Precisamos de um mapping:

- **Opção 1 (mais rápida)**: usar mapping pronto disponível em repositórios open-source (referência: github.com/toddwschneider/nyc-taxi-data).
- **Opção 2 (espacial)**: usar Geopandas + shapefile das duas geografias e calcular overlap por área (maior precisão, mais trabalho).

Decisão: opção 1 com ajustes manuais para zonas críticas (aeroportos, áreas industriais).

---

## 3. NOAA Weather (fonte externa — enrichment secundário)

### Sobre

Dados climáticos históricos da estação **Central Park** (NYC), via NOAA Climate Data Online (CDO).

### Por que essa fonte

Demonstra análise de **risco operacional** — "shocks externos" típicos em supply chain (clima, paralisações, eventos). Permite responder: a demanda cai em dias de chuva/neve? Lead time aumenta?

### Acesso

- **NOAA CDO API**: https://www.ncdc.noaa.gov/cdo-web/api/v2/
- **Estação**: `GHCND:USW00094728` (Central Park, NY)
- **Dataset**: `GHCND` (Daily Summaries)
- **Período**: 2022-01-01 a 2024-12-31

### Autenticação

- **Token gratuito** (obrigatório):
  - Cadastro: https://www.ncdc.noaa.gov/cdo-web/token
  - Variável de ambiente: `NOAA_TOKEN`
  - Limite: 1.000 requests/dia (suficiente).

### Variáveis trazidas

| Datatype | Descrição |
|---|---|
| `TMAX` | Temperatura máxima do dia (décimos de °C) |
| `TMIN` | Temperatura mínima do dia |
| `PRCP` | Precipitação (mm) |
| `SNOW` | Neve acumulada (mm) |
| `SNWD` | Profundidade da neve no solo (mm) |
| `AWND` | Velocidade média do vento |

### Modelagem no Gold

Atributos derivados:

- `weather_category`: Clear / Rain / Snow / Extreme.
- `is_extreme_weather` (flag): `PRCP > 25mm` ou `SNOW > 50mm`.
- Junção com `dim_date` via `full_date`.

---

## 4. Resumo das credenciais necessárias

Arquivo `.env` (não versionado):

```
# NOAA Climate Data Online
NOAA_TOKEN=<obtido em https://www.ncdc.noaa.gov/cdo-web/token>

# NYC Open Data
NYC_APP_TOKEN=<obtido em https://data.cityofnewyork.us/profile/app_tokens>
NYC_APP_SECRET=<obtido em https://data.cityofnewyork.us/profile/app_tokens>

# Fabric tenant (não secret, só referência)
FABRIC_TENANT=<seu>.onmicrosoft.com
```

## 5. Estratégia de versionamento dos dados

- **Bronze**: dados originais não vão pro Git (estão no Lakehouse e podem ser re-baixados).
- **Crosswalk NTA ↔ Taxi Zone**: vai pro repo (`sql/nta_zone_mapping.csv`) por ser pequeno e reproduzível.
- **Tokens**: nunca no Git (`.env` está no `.gitignore`).
- **Lookup oficial da TLC**: pode ir pro repo (`sql/taxi_zone_lookup.csv`) para garantir reprodutibilidade.
