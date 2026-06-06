# Data Dictionary — Gold Layer

> Dicionário de dados das tabelas do modelo dimensional (Gold). Use como referência ao construir medidas DAX e visuais no Power BI.

## fact_trips

Tabela-fato central. Granularidade: 1 linha por viagem (trip). Particionada por `year`, `month`.

| Coluna | Tipo | Origem | Descrição |
|---|---|---|---|
| `date_key` | int | derivado de pickup_datetime | FK para `dim_date`. Formato yyyyMMdd. |
| `time_key` | int | derivado de pickup_datetime | FK para `dim_time`. Valor 0-23 (hora do pickup). |
| `pu_zone_key` | bigint | derivado via join | FK para `dim_zone` (pickup location). |
| `do_zone_key` | bigint | derivado via join | FK para `dim_zone` (dropoff location). |
| `vendor_key` | int | derivado via join + COALESCE | FK para `dim_vendor`. COALESCE para 5 (Unknown) se NULL. |
| `ratecode_key` | int | derivado via join + COALESCE | FK para `dim_ratecode`. COALESCE para 7 (Unknown) se NULL. |
| `payment_key` | int | derivado via join + COALESCE | FK para `dim_payment`. COALESCE para 6 (Unknown) se NULL. |
| `year` | int | particionamento | Ano do pickup (2022, 2023, 2024). |
| `month` | int | particionamento | Mês do pickup (1-12). |
| `trip_count` | int | constante | Sempre 1. Permite SUM para contagem rápida. |
| `passenger_count` | int | TLC original | Número de passageiros declarado pelo motorista. |
| `trip_distance_mi` | double | TLC original | Distância em milhas. |
| `duration_min` | double | derivado (Silver) | Duração em minutos (dropoff - pickup). |
| `avg_speed_mph` | double | derivado (Silver) | Velocidade média em mph. NULL se duração ≤ 0. |
| `fare_amount` | double | TLC original | Tarifa base. |
| `extra_amount` | double | TLC original | Extras (rush hour, overnight). |
| `mta_tax` | double | TLC original | MTA tax (US$ 0.50 por trip elegível). |
| `tip_amount` | double | TLC original | Gorjeta (só em cartão, cash não rastreado). |
| `tolls_amount` | double | TLC original | Pedágios. |
| `improvement_surcharge` | double | TLC original | US$ 0.30 por trip. |
| `total_amount` | double | TLC original | Total cobrado (sem gorjeta em cash). |
| `congestion_surcharge` | double | TLC original | Sobretaxa Manhattan abaixo da 96th St. |
| `airport_fee` | double | TLC original | Taxa para LGA/JFK. |
| `is_anomalous` | int | derivado (Silver) | Flag 1/0. Critérios: duration > 360min OR distance > 200mi OR speed > 80mph OR total > $1000. |
| `is_weekend` | int | derivado (Silver) | Flag 1/0. Sábado ou domingo. |
| `day_part` | string | derivado (Silver) | Late Night / Morning / Afternoon / Evening. |

**Volume**: 115.756.175 linhas (97,16% do Bronze, após filtros de qualidade).

---

## dim_date

Calendário diário 2022-01-01 a 2024-12-31, enriquecido com clima diário do NOAA.

| Coluna | Tipo | Descrição |
|---|---|---|
| `date_key` | int | PK. Formato yyyyMMdd (ex.: 20220101). |
| `full_date` | date | Data completa. |
| `year` | int | Ano (2022, 2023, 2024). |
| `quarter` | int | Trimestre (1-4). |
| `month` | int | Mês (1-12). |
| `month_name` | string | "January", "February"... |
| `month_short` | string | "Jan", "Feb"... |
| `week_iso` | int | Semana ISO (1-53). |
| `day_of_month` | int | Dia do mês (1-31). |
| `day_of_week` | int | Dia da semana (1=domingo, 7=sábado, Spark default). |
| `day_name` | string | "Monday", "Tuesday"... |
| `day_short` | string | "Mon", "Tue"... |
| `is_weekend` | int | Flag 1/0. |
| `season` | string | Winter / Spring / Summer / Fall. |
| `year_quarter` | string | Ex.: "2022-Q3". |
| `year_month` | string | Ex.: "2022-03". |
| `temp_max_celsius_tenths` | int | Temperatura máxima em décimos de °C (NOAA). Divida por 10 para °C. |
| `temp_min_celsius_tenths` | int | Temperatura mínima em décimos de °C. |
| `precipitation_mm_tenths` | int | Precipitação em décimos de mm. |
| `snowfall_mm` | int | Neve acumulada do dia em mm. |
| `snow_depth_mm` | int | Profundidade da neve no solo em mm. |
| `weather_category` | string | Snow / Heavy Rain / Rain / Clear / Unknown. |
| `is_extreme_weather` | int | Flag 1/0. Critério: snowfall > 50mm OR precipitation > 25mm. |

**Volume**: 1.096 linhas (3 anos × 365 + 1 dia bissexto em 2024).

---

## dim_time

Hora do dia.

| Coluna | Tipo | Descrição |
|---|---|---|
| `time_key` | int | PK. Igual a `hour` (0-23). |
| `hour` | int | 0-23. |
| `hour_label` | string | "00:00", "01:00", ..., "23:00". |
| `day_part` | string | Late Night (0-5) / Morning (6-11) / Afternoon (12-17) / Evening (18-23). |
| `day_part_order` | int | 1-4. Para ordenação em visuais. |
| `is_rush_hour` | int | Flag 1/0. 7-9h ou 17-19h. |

**Volume**: 24 linhas.

---

## dim_zone

265 zonas TLC enriquecidas com demografia ACS no nível Borough.

| Coluna | Tipo | Descrição |
|---|---|---|
| `zone_key` | bigint | PK (surrogate via ROW_NUMBER). |
| `location_id` | bigint | ID natural da zona TLC. |
| `borough` | string | Manhattan, Brooklyn, Queens, Bronx, Staten Island, EWR, Unknown. |
| `zone_name` | string | Nome da zona (ex.: "JFK Airport", "Times Sq/Theatre District"). |
| `service_zone` | string | Boro Zone / Yellow Zone / EWR / Airports / N/A. |
| `total_population` | bigint | População do borough (ACS 2022 5-year). 0 se Unknown. |
| `median_household_income` | bigint | Renda mediana familiar USD do borough. 0 se Unknown. |
| `area_sq_mi` | double | Área do borough em milhas². 0 se Unknown. |
| `population_density_per_sq_mi` | double | Habitantes por milha². Calculado. |

**Volume**: 265 linhas.

**Nota de design**: enrichment ACS no nível Borough (não Zone). Decisão documentada em PRESENTATION_NOTES seção 4.3.

---

## dim_vendor

Provedor do sistema TPEP (Taxicab Passenger Enhancement Project).

| Coluna | Tipo | Descrição |
|---|---|---|
| `vendor_key` | int | PK. |
| `vendor_id` | int | ID natural TLC (1, 2, 6, 7 ou NULL). |
| `vendor_name` | string | Creative Mobile Technologies / Curb Mobility / Myle Technologies / Helix / Unknown. |

**Volume**: 5 linhas. `vendor_key=5` é o "Unknown" usado como fallback no fato.

---

## dim_ratecode

Código de tipo de tarifa.

| Coluna | Tipo | Descrição |
|---|---|---|
| `ratecode_key` | int | PK. |
| `ratecode_id` | int | ID natural TLC (1-6 ou 99 para Unknown). |
| `ratecode_name` | string | Standard Rate / JFK / Newark / Nassau or Westchester / Negotiated Fare / Group Ride / Null/Unknown. |

**Volume**: 7 linhas. `ratecode_key=7` é o "Unknown" usado como fallback.

**Notas operacionais**:
- `JFK` (2): tarifa fixa para John F. Kennedy Airport.
- `Newark` (3): tarifa fixa para Newark Airport (EWR).
- `Nassau or Westchester` (4): tarifa metered para condados externos.
- `Negotiated Fare` (5): tarifa negociada (típico de viagens longas).
- `Group Ride` (6): viagem compartilhada.

---

## dim_payment

Tipo de pagamento.

| Coluna | Tipo | Descrição |
|---|---|---|
| `payment_key` | int | PK. |
| `payment_type_id` | int | ID natural TLC (0-6 ou NULL). |
| `payment_type_name` | string | Flex Fare Trip / Credit Card / Cash / No Charge / Dispute / Unknown / Voided Trip. |

**Volume**: 7 linhas. `payment_key=6` é o "Unknown" usado como fallback.

**Notas operacionais**:
- `Dispute` (4) e `Voided Trip` (6): proxy de cancelamento — vão para análise no dashboard de Data Quality.
- `Cash` (2): gorjeta não é rastreada quando pagamento é em dinheiro (campo `tip_amount` aparece como 0).

---

## Relacionamentos no semantic model (a configurar no Power BI)

| Tabela origem | FK | Tabela destino | PK | Direção |
|---|---|---|---|---|
| `fact_trips` | `date_key` | `dim_date` | `date_key` | Single (Many → One) |
| `fact_trips` | `time_key` | `dim_time` | `time_key` | Single (Many → One) |
| `fact_trips` | `pu_zone_key` | `dim_zone` (Pickup Zone) | `zone_key` | Single, ativo |
| `fact_trips` | `do_zone_key` | `dim_zone` (Dropoff Zone) | `zone_key` | Single, ativo (segunda cópia da tabela) |
| `fact_trips` | `vendor_key` | `dim_vendor` | `vendor_key` | Single |
| `fact_trips` | `ratecode_key` | `dim_ratecode` | `ratecode_key` | Single |
| `fact_trips` | `payment_key` | `dim_payment` | `payment_key` | Single |

**Importante**: `dim_zone` será duplicada no semantic model (uma instância para Pickup Zone, outra para Dropoff Zone) — role-playing dimension explícita por clareza.

## Hierarquias sugeridas

- **Date**: Year → Quarter → Month → Day
- **Zone**: Borough → Service Zone → Zone Name
- **Time**: Day Part → Hour
