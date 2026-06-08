# Material de Apoio — Como funciona o código deste projeto

> Documento didático para entender cada linha que estamos escrevendo. Não é tutorial de Spark do zero — é um "decoder" do código deste projeto específico, peça por peça, em português direto.

## Índice

1. [Conceitos fundamentais](#1-conceitos-fundamentais)
2. [Notebook 01_bronze_ingest (PySpark)](#2-notebook-01_bronze_ingest-pyspark)
3. [Notebook 02_silver_clean (Spark SQL)](#3-notebook-02_silver_clean-spark-sql)
4. [Notebook 03_gold_model (Spark SQL + PySpark misto)](#4-notebook-03_gold_model-spark-sql--pyspark-misto)
5. [Power BI Semantic Model (dia 01/06)](#5-power-bi-semantic-model-dia-0106)
6. [DAX Advanced (dia 02/06)](#6-dax-advanced-dia-0206)
7. [Conceitos avançados de Power BI](#7-conceitos-avançados-de-power-bi)
8. [Glossário rápido](#8-glossário-rápido)
9. [DAX da Página 1 — medidas e gotchas (dia 4)](#9-dax-da-página-1--medidas-e-gotchas-dia-4)
10. [DAX da Página 2 — Demand Deep Dive (dia 8)](#10-dax-da-página-2--demand-deep-dive-dia-8)
11. [DAX da Página 3 — Operations & Cost (dia 8)](#11-dax-da-página-3--operations--cost-dia-8)
12. [DAX da Página 4 — Demand vs Demographics (dia 8)](#12-dax-da-página-4--demand-vs-demographics-dia-8)
13. [DAX da Página 5 — Data Quality & Governance (dia 8)](#13-dax-da-página-5--data-quality--governance-dia-8)

---

## 1. Conceitos fundamentais

### 1.1 PySpark vs Spark SQL — duas formas de falar com o mesmo motor

O **Apache Spark** é o engine de processamento distribuído. Ele recebe instruções e executa em paralelo.

Você pode dar essas instruções de duas formas:

- **PySpark**: API em Python. Você manipula objetos do tipo `DataFrame` com métodos (`.filter()`, `.select()`, `.groupBy()`).
- **Spark SQL**: você escreve SQL puro (`SELECT ... FROM ... WHERE`). O Spark traduz internamente.

**Quando usar cada um?**

| Caso | Use |
|---|---|
| Preciso de loops, condicionais complexos, libs Python externas (requests, pandas) | PySpark |
| Quero ler/escrever tabelas, joins, agregações | Spark SQL (mais legível) |
| Modelagem dimensional, CTAS | Spark SQL |
| Ingestão de arquivos da internet | PySpark (precisa de requests) |

Nosso projeto usa **PySpark no Bronze** (precisa baixar arquivos) e **Spark SQL no Silver/Gold** (transformações tabulares).

### 1.2 DataFrame — o que é

Um `DataFrame` é uma tabela em memória do Spark. Tem colunas com tipos, mas **não está realmente carregada** até você forçar uma ação (count, show, write).

Isso se chama **lazy evaluation**: o Spark vai acumulando operações e só executa quando precisa. Por isso quando você escreve:

```python
df = spark.read.parquet("...")
df = df.filter("year = 2022")
df = df.select("VendorID", "fare_amount")
```

Ainda não rodou nada. Só quando você faz:

```python
df.count()    # ou df.show(), df.write...
```

Aí o Spark planeja tudo, otimiza, e executa.

### 1.3 Delta Lake — o "Excel turbinado" do data lake

Delta é o formato em que salvamos as tabelas. Por baixo são arquivos Parquet + um log de transações. Vantagens:

- **ACID**: transações seguras (não corrompe se der erro no meio da escrita).
- **Time travel**: você pode consultar como estava 10 commits atrás.
- **Schema enforcement**: rejeita inserts com schema errado.
- **Update / Delete / Merge**: coisas que parquet puro não permite.

Quando a gente escreve `.saveAsTable("nome")`, o Spark cria uma **Managed Delta Table** no Lakehouse. Você acessa via `spark.read.table("nome")` ou `SELECT ... FROM nome`.

### 1.4 Arquitetura Medallion (Bronze / Silver / Gold)

Padrão de nomeação das camadas:

- **Bronze (raw)**: dados originais, intactos. Apenas formato e organização. Sem regras de negócio.
- **Silver (clean)**: dados limpos, tipados, com filtros de qualidade e derivações. Pronto pra análise.
- **Gold (modeled)**: modelo dimensional (fato + dimensões), pronto pra consumo do Power BI.

Por que separar? **Reprocessamento isolado**. Se eu mudar uma regra do Silver, não precisa baixar tudo de novo do Bronze.

### 1.5 Particionamento

Quando salvamos uma tabela com `PARTITIONED BY (year, month)`, o Delta cria subpastas:

```
silver_yellow_trips_clean/
├── year=2022/month=01/parte1.parquet
├── year=2022/month=02/parte1.parquet
...
```

Vantagem: queries com `WHERE year = 2022 AND month = 06` só leem **um arquivo**. Isso se chama **partition pruning** — Spark "poda" partições que não precisa ler.

---

## 2. Notebook 01_bronze_ingest (PySpark)

### Cell 1 — Imports e constantes

```python
import os
import time
import requests
from concurrent.futures import ThreadPoolExecutor

BASE_URL = "https://d37ci6vzurychx.cloudfront.net/trip-data"
LAKEHOUSE_ROOT = "/lakehouse/default/Files/bronze/yellow_taxi"
YEARS = [2022, 2023, 2024]
MONTHS = list(range(1, 13))
```

**O que cada parte faz:**
- `import os, time` — libs padrão Python para mexer com arquivos e medir tempo.
- `import requests` — lib para baixar arquivos HTTP (a TLC publica via HTTPS).
- `ThreadPoolExecutor` — permite rodar **múltiplas threads em paralelo** (3 downloads simultâneos).
- `BASE_URL` — URL base dos arquivos da TLC.
- `LAKEHOUSE_ROOT` — caminho no Lakehouse onde salvar. `/lakehouse/default/Files/` é o **caminho local** dentro do notebook Fabric para acessar a área de Files do Lakehouse anexado.
- `YEARS`, `MONTHS` — constantes do escopo.

### Cell 2 — Função `download_month`

```python
def download_month(year: int, month: int) -> dict:
    fname = f"yellow_tripdata_{year}-{month:02d}.parquet"
    url   = f"{BASE_URL}/{fname}"
    out_d = f"{LAKEHOUSE_ROOT}/year={year}/month={month:02d}"
    out_f = f"{out_d}/data.parquet"

    if os.path.exists(out_f) and os.path.getsize(out_f) > 0:
        return {"year": year, "month": month, "status": "skipped", ...}

    os.makedirs(out_d, exist_ok=True)
    ...
```

**Conceitos:**
- `f"..."` (f-string) — interpola variáveis no texto. `{month:02d}` formata o mês com 2 dígitos (`01`, `02`...).
- `os.path.exists()` + `os.path.getsize()` — verifica se o arquivo já existe e não está vazio. **Isso é idempotência**: rodar a célula 2 vezes não duplica o trabalho.
- `os.makedirs(out_d, exist_ok=True)` — cria os diretórios pai se não existem. `exist_ok=True` evita erro se já existirem.

**Por que estruturar como função?** Encapsula a lógica de 1 arquivo. Aí no Cell 3 a gente chama essa função 36 vezes em paralelo.

```python
with requests.get(url, stream=True, timeout=180) as r:
    r.raise_for_status()
    with open(out_f, "wb") as f:
        for chunk in r.iter_content(1024 * 1024):
            f.write(chunk)
```

- `stream=True` — não carrega o arquivo inteiro em memória. Lê pedaços.
- `iter_content(1024 * 1024)` — lê em blocos de 1 MB. Isso é importante para não estourar RAM com arquivos grandes.
- `r.raise_for_status()` — se o HTTP der erro (404, 500), levanta exceção.
- `with open(...) as f:` — abre o arquivo para escrita binária (`"wb"`) e garante que será fechado depois (mesmo se der erro).

### Cell 3 — Execução paralela

```python
tasks = [(y, m) for y in YEARS for m in MONTHS]   # lista de 36 tuplas

with ThreadPoolExecutor(max_workers=3) as executor:
    for r in executor.map(lambda t: download_month(*t), tasks):
        results.append(r)
```

**Conceitos:**
- **List comprehension**: `[(y, m) for y in YEARS for m in MONTHS]` gera `[(2022,1), (2022,2), ..., (2024,12)]` — 36 pares.
- `ThreadPoolExecutor(max_workers=3)` — pool de 3 threads. O Python vai pegar 3 tarefas por vez e rodar em paralelo. Por que só 3? Pra não estressar o CloudFront e ter throttle.
- `executor.map(lambda t: download_month(*t), tasks)` — aplica a função em cada tarefa. `*t` desempacota a tupla `(year, month)` nos parâmetros da função.

### Cell 4 — Ler todos os parquets com schema canônico

Já documentamos em detalhes no PRESENTATION_NOTES (seção 4: schema drift). Aqui o resumo do que faz cada peça:

```python
TARGET_COLS = [
    ("VendorID", "bigint"),
    ...
]

def read_normalized(year, month):
    df = spark.read.parquet(path)             # lê o parquet do mês
    col_lookup = {c.lower(): c for c in df.columns}   # mapa case-insensitive
    select_exprs = []
    for tgt_name, tgt_type in TARGET_COLS:
        src_name = col_lookup.get(tgt_name.lower())
        if src_name is None:
            select_exprs.append(F.lit(None).cast(tgt_type).alias(tgt_name))
        else:
            select_exprs.append(F.col(src_name).cast(tgt_type).alias(tgt_name))
    return df.select(*select_exprs)...
```

**Conceitos novos:**
- `F.col("xxx")` — referência a uma coluna do DataFrame. `F` é o alias de `pyspark.sql.functions`.
- `.cast("bigint")` — converte o tipo da coluna.
- `.alias("novo_nome")` — renomeia.
- `F.lit(None)` — literal NULL com tipo definido pelo cast.
- `df.select(*select_exprs)` — o `*` "explode" a lista de expressions como argumentos separados.

**O `reduce(...)`:**
```python
df_bronze = reduce(lambda a, b: a.unionByName(b), dfs)
```
- `reduce` aplica uma função 2 a 2 numa lista. Aqui une os 36 DataFrames em um só.
- `unionByName` — une por **nome de coluna** (mais seguro que `union` que une por posição).

### Cell 5 — Persistir como Delta

```python
(df_bronze
    .write
    .mode("overwrite")
    .option("overwriteSchema", "true")
    .saveAsTable("bronze_yellow_trips_raw"))
```

- `.write` — entra no modo de escrita.
- `.mode("overwrite")` — se a tabela já existe, sobrescreve. (`"append"` adicionaria.)
- `.option("overwriteSchema", "true")` — se o schema mudou, atualiza.
- `.saveAsTable("nome")` — cria uma Managed Delta Table.

### Cell 6 — Lookup das zonas

```python
urllib.request.urlretrieve(zone_url, zone_path)
```

Diferente do Cell 2 que usou `requests`, aqui usei `urllib.request` (lib padrão sem precisar de instalação). Para arquivos pequenos (CSV de 16 KB), tanto faz.

```python
df_zones = (
    spark.read
         .option("header", "true")
         .option("inferSchema", "true")
         .csv("Files/...")
)
```

- `header=true` — primeira linha é cabeçalho.
- `inferSchema=true` — Spark detecta o tipo de cada coluna (Int, String etc.). Para CSV pequeno funciona bem. Para CSV grande, melhor passar schema explícito.

### Cell 7 — Validações com PySpark Functions

```python
from pyspark.sql import functions as F

(spark.read.table("bronze_yellow_trips_raw")
    .groupBy(
        F.year("tpep_pickup_datetime").alias("year"),
        F.month("tpep_pickup_datetime").alias("month")
    )
    .count()
    .orderBy("year", "month")
    .show(40))
```

- `F.year(col)` — extrai o ano de um timestamp.
- `F.month(col)` — extrai o mês.
- `.groupBy(...)` + `.count()` — agrupa e conta.
- `.orderBy(...)` — ordena.
- `.show(40)` — exibe 40 linhas.

---

## 3. Notebook 02_silver_clean (Spark SQL)

### Cell 2 — A célula central, dissecada

A query principal usa **CTEs** (Common Table Expressions). Pense em CTE como "subquery nomeada que você define no topo e usa abaixo". Sintaxe:

```sql
WITH base AS (
    SELECT ... FROM bronze
),
enriched AS (
    SELECT *, derivacao... FROM base
)
SELECT *, flag... FROM enriched;
```

Cada CTE é um passo lógico:

**Step 1: `base` — filtros e renomeação**

```sql
SELECT
    VendorID                      AS vendor_id,
    tpep_pickup_datetime          AS pickup_datetime,
    ...
    (UNIX_TIMESTAMP(tpep_dropoff_datetime) - 
     UNIX_TIMESTAMP(tpep_pickup_datetime)) / 60.0  AS duration_min
FROM bronze_yellow_trips_raw
WHERE tpep_pickup_datetime >= '2022-01-01'
  AND tpep_pickup_datetime <  '2025-01-01'
  AND trip_distance > 0
  AND fare_amount  >= 0
  AND tpep_dropoff_datetime > tpep_pickup_datetime
```

**Funções SQL usadas:**
- `UNIX_TIMESTAMP(timestamp)` — converte timestamp para segundos desde 1970. Subtraindo dois e dividindo por 60 → duração em minutos.
- `AS xxx` — renomeia.
- `WHERE` — aplica filtros de qualidade.

**Step 2: `enriched` — derivações**

```sql
SELECT
    base.*,
    CASE 
        WHEN duration_min > 0 THEN trip_distance_mi / (duration_min / 60.0)
        ELSE NULL
    END AS avg_speed_mph,
    HOUR(pickup_datetime)         AS pickup_hour,
    DAYOFWEEK(pickup_datetime)    AS pickup_dow,
    ...
FROM base
```

**Conceitos:**
- `base.*` — todas as colunas da CTE `base`.
- `CASE WHEN ... THEN ... ELSE ... END` — if/else em SQL.
- `HOUR(timestamp)` — extrai a hora (0-23).
- `DAYOFWEEK(timestamp)` — extrai dia da semana (1=Sunday, 7=Saturday em Spark SQL).
- `DATE_FORMAT(timestamp, 'EEEE')` — formata data; `'EEEE'` retorna nome completo do dia ("Monday", "Tuesday"...).

**Por que o `day_part` usa `CASE` em vez de `IIF`?**

`CASE` é SQL padrão e funciona em qualquer engine. `IIF` é dialeto. Sempre que possível, prefira `CASE` para portabilidade.

**Step 3: flag de anomalia**

```sql
SELECT *,
    CASE
        WHEN duration_min     > 360  THEN 1
        WHEN trip_distance_mi > 200  THEN 1
        WHEN avg_speed_mph    > 80   THEN 1
        WHEN total_amount     > 1000 THEN 1
        ELSE 0
    END AS is_anomalous
FROM enriched
```

**Por que não usar `OR` num único `CASE WHEN`?**

Funcionaria igual. Separei em múltiplos `WHEN` para deixar **explícito quais regras** estão flagrando. Em produção poderia adicionar uma coluna `anomaly_reason` para rastrear motivo, mas para o teste essa simplicidade basta.

### CREATE TABLE — sintaxe Delta

```sql
CREATE OR REPLACE TABLE silver_yellow_trips_clean
USING DELTA
PARTITIONED BY (year, month)
AS
SELECT ...;
```

- `CREATE OR REPLACE` — se a tabela existe, dropa e recria.
- `USING DELTA` — força formato Delta (em Fabric Lakehouse já é o default).
- `PARTITIONED BY (year, month)` — divide em subpastas por ano/mês (lê só o que precisa nas queries).
- `AS SELECT ...` — popula a tabela com o resultado da query (CTAS — Create Table As Select).

### Cell 3 — Subquery scalar

```sql
SELECT
    (SELECT COUNT(*) FROM bronze_yellow_trips_raw)  AS bronze_rows,
    (SELECT COUNT(*) FROM silver_yellow_trips_clean) AS silver_rows,
    ...
```

`(SELECT COUNT(*) FROM ...)` é uma **subquery scalar** — retorna 1 linha × 1 coluna, que é usada como valor. Útil para juntar contagens de tabelas diferentes na mesma linha de resultado.

### Cell 4 — Pattern de breakdown com `SUM(CASE)`

```sql
SELECT
    SUM(CASE WHEN <regra1> THEN 1 ELSE 0 END) AS count_regra1,
    SUM(CASE WHEN <regra2> THEN 1 ELSE 0 END) AS count_regra2,
    ...
FROM bronze_yellow_trips_raw;
```

Esse é um **padrão clássico de SQL** para contar quantas linhas satisfazem cada condição numa única passagem pela tabela. Equivalente a múltiplos `COUNT(*) WHERE`, mas roda em 1 scan só — muito mais eficiente.

### Cell 5 — `GROUP BY` simples

```sql
SELECT
    is_anomalous,
    COUNT(*)                                     AS rows,
    ROUND(COUNT(*) * 100.0 / 
        (SELECT COUNT(*) FROM tabela), 4)        AS pct
FROM silver_yellow_trips_clean
GROUP BY is_anomalous;
```

- `GROUP BY is_anomalous` — agrupa por valor (0 ou 1).
- `COUNT(*)` aplicado por grupo.
- A subquery scalar `(SELECT COUNT(*) FROM tabela)` traz o total para calcular o percentual.

### Cell 6 — Agregações múltiplas

```sql
SELECT
    year, month,
    COUNT(*)                          AS trips,
    ROUND(AVG(duration_min), 2)       AS avg_duration_min,
    ROUND(AVG(trip_distance_mi), 2)   AS avg_distance_mi,
    ROUND(AVG(total_amount), 2)       AS avg_total_usd
FROM silver_yellow_trips_clean
GROUP BY year, month
ORDER BY year, month;
```

- `AVG(col)` — média.
- `ROUND(valor, 2)` — arredonda para 2 casas decimais.
- Múltiplas agregações na mesma query — Spark executa todas em paralelo no mesmo scan.

### Cell 7 — Filtro + Order + Limit

```sql
SELECT col1, col2, ...
FROM silver_yellow_trips_clean
WHERE is_anomalous = 1
ORDER BY total_amount DESC
LIMIT 20;
```

Padrão "top N por critério". `LIMIT 20` corta no top 20.

---

## 4. Notebook 03_gold_model (Spark SQL + PySpark misto)

Esse notebook é o mais variado: combina Spark SQL (dimensões), PySpark (APIs externas) e SQL com joins complexos (fact_trips).

### 4.1 Padrões SQL novos usados

#### Sequence + explode para gerar calendário

```sql
WITH date_range AS (
    SELECT explode(sequence(
        to_date('2022-01-01'),
        to_date('2024-12-31'),
        interval 1 day
    )) AS full_date
)
SELECT ... FROM date_range
```

- `sequence(start, end, step)` — gera um array de valores (datas, números) entre start e end.
- `explode(array)` — transforma um array em múltiplas linhas (uma por elemento).
- Resultado: 1.096 linhas, uma por dia de 2022 a 2024.

Esse padrão dispensa criar tabela auxiliar de calendário — gera no momento.

#### VALUES table constructor

```sql
SELECT * FROM VALUES
    (1, 1, 'Creative Mobile Technologies'),
    (2, 2, 'Curb Mobility'),
    ...
AS dim_vendor(vendor_key, vendor_id, vendor_name);
```

- `VALUES (...), (...), ...` — cria uma tabela inline com as tuplas listadas.
- `AS tablename(col1, col2, ...)` — nomeia tabela e colunas.

Substitui múltiplos `INSERT INTO`. Útil para dimensões pequenas com valores fixos.

#### ROW_NUMBER() OVER para surrogate keys

```sql
SELECT
    ROW_NUMBER() OVER (ORDER BY z.LocationID) AS zone_key,
    ...
FROM bronze_taxi_zone_lookup z
```

- `ROW_NUMBER()` é uma **window function** que numera linhas sequencialmente.
- `OVER (ORDER BY ...)` define a ordem de numeração.
- Resultado: cada linha ganha um inteiro único (1, 2, 3...) — perfeito para surrogate key.

Por que surrogate key e não usar o `location_id` natural? Em star schema clássico, surrogate keys: (a) são imutáveis mesmo se a chave de negócio mudar, (b) economizam espaço (int pequeno), (c) facilitam slowly changing dimensions.

#### LEFT JOIN + COALESCE para tolerância a dados sujos

```sql
SELECT
    COALESCE(dv.vendor_key, 5) AS vendor_key
FROM silver_yellow_trips_clean s
LEFT JOIN dim_vendor dv ON dv.vendor_id = s.vendor_id;
```

- `LEFT JOIN` — preserva todas as linhas do fato, mesmo as que não casam com a dimensão.
- `COALESCE(a, b)` — retorna `a` se não-nulo, senão `b`.

Combinado: se o `vendor_id` do fato não existe em `dim_vendor`, o LEFT JOIN dá NULL → COALESCE substitui pela chave do "Unknown" na dimensão. Garante que **toda linha do fato tem dimensão**, mesmo com dados sujos.

#### CTAS com PARTITIONED BY

```sql
CREATE OR REPLACE TABLE fact_trips
USING DELTA
PARTITIONED BY (year, month)
AS
SELECT ... FROM silver_yellow_trips_clean s ...
```

- `CREATE TABLE ... AS SELECT` (CTAS) — cria e popula em uma operação.
- `PARTITIONED BY (year, month)` — divide os arquivos em subpastas `year=YYYY/month=MM/`.

Vantagem: queries com filtro temporal (90% das queries de BI) só leem partições relevantes.

### 4.2 Padrões PySpark novos usados

#### Criar DataFrame a partir de Row list

```python
from pyspark.sql import Row

data = [
    Row(borough="Manhattan", total_population=1629153, ...),
    Row(borough="Brooklyn",  total_population=2561225, ...),
]
df = spark.createDataFrame(data)
```

- `Row(col=value, ...)` — fábrica de linha.
- `spark.createDataFrame(lista_de_rows)` — converte em DataFrame.

Útil para dados hardcoded (reference data, lookups manuais, test fixtures).

#### withColumn + when/otherwise (CASE em PySpark)

```python
from pyspark.sql import functions as F

df = df.withColumn(
    "weather_category",
    F.when(F.col("snowfall_mm") > 50,                  "Snow")
     .when(F.col("precipitation_mm_tenths") > 250,     "Heavy Rain")
     .when(F.col("precipitation_mm_tenths") > 0,       "Rain")
     .otherwise("Clear")
)
```

Equivalente PySpark do `CASE WHEN` do SQL. Cada `.when()` é uma condição encadeada.

#### Pivot para transformar long → wide

```python
df_wide = (df_long
    .groupBy("date_only")
    .pivot("datatype", ["TMAX", "TMIN", "PRCP"])
    .agg(F.first("value"))
)
```

- `.groupBy(col)` — agrupa por uma coluna.
- `.pivot(col, values)` — rotaciona: cada valor da coluna `col` vira uma nova coluna.
- `.agg(F.first(...))` — para cada par (grupo, valor pivotado), retorna o primeiro registro.

Resultado: dado em "long format" (várias linhas por dia, uma por métrica) vira "wide format" (1 linha por dia, várias colunas).

Por que importa? APIs costumam retornar long format. BI consome wide format. Pivot é a ponte.

#### API call paginado com retry

```python
import requests, time

def fetch_year(year):
    records = []
    offset = 1
    while True:
        params = {"startdate": f"{year}-01-01", "enddate": f"{year}-12-31",
                  "limit": 1000, "offset": offset, ...}
        r = requests.get(url, params=params, headers=headers, timeout=60)
        if r.status_code != 200:
            print(f"HTTP {r.status_code}: {r.text[:300]}")
            r.raise_for_status()
        results = r.json().get("results", [])
        if not results:
            break
        records.extend(results)
        if len(results) < 1000:
            break
        offset += 1000
        time.sleep(0.3)   # respeita rate limit
    return records
```

Padrão de pipeline robusto para consumir REST API com paginação:
- Loop while True com break condition.
- Verificação de status HTTP.
- Print do body em caso de erro (debugging).
- Sleep entre requests (rate limiting).
- Break se a página retornou menos que o limit (sinal de última página).

### 4.3 Por que enriquecer dim_date com weather (e não criar dim_weather)?

Tinha duas opções:
1. **Criar `dim_weather`** separada com `weather_key` e relacionar via `weather_key` no fato.
2. **Enriquecer `dim_date`** adicionando colunas de clima diretamente.

Escolhi #2 porque:
- Clima é **propriedade de uma data**, não entidade própria.
- 1 dia = 1 medição de clima, não há dimensão N:1 sentido.
- Reduz joins no Power BI (1 join em vez de 2).
- Field Parameters do PBI funcionam melhor com dimensões "ricas" (muitas colunas).

Trade-off: `dim_date` fica mais larga (~20 colunas), mas com apenas 1.096 linhas isso é irrelevante.

---

## 5. Power BI Semantic Model (dia 01/06)

Camada de modelagem que conecta o Lakehouse Gold ao Power BI Desktop. Aqui o "modelo" deixa de ser conceito SQL e vira **modelo semântico** — tabular, com relacionamentos, hierarquias e medidas DAX.

### 5.1 DirectLake vs Live Connection vs Composite Model

Quando você cria um semantic model no Fabric e conecta o Power BI Desktop, três modos são possíveis:

| Modo | O que faz | Pode editar no PBI Desktop? |
|---|---|---|
| **Live connection** (padrão) | Lê só a metadata do semantic model do Fabric | ❌ Não. Modelo é read-only. |
| **DirectLake** | Lê Delta direto do OneLake sem importar | ❌ Também read-only no PBI Desktop |
| **Composite Model (DirectQuery + remote)** | Modelo local em DirectQuery sobre o semantic model | ✅ Sim — pode adicionar medidas, relações, etc. |

No nosso caso, ao clicar em **"Fazer alterações neste modelo"** no PBI Desktop, ele converteu de Live para **Composite**. Os dados continuam morando no Lakehouse (não são importados pro `.pbix`), mas agora podemos adicionar uma camada local de DAX e relacionamentos.

Trade-off arquitetural: leve perda de performance vs DirectLake puro, mas ganho de flexibilidade pra customizações sem precisar voltar pro Fabric a cada mudança.

### 5.2 Relacionamentos no Power BI vs no Fabric

Os 7 relacionamentos do star schema foram criados **no editor web do Fabric** (semantic model) — não no PBI Desktop. Razão: o diálogo "Novo relacionamento" do PBI Desktop tem um bug conhecido em composite models recém-criados (dropdown da segunda tabela fica vazio).

Workflow correto:
1. Cria relação no Fabric semantic model editor (`Gerenciar relações` → `Novo`).
2. Volta no PBI Desktop e clica em **Página Inicial → Atualizar** (refresh do composite model).
3. As relações aparecem no Model view do PBI.

Todas as 7 relações são **Many-to-One** com direção single. O modelo é star schema clássico.

### 5.3 `dim_date` marcada como Date Table

Uma tabela tem que ser explicitamente "marcada como tabela de data" pra que as funções de Time Intelligence (TOTALYTD, SAMEPERIODLASTYEAR, DATESINPERIOD) funcionem.

Procedimento:
1. Painel **Dados** → clica direito em `dim_date` → **Marcar como tabela de data**.
2. Escolhe a coluna `full_date` (a única que é DATE puro, sem horas).

DAX agora reconhece a `dim_date` como a tabela de calendário canônica do modelo.

### 5.4 Hierarquias

Hierarquias agrupam colunas relacionadas pra criar drill-down nos visuais. Foram criadas 4:

- **Date Hierarchy** (`dim_date`): Year → Quarter → Month → Day
- **Pickup Zone Hierarchy** (`dim_zone`): Borough → Service Zone → Zone Name
- **Dropoff Zone Hierarchy** (`dim_zone_dropoff`): Dropoff Borough → Dropoff Service Zone → Dropoff Zone Name
- **Time Hierarchy** (`dim_time`): Day Part → Hour

Quando arrasta a hierarquia pra um visual, o usuário pode dar drill-down (Year → Quarter → Month) com botões de navegação no visual.

### 5.5 Tabela `_Measures` (medidas organizadas)

Boa prática: criar uma tabela vazia chamada `_Measures` (underscore na frente faz aparecer no topo do painel) e armazenar todas as medidas DAX nela.

Vantagens:
- Separa medidas dos campos reais.
- Facilita encontrar quando o modelo cresce.
- Convenção comum em projetos profissionais de Power BI.

Como criar: **Página Inicial → Inserir dados → tabela com 1 coluna vazia → nomeia `_Measures` → carrega → oculta a coluna**.

### 5.6 As 10 medidas base — padrões DAX

Todas em `_Measures` com sintaxe consistente:

```dax
Total Trips = SUM ( fact_trips[trip_count] )
```

Padrão `SUM` puro sobre coluna do fato. `trip_count` é sempre 1, então `SUM(trip_count) = COUNT(trip_count) = número de linhas`.

```dax
Trips per Day = 
DIVIDE ( [Total Trips], DISTINCTCOUNT ( dim_date[full_date] ) )
```

Padrão `DIVIDE(numerator, denominator)` — sempre prefira `DIVIDE` em vez de `/` porque ele lida com divisão por zero retornando BLANK (ou um valor alternativo se passar o 3º parâmetro).

`DISTINCTCOUNT( dim_date[full_date] )` conta o número de dias **no contexto atual do filtro**. Se o filtro é só 2024, retorna ~366 dias.

```dax
Avg Fare = DIVIDE ( [Total Revenue], [Total Trips] )
```

Referência a outras medidas com `[Nome]` — DAX resolve a referência dinamicamente no contexto do visual.

### 5.7 Esconder PKs/FKs e business IDs

Boas práticas Kimball aplicadas:
- **Surrogate keys (`*_key`)**: ocultar permanentemente. Uso interno do modelo, usuário final não deve ver.
- **Business IDs (`*_id`)**: ocultar do painel, mas mantidos no modelo. Servem como audit trail e ponte pra joins externos.

No painel **Dados** do PBI Desktop, clica direito na coluna → **Ocultar na exibição de relatório**. Total: 17 colunas ocultadas no nosso modelo.

---

## 6. DAX Advanced (dia 02/06)

Os 3 recursos mais sofisticados do DAX moderno, que diferenciam dashboard amador de profissional.

### 6.1 Calculation Group "Time Analytics"

**O que é**: grupo de itens DAX (calculation items) que aplicam time intelligence em **qualquer medida selecionada**, em runtime. Em vez de duplicar cada medida em 7 versões (Total Revenue, Total Revenue YoY, Total Revenue YTD, etc.), você cria 1 grupo com 7 itens, e o usuário escolhe o item via slicer.

**Sintaxe** (criada no Tabular Editor 2, salva via TMDL):

```dax
// Calculation Item: Current
SELECTEDMEASURE ()

// Calculation Item: PY (Prior Year)
CALCULATE (
    SELECTEDMEASURE (),
    SAMEPERIODLASTYEAR ( dim_date[full_date] )
)

// Calculation Item: YoY (Year over Year delta)
SELECTEDMEASURE () -
CALCULATE (
    SELECTEDMEASURE (),
    SAMEPERIODLASTYEAR ( dim_date[full_date] )
)

// Calculation Item: YoY %
VAR _Current = SELECTEDMEASURE ()
VAR _PY = CALCULATE ( SELECTEDMEASURE (), SAMEPERIODLASTYEAR ( dim_date[full_date] ) )
RETURN DIVIDE ( _Current - _PY, _PY )

// Calculation Item: YTD (Year to Date)
TOTALYTD ( SELECTEDMEASURE (), dim_date[full_date] )

// Calculation Item: PYTD (Prior Year to Date)
CALCULATE (
    TOTALYTD ( SELECTEDMEASURE (), dim_date[full_date] ),
    SAMEPERIODLASTYEAR ( dim_date[full_date] )
)

// Calculation Item: MAT (Moving Annual Total)
CALCULATE (
    SELECTEDMEASURE (),
    DATESINPERIOD (
        dim_date[full_date],
        MAX ( dim_date[full_date] ),
        -12,
        MONTH
    )
)
```

**Conceito-chave**: `SELECTEDMEASURE()` é um placeholder. Quando o usuário arrasta `[Total Revenue]` num visual e seleciona "YoY" no slicer do Calc Group, o engine substitui `SELECTEDMEASURE()` por `[Total Revenue]` em runtime.

**Format String dinâmico**: para itens como `YoY %` que precisam aparecer como percentual independente da medida base, configura no item:
```dax
"0.00%;-0.00%;0.00%"
```

Isso sobrescreve o formato natural da medida quando esse item está ativo.

**Ordinal**: campo zero-based que define ordem dos itens no slicer (0, 1, 2, 3...). Sem isso, ficam em ordem alfabética.

**Onde criar**: Tabular Editor 2 (TE2) é a ferramenta padrão. PBI Desktop não tem UI nativa pra criar Calc Groups (precisa de TE2). Workflow:
1. PBI Desktop → **Ferramentas externas → Tabular Editor**.
2. TE2 abre conectado ao modelo.
3. **Tables** (clica direito) → **Create New → Calculation Group**.
4. Cria os Calculation Items dentro.
5. **File → Save** no TE2 — propaga as mudanças pro PBI Desktop.

### 6.2 Field Parameters

**O que é**: tabela "metadata" que permite o usuário escolher dinamicamente qual **campo** (medida ou coluna) está sendo usado num visual.

Diferente do Calc Group que escolhe **transformação** sobre uma medida, o Field Parameter escolhe **qual medida ou dimensão** entra no visual.

**Como criar**: PBI Desktop → **Modelagem → Novo parâmetro → Campos** (Fields).

Diálogo:
- Nome: `FP_Metric`
- Lista de medidas a expor: `[Total Trips]`, `[Total Revenue]`, etc.
- Marca "Adicionar e segmentar dados nesta página" → cria slicer automático.

Por baixo dos panos, o PBI cria uma **tabela calculada** com sintaxe TMDL especial:

```dax
FP_Metric = {
    ("Trips",    NAMEOF ( '_Measures'[Total Trips] ),    0),
    ("Revenue",  NAMEOF ( '_Measures'[Total Revenue] ),  1),
    ("Distance", NAMEOF ( '_Measures'[Total Distance] ), 2),
    ("Duration", NAMEOF ( '_Measures'[Avg Duration] ),   3),
    ("Tip",      NAMEOF ( '_Measures'[Total Tip] ),      4)
}
```

`NAMEOF()` retorna referência tipada à medida — diferente de `"Total Trips"` (string) ou `[Total Trips]` (medida real). É o que permite o "switch" dinâmico no visual.

Foram criados 2 Field Parameters:
- **FP_Metric**: alterna entre 5 medidas (Trips, Revenue, Distance, Duration, Tip).
- **FP_Dimension**: alterna entre 5 colunas categóricas (Borough, Zone, Vendor, Payment, Day Part).

**Uso no visual**: arrasta `FP_Metric[FP_Metric]` no eixo Y e `FP_Dimension[FP_Dimension]` no eixo X. Dois slicers permitem combinações 5 × 5 = **25 análises possíveis com 1 visual**.

### 6.3 UDFs (DAX User-Defined Functions)

**O que é**: funções DAX nomeadas e reutilizáveis. Recurso preview do Power BI 2025+.

**Sintaxe** (via DAX Query View, NOT via Modelagem ribbon):

```dax
DEFINE
    FUNCTION ClassifyTripDuration = ( minutes ) =>
    SWITCH (
        TRUE (),
        minutes < 5,   "Very Short",
        minutes < 15,  "Short",
        minutes < 30,  "Medium",
        minutes < 60,  "Long",
        minutes < 180, "Very Long",
                       "Outlier"
    )

EVALUATE
{ ClassifyTripDuration ( 25 ) }
```

**Gotcha 1 — Sem aspas no nome ao DEFINE**: a documentação inicial sugere `FUNCTION 'ClassifyTripDuration' = ...` mas isso retorna erro de sintaxe. O correto é **sem aspas** no DEFINE.

**Gotcha 2 — Tipos não obrigatórios**: a anotação `: NUMERIC` que aparece em alguns docs não é válida em todas as builds. Mais seguro deixar **sem tipo** e deixar o DAX inferir.

**Gotcha 3 — BLANK → 0 coercion**: em comparações numéricas, `BLANK < 5` é `TRUE` (porque BLANK é coerced para 0). Isso fez aparecer linhas vazias classificadas como "Very Short" / "Economy" na tabela de teste. Fix:

```dax
DEFINE
    FUNCTION ClassifyTripDuration = ( minutes ) =>
    IF (
        ISBLANK ( minutes ),
        BLANK (),
        SWITCH ( TRUE (), ... )
    )
```

**Como salvar**: depois do `EVALUATE` retornar resultado correto, clica em **"Atualizar o modelo com alterações"** no canto superior direito da DAX Query View. A função fica salva no modelo permanentemente.

**Por que isso vale na entrevista**: UDFs são a evolução natural do DAX — antes você duplicava lógica em N medidas, agora você define em 1 lugar. Mesmo padrão de DRY que se vê em Python ou SQL com funções/procedures.

As 3 UDFs criadas:

| Função | Inputs | Output | Uso |
|---|---|---|---|
| `ClassifyTripDuration` | `minutes` | Very Short / Short / Medium / Long / Very Long / Outlier | Categoriza duração de trip |
| `ClassifyFareTier` | `fare` | Economy / Standard / Premium / Luxury / Anomalous | Categoriza tier de tarifa |
| `ComputeRevenuePerMile` | `revenue, distance` | `DIVIDE(rev, dist, 0)` | Métrica derivada com guard |

### 6.4 Como Calc Group + Field Parameter + UDF se combinam

Os 3 recursos são complementares, não substitutos:

- **Field Parameter** escolhe **qual medida** está no visual.
- **Calc Group** transforma **a medida escolhida** com Time Intelligence.
- **UDF** encapsula **lógica reutilizável de classificação**.

Exemplo combinado:
1. Usuário arrasta `FP_Metric[FP_Metric]` num gráfico → escolhe `Total Revenue` no slicer.
2. Adiciona `Time Analytics[Time Analytics]` em outro slicer → escolhe `YoY %`.
3. Gráfico mostra: "YoY % de Total Revenue por Borough".
4. Adiciona coluna `[Avg Fare Tier]` em outra visualização → mostra classificação textual.

Tudo isso **sem criar uma única medida adicional**. É o que diferencia dashboard "código duplicado" de dashboard "modelo enxuto".

---

## 7. Conceitos avançados de Power BI

Seção dedicada a entender as **escolhas arquiteturais** do projeto. Cada conceito tem: definição, por que usamos, trade-offs e como aparece no nosso modelo.

### 7.1 RLS (Row-Level Security)

**O que é**: filtro automático aplicado ao modelo baseado em **papel do usuário** (role). Cada role tem regras DAX que limitam quais linhas do modelo o usuário enxerga. O filtro é transparente — visuais "se adaptam" sem o usuário saber.

**Sintaxe**: regra DAX em formato de expressão booleana sobre uma coluna de dimensão. Exemplo:
```
[borough] = "Manhattan"
```

**Por que usamos**: simula cenário corporativo onde "manager regional só vê sua região". Em supply chain 3M, equivale a "site manager só vê sua planta" ou "regional director só vê sua territory".

**Onde configurado**: no semantic model do **Fabric**, não no .pbix local. Razão arquitetural: governança fica na **fonte de verdade dos dados**. Se configurássemos no .pbix, outro relatório consumindo o mesmo semantic model perderia a proteção.

**As 6 roles criadas**:
- `Manhattan Manager` → `dim_zone[borough] = "Manhattan"`
- `Brooklyn Manager` → `dim_zone[borough] = "Brooklyn"`
- `Queens Manager` → `dim_zone[borough] = "Queens"`
- `Bronx Manager` → `dim_zone[borough] = "Bronx"`
- `Staten Island Manager` → `dim_zone[borough] = "Staten Island"`
- `Global Manager` → sem filtro (vê tudo)

**Limitação do composite model**: quando o PBI Desktop está conectado a um semantic model remoto via composite, o "Exibir como" do PBI Desktop **não mostra as roles do remote**. Solução pra demonstrar: usar a feature "Test as role" do Power BI Service após publicar o relatório.

**Diferença vs OLS (Object-Level Security)**:
- **RLS** filtra **linhas** (ex.: usuário X só vê dados de Manhattan).
- **OLS** filtra **objetos/colunas** (ex.: usuário X não vê coluna "salary").
- OLS exige Tabular Editor pra configurar (não tem UI no PBI Desktop).

**Padrão DAX equivalente** (sem RLS, mas mesma lógica) — útil quando quer **demonstrar visualmente** o efeito de uma role:
```dax
Manhattan Manager View = 
CALCULATE ( [Total Revenue], dim_zone[borough] = "Manhattan" )
```

### 7.2 Star Schema vs Snowflake vs Flat

Três jeitos de organizar tabelas num modelo dimensional:

**Star Schema** (nosso modelo):
- 1 tabela fato + N dimensões diretamente conectadas.
- Dimensões **denormalizadas** (toda info numa única tabela).
- Visualmente: estrela com o fato no centro.

```
       dim_date
          │
dim_zone──fact_trips──dim_vendor
          │
       dim_time
```

**Snowflake**:
- Dimensões **normalizadas** em sub-dimensões.
- Ex.: `dim_zone` → `dim_borough` → `dim_region`.
- Mais joins, modelo mais "normalizado" como banco transacional.

**Flat (One Big Table)**:
- Tudo em 1 tabela única, sem dimensões separadas.
- Comum em dataframes Pandas/Polars, mas péssimo pra BI.

**Por que escolhemos Star Schema**:
1. **Performance Vertipaq**: o engine do Power BI é otimizado pra star schema. Joins são pré-calculados.
2. **Field Parameters e Calc Groups**: funcionam melhor com dimensões denormalizadas.
3. **Manutenibilidade**: modelo mais simples = menos joins = menos erros.
4. **Padrão Kimball**: é o padrão de mercado pra BI desde 1996.

**Trade-off**: dimensões denormalizadas têm dado repetido (ex.: borough "Manhattan" aparece em 30 zonas). Storage maior, mas Vertipaq comprime tão bem que é irrelevante.

### 7.3 Composite Model (DirectQuery + remote semantic model)

**O que é**: tipo de modelo Power BI que **combina duas camadas**:
1. **Remote layer**: tabelas que vêm de um semantic model externo (em DirectLake ou DirectQuery).
2. **Local layer**: medidas, relacionamentos, hierarquias e roles criadas no `.pbix` localmente.

**Como surge**: quando você está em Live connection a um semantic model do Fabric e clica em "Fazer alterações neste modelo", o Power BI converte pra composite — adiciona uma camada local DirectQuery em cima das tabelas remotas.

**Vantagens**:
- Não duplica os 115M de linhas do fato no `.pbix` (continua via DirectQuery ao Lakehouse).
- Permite adicionar medidas, calc groups, hierarquias localmente sem precisar voltar pro Fabric.
- Permite combinar com outras fontes locais (ex.: planilha Excel + dados Fabric).

**Desvantagens**:
- Performance ligeiramente pior que DirectLake puro (queries vão via SQL endpoint).
- RLS configurado localmente ≠ RLS do remote — pode confundir.
- Não dá pra "voltar" pra Live connection puro depois (é mudança permanente no .pbix).

**Nossa decisão**: aceitar o composite model pra ter flexibilidade de DAX local. Para o critério "Front End" do teste, isso é mais flexível.

### 7.4 DirectLake vs DirectQuery vs Import

Três modos de conexão Power BI ↔ fonte de dados:

| Modo | Como funciona | Performance | Dados duplicados? | Quando usar |
|---|---|---|---|---|
| **Import** | Carrega dados pro arquivo `.pbix` em memória | Rápido (Vertipaq) | Sim — `.pbix` e fonte | Datasets até ~1GB |
| **DirectQuery** | Cada visual envia SQL à fonte | Lento (latência por query) | Não | Dados em tempo real ou >1GB |
| **DirectLake** | Vertipaq lê Delta direto do OneLake | Rápido (~Import) | Não | Fabric Lakehouse + dados grandes |

**Detalhes do DirectLake**:
- É o **novo modo** do Fabric (2023+).
- Funciona **apenas** com Delta tables no OneLake.
- Lê o Parquet do Delta nativamente — não precisa traduzir pra SQL.
- "Magia": tem a performance de Import mas sem importar.

**Por que DirectLake foi pensado pro nosso projeto**:
- 115M linhas no fato — Import duplicaria os dados (overhead de 1-2 GB no .pbix).
- DirectQuery seria lento demais com agregações sobre 115M rows.
- DirectLake é o sweet spot — performance de Import sem o overhead.

**Realidade**: no nosso caso ficamos em **composite (DirectQuery sobre o semantic model do Fabric)** porque precisamos adicionar medidas e RLS locais. Em produção otimizada, voltaríamos pra DirectLake puro depois.

**Comparação de latência (estimativa)**:
- Import: 100ms por visual
- DirectLake: 150-200ms
- DirectQuery: 500-2000ms
- Composite (nosso): 300-500ms

### 7.5 Theme JSON do Power BI

**O que é**: arquivo JSON que define **toda a aparência visual** do relatório — cores, fontes, paddings, bordas. Aplica uma vez, propaga pra todos os visuais.

**Estrutura básica**:
```json
{
  "name": "Nome do tema",
  "dataColors": ["#cor1", "#cor2", ...],
  "background": "#FFFFFF",
  "foreground": "#1F3864",
  "good": "#22A06B",
  "bad": "#D32F2F",
  "textClasses": {
    "title": { "fontFace": "Segoe UI", "fontSize": 18 }
  },
  "visualStyles": {
    "card": {
      "*": { ... configurações específicas pra Cards ... }
    }
  }
}
```

**Hierarquia de overrides**:
1. **dataColors**: paleta padrão de séries (1ª série usa cor 1, 2ª série usa cor 2, etc.).
2. **textClasses**: 4 níveis (`title`, `header`, `label`, `callout`) — define fonte default por contexto.
3. **visualStyles**: customizações específicas por tipo de visual.

**Por que usar JSON em vez de configurar visual a visual**:
- Consistência: 1 mudança no JSON → 50 visuais atualizam.
- Versionamento: tema entra no Git como código.
- Reproducibilidade: clonar o tema entre relatórios = identidade visual unificada.

**Nossa paleta**:
- Principal: azul corporativo (`#1F3864`) e azul médio (`#2E75B6`).
- Acentos: verde positivo (`#22A06B`), vermelho negativo (`#D32F2F`), laranja alerta (`#F29111`).
- Cinzas neutros (`#404040`, `#595959`, `#808080`) pra texto secundário.

### 7.6 Conditional Formatting — padrões DAX

**Conceito**: capacidade do Power BI de **mudar cor, ícone, texto ou tamanho** de uma célula baseado em regras dinâmicas. Substitui interpretação humana ("esse número está bom ou ruim?") por sinal visual instantâneo.

**4 técnicas principais**:

#### 7.6.1 Cor por valor de campo (via medida DAX)

Mais poderoso. Cria uma medida que retorna **string com código hexadecimal** baseado em lógica:

```dax
YoY Color = 
VAR _YoY = CALCULATE ( [Total Revenue], 'Time Analytics'[Time Analytics] = "YoY %" )
RETURN
    SWITCH (
        TRUE (),
        _YoY > 0.05,    "#22A06B",    -- verde forte
        _YoY > 0,       "#5B9BD5",    -- azul claro
        _YoY < -0.05,   "#D32F2F",    -- vermelho forte
        _YoY < 0,       "#F29111",    -- laranja
                        "#808080"     -- cinza neutro
    )
```

Aplica em: KPI cards (cor do callout value), barras de gráfico (Data Colors), célula de matriz (background color).

**Como aplicar no visual**:
1. Seleciona o visual → painel **Formatar visual** → encontra a propriedade de cor.
2. Clica no botão `fx` ao lado da cor.
3. Escolhe **Formatação por valor de campo** → seleciona a medida criada.

#### 7.6.2 Heat Map (gradient)

Matriz onde cada célula tem **background color em gradiente** baseado no valor.

Configuração via UI:
- Matriz → Formatar visual → Elementos da célula → Cor da tela de fundo → `fx`.
- Escolhe **Gradiente** → define cor mínima (claro) e máxima (escuro).

Resultado visual: matriz vira heat map. Padrões aparecem instantaneamente.

#### 7.6.3 Data Bars

Barras horizontais **embutidas** na célula, proporcionais ao valor.

Configuração via UI:
- Tabela → Formatar visual → Elementos da célula → Barras de dados.
- Define cor da barra (positiva e negativa).

Resultado: linha de tabela tem mini-gráfico visual sem ocupar espaço de coluna extra.

#### 7.6.4 Ícones (semáforo)

Ícone (🟢🟡🔴 ou setas) à esquerda/direita da célula baseado em regras.

Configuração via UI:
- Tabela → Formatar visual → Elementos da célula → Ícones.
- Define regras por range de valor + escolhe ícone.

**Padrão semáforo**:
- 0% até 0.1% → verde 🟢 (saudável)
- 0.1% até 0.3% → amarelo 🟡 (atenção)
- > 0.3% → vermelho 🔴 (crítico)

### 7.7 Tabular Editor 2 (External Tool)

**O que é**: ferramenta externa, gratuita, que permite editar o modelo semântico do Power BI via **interface mais poderosa** que a do PBI Desktop. Acessa o modelo via TMSL/TMDL (Tabular Model Definition Language).

**Por que usamos**: alguns recursos avançados **só podem ser criados via Tabular Editor**:
- **Calculation Groups**: PBI Desktop tem suporte limitado.
- **Calculation Items com Format String Expression**: precisa de TE2.
- **Edição em massa de medidas**: TE2 permite scripts C#.
- **Perspectivas**: PBI Desktop não tem UI pra isso.

**Como acessar**: PBI Desktop → faixa **Ferramentas externas** → clica no ícone do Tabular Editor. TE2 abre conectado ao modelo local do .pbix em runtime.

**Workflow**:
1. PBI Desktop aberto com `.pbix`.
2. Abre TE2 via "Ferramentas externas".
3. TE2 mostra TOM Explorer (estrutura do modelo) à esquerda.
4. Cria/edita objetos no painel.
5. **File → Save** no TE2 — propaga mudanças pro modelo local do PBI.
6. PBI Desktop pergunta "Reload model?" → aceita.

**TE2 (gratuito) vs TE3 (pago)**:
- TE2 cobre 90% dos casos.
- TE3 tem UI moderna, suporte a versionamento, debugging DAX.

---

## 8. Glossário rápido

| Termo | O que é |
|---|---|
| **DataFrame** | Tabela em memória do Spark |
| **Lazy evaluation** | Spark não executa até precisar (count, write, show) |
| **Delta Lake** | Formato de tabela ACID (Parquet + transaction log) |
| **Lakehouse** | Storage Fabric que combina arquivos + tabelas |
| **Managed Table** | Tabela cujo dado + metadado são gerenciados pelo Spark |
| **CTAS** | Create Table As Select — cria tabela e popula em um comando |
| **CTE** | Common Table Expression — subquery nomeada (WITH) |
| **Partition pruning** | Otimização que ignora partições não necessárias |
| **Idempotência** | Propriedade de operação que pode ser repetida sem efeito colateral |
| **Medallion** | Padrão Bronze (raw) → Silver (clean) → Gold (modeled) |
| **Schema drift** | Variação de schema entre fontes/lotes do mesmo dataset |
| **DirectLake** | Modo Power BI que lê Delta direto do OneLake sem cópia |
| **DirectQuery** | Modo Power BI que envia SQL ao banco a cada visual (não importa dados) |
| **Composite Model** | Modelo PBI que combina local + remoto (DirectQuery + Live) |
| **F-string** (Python) | `f"texto {variavel}"` — interpolação |
| **List comprehension** | `[x for x in lista]` — lista construída inline |
| **ThreadPoolExecutor** | Executa tarefas em paralelo em threads |
| **Calc Group** | Grupo de calculation items que aplica time intelligence em qualquer medida via slicer |
| **Calculation Item** | Item dentro de um Calc Group que define uma transformação DAX |
| **Field Parameter** | Tabela "metadata" que permite escolher dinamicamente qual campo entra no visual |
| **UDF (DAX)** | User-Defined Function — função DAX nomeada e reutilizável |
| **SELECTEDMEASURE()** | Placeholder em Calc Items que vira a medida ativa no contexto do visual |
| **NAMEOF()** | Função DAX que retorna referência tipada a coluna/medida (usado em Field Parameters) |
| **Date Table** | Tabela explicitamente marcada como calendário, habilita Time Intelligence |
| **Surrogate Key** | Chave artificial (`*_key`) gerada via ROW_NUMBER, substitui chave natural |
| **Tabular Editor 2** | Ferramenta externa pra editar modelo semântico (Calc Groups, UDFs via TMDL) |
| **TMDL** | Tabular Model Definition Language — formato de definição de modelo semântico |

---

## 9. DAX da Página 1 — medidas e gotchas (dia 4)

Esta seção documenta as medidas DAX criadas/refatoradas para finalizar a Página 1 — Executive Overview, junto com 2 gotchas críticos descobertos.

### 9.1 Medidas criadas/refatoradas

#### `Anomalous Trips`
Conta de viagens marcadas como anomalia pelo pipeline Silver.

```dax
Anomalous Trips =
CALCULATE(
    COUNTROWS(fact_trips),
    fact_trips[is_anomalous] = 1
)
```

**Por quê `= 1` e não `= TRUE()`**: ver gotcha 9.2.1.

---

#### `Anomaly Rate`
Razão entre viagens anômalas e total de viagens.

```dax
Anomaly Rate =
DIVIDE(
    [Anomalous Trips],
    [Total Trips],
    0
)
```

Formato: Percentual, 2 decimais. Resultado no contexto total: **0,16%** (180k anomalias / 115,7M trips).

---

#### `Anomaly Color` (refatorada)
Cor dinâmica para o callout value do card. Encapsula thresholds de severidade.

```dax
Anomaly Color =
VAR _Rate = [Anomaly Rate]
RETURN
SWITCH(
    TRUE(),
    ISBLANK(_Rate), "#6B7280",
    _Rate < 0.03, "#10B981",   -- verde: Healthy
    _Rate < 0.10, "#F59E0B",   -- amarelo: Warning
    "#EF4444"                  -- vermelho: Critical
)
```

**Por quê uma medida só pra cor**: separação de responsabilidades. Se um dia mudarem os thresholds (3%, 10%), troca em 1 lugar e propaga pra todas as visualizações que usam. Padrão "single source of truth" — vira ponto de discussão técnica em entrevista.

**Como aplicar no Power BI**: Card → Format pane → Visual → Callout value → Color → `fx` → Field value → `[Anomaly Color]`.

---

#### `Anomaly Subtitle`
Texto dinâmico que reage ao threshold + traduz o número absoluto.

```dax
Anomaly Subtitle =
VAR _Count = [Anomalous Trips]
VAR _Rate  = [Anomaly Rate]
VAR _CountFmt =
    IF(_Count >= 1000,
       FORMAT(_Count / 1000, "0") & "k",
       FORMAT(_Count, "#,##0"))
RETURN
SWITCH(
    TRUE(),
    ISBLANK(_Rate) || _Rate = 0, "No anomalies in period",
    _Rate < 0.03, _CountFmt & " flagged • Healthy",
    _Rate < 0.10, _CountFmt & " flagged • Warning",
    _CountFmt & " flagged • Critical"
)
```

**Bonito porque**: o usuário não precisa interpretar 0,16% — o texto traduz. "180k flagged • Healthy" é direto pro ponto.

---

#### `Avg Lead Time (min)`
Média da duração das viagens em minutos. KPI hero substituindo o redundante YoY Growth.

```dax
Avg Lead Time (min) =
AVERAGE(fact_trips[duration_min])
```

---

#### `Lead Time Subtitle` (versão final com locale forçado)

```dax
Lead Time Subtitle =
VAR _MaxYear = MAX(dim_date[year])
VAR _Current = CALCULATE([Avg Lead Time (min)], dim_date[year] = _MaxYear)
VAR _PY      = CALCULATE([Avg Lead Time (min)], dim_date[year] = _MaxYear - 1)
VAR _Delta   = DIVIDE(_Current - _PY, _PY)
RETURN
IF(
    ISBLANK(_PY) || _PY = 0,
    "Average trip duration",
    FORMAT(_Delta, "+0.0%;-0.0%", "en-US") & " vs " & (_MaxYear - 1)
)
```

**Padrão MAX(year) (ao invés de DATEADD)**: garante que mesmo no contexto "Todos selecionados" o subtítulo sempre compare o ano mais recente vs o anterior. É o mesmo padrão usado no `YoY % Revenue`. **Consistência interna do modelo.**

---

#### `Fare Subtitle`, `Trips Subtitle`, `Revenue Subtitle`

Mesmo padrão MAX(year) + IF fallback. Localização forçada com `"en-US"` no FORMAT (ver gotcha 9.2.2).

```dax
Fare Subtitle =
VAR _MaxYear = MAX(dim_date[year])
VAR _Current = CALCULATE([Avg Fare], dim_date[year] = _MaxYear)
VAR _PY      = CALCULATE([Avg Fare], dim_date[year] = _MaxYear - 1)
VAR _Delta   = DIVIDE(_Current - _PY, _PY)
RETURN
IF(
    ISBLANK(_PY) || _PY = 0,
    "Average ticket",
    FORMAT(_Delta, "+0.0%;-0.0%", "en-US") & " vs " & (_MaxYear - 1)
)
```

---

### 9.2 Gotchas críticos descobertos no dia 4

#### 9.2.1 — DAX não faz coerção INT ↔ BOOL

**Sintoma**: ao criar `Anomalous Trips` com `fact_trips[is_anomalous] = TRUE()`, o engine retornou:

> "Operações de comparação DAX não oferecem suporte à comparação de valores do tipo Integer com valores do tipo True/False. Considere usar a função VALUE ou FORMAT para converter um dos valores."

**Causa**: o pipeline Silver gravou a flag `is_anomalous` como **Integer (0/1)** no parquet (provavelmente via `CASE WHEN ... THEN 1 ELSE 0 END` no Spark SQL). DAX é estrito com tipos — não tem coerção implícita entre INT e BOOL como Python e SQL têm.

**Fix**: comparar com `= 1` em vez de `= TRUE()`.

```dax
-- ❌ ERRADO (quando is_anomalous é INT):
CALCULATE(COUNTROWS(fact_trips), fact_trips[is_anomalous] = TRUE())

-- ✅ CERTO:
CALCULATE(COUNTROWS(fact_trips), fact_trips[is_anomalous] = 1)
```

**Resposta de entrevista**: "DAX é estrito com tipos — não faz coerção implícita INT↔BOOL. Toda flag binária do lakehouse precisa ser comparada com `= 1` ou `= 0`, não com `= TRUE()`. Alternativa de longo prazo seria criar uma coluna calculada que retorne BOOL real, mas custa memória — não vale a pena pra uma flag binária. Em produção, eu padronizaria com uma medida helper `Is Anomalous Filter = fact_trips[is_anomalous] = 1` reutilizável em qualquer CALCULATE."

---

#### 9.2.2 — FORMAT respeita locale do client por padrão (`+0.0%` virou `+01%` em PT-BR)

**Sintoma**: subtítulo do card de Avg Fare mostrava `"-01% vs 2023"` em vez do esperado `"-1,0% vs 2023"`.

**Causa**: a fórmula original era `FORMAT(_Delta, "+0.0%;-0.0%")` (sem terceiro argumento). O Power BI Desktop em locale PT-BR interpretou o `.` (ponto) como **separador de milhar**, não decimal. Resultado: o `0.0` virou `00` (zero seguido de zero) e o número inteiro fica colado.

**Fix**: forçar locale via 3º argumento do FORMAT:

```dax
-- ❌ ERRADO (depende do locale do client):
FORMAT(_Delta, "+0.0%;-0.0%")

-- ✅ CERTO (locale-independent):
FORMAT(_Delta, "+0.0%;-0.0%", "en-US")
```

**Alternativa equivalente** (se preferir vírgula no display): `FORMAT(_Delta, "+0,0%;-0,0%", "pt-BR")`.

**Resposta de entrevista**: "Em um modelo global como o da 3M, com usuários em múltiplos países, FORMAT sem locale explícito é uma armadilha silenciosa. O mesmo `.pbix` pode renderizar de jeito diferente dependendo da configuração regional da máquina do consumidor. A boa prática é sempre passar o terceiro argumento de FORMAT — força consistência visual independente de onde o usuário abre o relatório. Documentei como convenção do projeto."

---

#### 9.2.3 — Use `MAX(year)` em vez de `DATEADD` para YoY robusto

**Cenário**: o subtítulo de Lead Time mostrava sempre "Average trip duration" (fallback) quando nenhum filtro estava aplicado, mesmo havendo 3 anos no modelo.

**Causa**: `CALCULATE([Avg Lead Time], DATEADD(dim_date[full_date], -1, YEAR))` em contexto multi-ano retorna BLANK porque o DATEADD não tem âncora clara para "ano anterior" quando todos os anos estão selecionados.

**Fix**: usar o padrão `MAX(dim_date[year])` para ancorar explicitamente o ano de comparação:

```dax
VAR _MaxYear = MAX(dim_date[year])
VAR _Current = CALCULATE([medida], dim_date[year] = _MaxYear)
VAR _PY      = CALCULATE([medida], dim_date[year] = _MaxYear - 1)
```

Funciona em qualquer contexto:
- Filtrado em 1 ano: `_MaxYear` = aquele ano.
- Filtrado em vários anos ou "Todos": `_MaxYear` = ano mais recente disponível.
- Filtrado em mês de 2024: `_MaxYear` = 2024, compara com 2023.

**Resposta de entrevista**: "DATEADD é elegante mas frágil em contexto multi-ano. O padrão MAX(year) com CALCULATE explícito é mais robusto — sempre compara o ano mais recente do contexto vs o anterior. É o mesmo padrão que usei no `YoY % Revenue`. Consistência interna do modelo: todo cálculo de YoY no projeto segue o mesmo template."

---

### 9.3 Resumo das 8 medidas da Página 1

| Medida | Tipo | Onde aparece |
|---|---|---|
| `Total Revenue` | Numérica | Card Revenue + Trendline + Borough + Tabela |
| `Total Trips` | Numérica | Card Trips + Tabela |
| `Avg Fare` | Numérica | Card Fare + Tabela |
| `Avg Lead Time (min)` | Numérica | Card Lead Time |
| `Anomalous Trips` | Numérica (helper) | Anomaly Subtitle |
| `Anomaly Rate` | Numérica (%) | Card Anomaly + helper p/ Anomaly Color |
| `Anomaly Color` | String (HEX) | fx na Callout Value do Anomaly card |
| `YoY Color Revenue` | String (HEX) | fx na Callout Value do Revenue card |
| `Revenue Subtitle` | String | Reference label do Revenue card |
| `Trips Subtitle` | String | Reference label do Trips card |
| `Fare Subtitle` | String | Reference label do Fare card |
| `Lead Time Subtitle` | String | Reference label do Lead Time card |
| `Anomaly Subtitle` | String | Reference label do Anomaly card |

---

## 10. DAX da Página 2 — Demand Deep Dive (dia 8)

> Página em 1920×1080. 4 visuais (heat map, sazonalidade, top zonas, explorador Field Parameters) + 5 KPIs. Aqui ficam as medidas novas e os gotchas resolvidos.

### 10.1 O padrão "argmax" em DAX — Peak Hour e Peak Day

A pergunta "qual a hora de pico?" não é `MAX(hour)` — isso daria a maior hora (23h). É a hora **que tem** mais viagens (argmax). Em DAX:

```dax
Peak Hour =
VAR _byHour = ADDCOLUMNS ( VALUES ( dim_time[hour_label] ), "@t", [Total Trips] )
RETURN MAXX ( TOPN ( 1, _byHour, [@t], DESC ), dim_time[hour_label] )
```

Linha a linha:
- `VALUES(dim_time[hour_label])` → as 24 horas visíveis no contexto atual.
- `ADDCOLUMNS(..., "@t", [Total Trips])` → tabela virtual: cada hora + suas viagens.
- `TOPN(1, ..., [@t], DESC)` → mantém só a linha de maior `@t` (a hora campeã).
- `MAXX(essa-1-linha, dim_time[hour_label])` → extrai o rótulo dessa hora.

`Peak Day` é idêntico trocando `hour_label` por `dim_date[day_name]`. Como tudo passa por `VALUES` no contexto atual, **as medidas respondem aos filtros**: filtrou Brooklyn, o pico recalcula.

### 10.2 Share e média diária

```dax
Rush Hour Trip % = DIVIDE ( CALCULATE ( [Total Trips], dim_time[is_rush_hour] = 1 ), [Total Trips] )
Weekend Trip %   = DIVIDE ( CALCULATE ( [Total Trips], dim_date[is_weekend] = 1 ), [Total Trips] )
Avg Daily Trips  = AVERAGEX ( VALUES ( dim_date[full_date] ), [Total Trips] )
```
- Flags sempre com `= 1`, nunca `TRUE()` — DAX não coage Integer↔Boolean (gotcha §9.2.1).
- `Avg Daily Trips`: itera cada data e tira a média → throughput diário médio (linguagem SC: unidades/dia).

### 10.3 Subtítulos dinâmicos + o gotcha do FORMAT com escala

Os 5 cards têm subtítulo de texto que muda com o filtro. Exemplo do Rush Hour:
```dax
Rush Hour Subtitle =
VAR _rushM = DIVIDE ( CALCULATE ( [Total Trips], dim_time[is_rush_hour] = 1 ), 1000000 )
RETURN FORMAT ( _rushM, "#,##0.0", "en-US" ) & "M in rush windows"
```
**Gotcha (resolvido no dia 8):** a 1ª versão usava a escala de milhar do FORMAT (`"#,##0.0,,"`) pra mostrar milhões. Saiu `35.930.072,0M` — a escala `,,` é frágil e não dividiu. Correção robusta: **dividir explícito por 1e6** e formatar o resultado. Sempre com 3º argumento `"en-US"` (senão ponto/vírgula sai em PT-BR).

### 10.4 `day_order_mon` — por que vive no Gold, não como calculated column

O heat map precisa das colunas em **Seg→Dom**, mas o Spark numera `day_of_week` com **domingo = 1**. A coluna de ordenação:
```sql
-- Gold, dentro do SELECT do dim_date
CASE WHEN day_of_week = 1 THEN 7 ELSE day_of_week - 1 END AS day_order_mon
```
**Por que no Gold e não "Nova coluna" no Power BI?** O modelo é **composite sobre DirectLake** (`sm_yellow_taxi_3m` remoto). Power BI **não permite calculated column em tabela remota via DirectQuery** — só medidas locais. Empurramos a lógica pro Gold: modelo fino, lógica centralizada (boa frase de entrevista).

Três gotchas que apareceram nesse caminho:
1. **`CREATE OR REPLACE TABLE dim_date` apagou as colunas de clima.** O `dim_date` é montado em 2 etapas (calendário + enrichment NOAA). Rodar só a 1ª recriou a tabela sem `snow_depth_mm` etc. → refresh quebrou ("column not found in delta table"). Lição: `CREATE OR REPLACE` é destrutivo; re-rodar o enrichment depois.
2. **DirectLake não puxa coluna nova sozinho.** Depois de adicionar `day_order_mon` no Delta, foi preciso sincronizar a coluna no modelo semântico (Edit tables / refresh do schema).
3. **Sort by column é no modelo semântico, não no Power BI.** Como `day_name` é coluna remota, o "Classificar por coluna" fica bloqueado no PBI Desktop (composite). Setar no modelo (Propriedades → Avançado) → herda em todas as páginas.

### 10.5 Resumo das medidas da Página 2

| Medida | Tipo | Onde |
|---|---|---|
| Peak Hour / Peak Day | String (argmax) | KPI cards |
| Avg Daily Trips | Número | KPI card |
| Rush Hour Trip % / Weekend Trip % | % | KPI / análise |
| Avg Daily Trips / Peak Day / Peak Hour / Rush Hour Subtitle | String | Reference labels |
| day_order_mon (coluna Gold) | Int | Sort de `day_name` |

---

## 11. DAX da Página 3 — Operations & Cost (dia 8)

### 11.1 Medidas de eficiência e custo
```dax
Avg Distance (mi)     = DIVIDE ( [Total Distance], [Total Trips] )
Avg Speed (mph)       = CALCULATE ( AVERAGE ( fact_trips[avg_speed_mph] ), fact_trips[is_anomalous] = 0 )
Avg Cost per Trip ($) = DIVIDE ( SUM ( fact_trips[total_amount] ), [Total Trips] )
Cost per Mile ($)     = ComputeRevenuePerMile ( [Total Revenue], [Total Distance] )   -- UDF ao vivo
```
- **`Avg Speed`** exclui anomalias (`is_anomalous = 0`) — senão viagens com speed >80 mph (erro de meter) inflam a média.
- **`Avg Distance`** hoje inclui tudo (5,09 mi, +20,4% YoY): parte é tendência real (viagens mais longas pós-pandemia), parte pode ser outlier. Versão "limpa" = replicar o padrão do Speed.
- **`Cost per Mile`** chama o UDF `ComputeRevenuePerMile` (`DIVIDE(rev, dist, 0)` com guard) — a feature UDF numa medida de produção, não de teste.

### 11.2 Componentes da tarifa (composição)
```dax
Avg Base Fare = DIVIDE ( SUM ( fact_trips[fare_amount] ), [Total Trips] )
Avg Tip       = DIVIDE ( SUM ( fact_trips[tip_amount] ),  [Total Trips] )
Avg Tolls     = DIVIDE ( SUM ( fact_trips[tolls_amount] ),[Total Trips] )
Avg Surcharges & Taxes =
DIVIDE (
    SUM ( fact_trips[extra_amount] ) + SUM ( fact_trips[mta_tax] )
      + SUM ( fact_trips[improvement_surcharge] ) + SUM ( fact_trips[congestion_surcharge] )
      + SUM ( fact_trips[airport_fee] ),
    [Total Trips]
)
```
As 4 somam ~$26,42 (= `Avg Cost per Trip`). Na barra empilhada mostram: ~69% base, ~19% surcharges/taxes, ~12% tip, tolls residual.

### 11.3 Subtítulos (template YoY)
K1 reaproveita `[Lead Time Subtitle]`; os outros 4 seguem o mesmo template trocando a medida base:
```dax
Avg Speed Subtitle =
VAR _y = MAX ( dim_date[year] )
VAR _cur = CALCULATE ( [Avg Speed (mph)], dim_date[year] = _y )
VAR _py  = CALCULATE ( [Avg Speed (mph)], dim_date[year] = _y - 1 )
RETURN IF ( ISBLANK ( _py ), "non-anomalous trips", "vs PY " & FORMAT ( DIVIDE ( _cur - _py, _py ), "+0.0%;-0.0%", "en-US" ) )
```
(idem `Avg Cost Subtitle` e `Cost per Mile Subtitle`.)

### 11.4 `duration_class` — limites = UDF + resultado validado
Coluna no Gold (regra do DirectLake em §10.4). Limites idênticos ao UDF `ClassifyTripDuration`: `<5 / <15 / <30 / <60 / <180 / else`, com guarda de NULL (`'0 · Unknown'`). Distribuição nos 115.756.175 (soma bateu): 78% em 5–30 min, Outlier 0,09%, zero nulos.

### 11.5 Ajustes de leitura (gotchas de visualização, dia 8)
1. **Scatter esticado por outliers** — média de distância por zona distorcida por viagens anômalas (eixo ia a 300mi). Fix: filtro `is_anomalous = 0` no visual + cap do eixo X.
2. **Composição ilegível com barras separadas** — escalas muito diferentes. Fix: **barra empilhada** (4 medidas em Valores, categoria vazia → 1 barra, 4 segmentos).
3. **Título de card sobrando da Página 2** — ao copiar cards, o título não atualiza junto da medida. Sempre conferir título + medida + subtítulo.

### 11.6 Resumo das medidas da Página 3
| Medida | Fórmula-chave | Card/Visual |
|---|---|---|
| Avg Distance (mi) | DIVIDE(Total Distance, Total Trips) | K2 |
| Avg Speed (mph) | AVERAGE(avg_speed_mph) sem anomalias | K3 |
| Avg Cost per Trip ($) | SUM(total_amount)/Trips | K4 |
| Cost per Mile ($) | UDF ComputeRevenuePerMile | K5 |
| Avg Base Fare / Tip / Tolls / Surcharges & Taxes | SUM(x)/Trips | V3 composição |
| duration_class (Gold) | CASE = UDF ClassifyTripDuration | V2 histograma |

## 12. DAX da Página 4 — Demand vs Demographics (dia 8)

### 12.1 Atributos de borough sem dupla contagem
`total_population` e `median_household_income` ficam repetidos em cada zona do borough. Para não somar o mesmo número 265×:
```dax
Population (covered) = SUMX ( VALUES ( dim_zone[borough] ), CALCULATE ( MAX ( dim_zone[total_population] ) ) )
```
`VALUES(borough)` pega cada borough uma vez; `MAX(total_population)` pega o valor do borough; `SUMX` soma os distintos. Income é **média ponderada pela população** (média simples de medianas seria errada):
```dax
Median HH Income =
VAR _t = ADDCOLUMNS ( VALUES ( dim_zone[borough] ),
    "@pop", CALCULATE ( MAX ( dim_zone[total_population] ) ),
    "@inc", CALCULATE ( MAX ( dim_zone[median_household_income] ) ) )
RETURN DIVIDE ( SUMX ( _t, [@pop]*[@inc] ), SUMX ( _t, [@pop] ) )
```

### 12.2 Shares e Demand Index
```dax
Revenue Share    = DIVIDE ( [Total Revenue], CALCULATE ( [Total Revenue], REMOVEFILTERS ( dim_zone ) ) )
Population Share = DIVIDE ( [Population (covered)], CALCULATE ( [Population (covered)], REMOVEFILTERS ( dim_zone ) ) )
Demand Index (pp) = [Revenue Share] - [Population Share]
```
`REMOVEFILTERS(dim_zone)` traz o total da rede como denominador → cada borough vira % do todo. O índice (diferença) >0 = over-served, <0 = sub-servido.

### 12.3 Correlação de Pearson em DAX (toque de data science)
```dax
Demand-Income Corr =
VAR _t = ADDCOLUMNS ( FILTER ( VALUES ( dim_zone[borough] ), CALCULATE ( MAX ( dim_zone[total_population] ) ) > 0 ),
    "@x", CALCULATE ( MAX ( dim_zone[median_household_income] ) ), "@y", [Total Revenue] )
VAR _n=COUNTROWS(_t) VAR _sx=SUMX(_t,[@x]) VAR _sy=SUMX(_t,[@y])
VAR _sxy=SUMX(_t,[@x]*[@y]) VAR _sx2=SUMX(_t,[@x]*[@x]) VAR _sy2=SUMX(_t,[@y]*[@y])
RETURN DIVIDE ( _n*_sxy-_sx*_sy, SQRT((_n*_sx2-_sx*_sx)*(_n*_sy2-_sy*_sy)) )
```
n=5 (boroughs) → direcional, não conclusivo. Resultado ≈ 0,43 (moderado).

### 12.4 Gotchas
1. **Uma medida por vez** — colar o bloco todo num measure → erro "sintaxe de Median" (o parser lê "Median HH Income =" como parte da expressão).
2. **Filtro de página** pra remover Unknown/N/A/EWR dos 4 visuais de uma vez (poluíam tudo + geravam valores estranhos, ex.: EWR sem população residente).
3. **Caveat da per-capita** — denominador = população residente; Manhattan inflada (riders ≠ moradores). Citar na demo = maturidade analítica.

### 12.5 Resumo das medidas da Página 4
| Medida | Papel |
|---|---|
| Population (covered) / Median HH Income | tamanho e riqueza do mercado (sem dupla contagem) |
| Revenue/Trips per Capita | intensidade de demanda normalizada |
| Revenue Share / Population Share | base do índice de penetração |
| Demand Index (pp) | over/under-served (hero divergente) |
| Demand-Income Corr | correlação demanda × renda (DS) |

## 13. DAX da Página 5 — Data Quality & Governance (dia 8)

### 13.1 KPIs do funil
```dax
Raw Rows = 119136044                       -- fato do pipeline (Bronze), hardcoded
Clean Rows = [Total Trips]                 -- 115,7M, vem do modelo
Rows Discarded = [Raw Rows] - [Clean Rows]
Retained % = DIVIDE ( [Clean Rows], [Raw Rows] )   -- 97,16%
Discarded Rows = SUM ( discard_reasons[rows] )     -- valor do V1
```
**Gotcha**: com o calc group ligado, "desencorajar medidas implícitas" fica on → não dá pra jogar `discard_reasons[rows]` cru num visual; tem que ser a medida `Discarded Rows`.

### 13.2 RLS — medidas "Manager View"
```dax
Global Manager View    = [Total Revenue]
Manhattan Manager View = CALCULATE ( [Total Revenue], dim_zone[borough] = "Manhattan" )
-- idem Queens, Brooklyn, Bronx, Staten Island
```
As 6 roles vivem no semantic model do Fabric. O composite (DirectQuery sobre modelo remoto) **não** mostra roles remotas no "Exibir como", então essas medidas demonstram o efeito do RLS no relatório. Cada gestor vê só seu borough; o global vê os $3,06 Bi.

### 13.3 Top anomalies — agregado por zona
Sem trip_id único no fato, não dá pra listar trips individuais nativamente (a tabela agrupa). Solução: max por zona.
```dax
Max Anomaly Total = CALCULATE ( MAX ( fact_trips[total_amount] ), fact_trips[is_anomalous] = 1 )
Max Anomaly Duration = CALCULATE ( MAX ( fact_trips[duration_min] ), fact_trips[is_anomalous] = 1 )
```
Tabela: `zone_name` × `[Anomalous Trips]`, `[Max Anomaly Total]`, `[Max Anomaly Duration]`, Top 10 por max → Gramercy ($401k) no topo.

### 13.4 Tabela auxiliar `discard_reasons`
4 linhas (reason, rows) via Inserir dados (ou Gold). Soma = 3,55M > líquido 3,38M (overlap: linhas que falham mais de um critério, contadas em cada motivo, removidas uma vez).

### 13.5 Gotchas
- Calc group → medidas explícitas obrigatórias.
- DirectLake não publica tabela/coluna nova sozinho → "Editar tabelas" no modelo + sync do SQL endpoint.
- Retenção real = **97,16%** (corrigido o typo 98,16% no DATA_DICTIONARY).

---

## 14. DAX da Página 6 — Weather Impact & Operational Risk (dia 9)

### 14.1 Objetivo e contexto

Usar os dados NOAA já presentes na `dim_date` para duas análises complementares:

1. **Sazonalidade** (`dim_date[season]`): qual estação gera mais demanda? onde o lead time deteriora?
2. **Eventos extremos** (`dim_date[is_extreme_weather]`): eventos climáticos severos comprimem demanda e afetam custo?

Colunas usadas da `dim_date`:
- `season` — Spring / Summer / Fall / Winter (derivada de MONTH no notebook Gold)
- `se