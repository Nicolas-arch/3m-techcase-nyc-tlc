# Material de Apoio — Como funciona o código deste projeto

> Documento didático para entender cada linha que estamos escrevendo. Não é tutorial de Spark do zero — é um "decoder" do código deste projeto específico, peça por peça, em português direto.

## Índice

1. [Conceitos fundamentais](#1-conceitos-fundamentais)
2. [Notebook 01_bronze_ingest (PySpark)](#2-notebook-01_bronze_ingest-pyspark)
3. [Notebook 02_silver_clean (Spark SQL)](#3-notebook-02_silver_clean-spark-sql)
4. [Glossário rápido](#4-glossário-rápido)

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

## 4. Glossário rápido

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
| **F-string** (Python) | `f"texto {variavel}"` — interpolação |
| **List comprehension** | `[x for x in lista]` — lista construída inline |
| **ThreadPoolExecutor** | Executa tarefas em paralelo em threads |

---

## Próximos passos do material

A medida que o projeto avançar (Gold, modelo semântico Power BI, DAX), este documento vai ganhar mais seções:

- **DAX explained** — para Calc Groups, UDFs, Field Parameters
- **M / Power Query explained** — se precisarmos de transformações no nível do PBI
- **Diagramas visuais** — exportados do draw.io

Se quiser entrar fundo em algum tópico específico (ex.: "como funciona shuffling no Spark", "diferença entre wide e narrow transformations"), me avisa que crio uma seção dedicada.
