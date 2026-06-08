# Notas para a Apresentação (decisões técnicas + respostas-padrão)

> Documento vivo. Atualizado a cada decisão técnica relevante tomada durante o desenvolvimento. Vira a base do roteiro do PPTX final.

## Como usar este documento

Cada seção tem três blocos:

- **Decisão / problema** — o que aconteceu.
- **Resposta curta** — frase de 1-2 linhas para slide.
- **Resposta longa (entrevista)** — argumento completo para a reunião de 50 min.

---

## 1. Escolha da stack: Microsoft Fabric Lakehouse

### Decisão
Usar Microsoft Fabric (Lakehouse + Notebook PySpark/Spark SQL) em vez de DuckDB, Snowflake, ou Power Query puro.

### Resposta curta (slide)
"Fabric é a stack que o time de Enterprise Supply Chain Analytics da 3M usa. Demonstrar fluência nessa stack > entregar em ferramenta isolada."

### Resposta longa (entrevista)
"Considerei três alternativas: (1) DuckDB local — tecnicamente excelente, mas pouco conhecido em ambiente corporativo; (2) Snowflake — sólido, mas a 3M já está investindo em Fabric e mostrar a mesma stack é mais relevante; (3) Power Query puro — não escala para 110M linhas. Escolhi Fabric porque (a) é o stack que vocês usam, (b) Lakehouse + Delta dão arquitetura medallion nativa, (c) DirectLake permite zero-copy ao Power BI, (d) Trial de 60 dias cobre o teste sem custo. Trade-off: Fabric Trial tem limite de capacity units; mitiguei rodando ingestão paralela com poucos workers e materializando Delta uma única vez."

---

## 2. PySpark vs Python puro (Pandas)

### Decisão
Usar PySpark na ingestão (Bronze) e Spark SQL nas camadas Silver e Gold. Pandas/Polars só no plano B local.

### Resposta curta (slide)
"PySpark na ingestão porque precisava de libs Python (requests). Spark SQL no Silver/Gold para clareza e auditoria."

### Resposta longa (entrevista)
"A pergunta natural é 'por que Spark se Pandas seria mais leve?'. Resposta: 110 milhões de linhas não cabem confortavelmente em Pandas single-node — estouraria RAM. Polars resolveria localmente mas eu perderia integração nativa com Delta Lake do Fabric, DirectLake no Power BI e escalabilidade horizontal. Spark me dá um pipeline que escala de MB a TB sem reescrever código. Sobre consumo: sim, Spark usa mais CUs que script Python puro, mas o trade-off é deliberado — o ganho em manutenibilidade, integração nativa e padronização (mesma linguagem usada por DEs sêniores) compensa. Em produção 3M, esse pipeline já estaria pronto para escalar."

---

## 3. Arquitetura medallion (Bronze / Silver / Gold)

### Decisão
Pipeline em 3 camadas Delta no Lakehouse: Bronze (raw), Silver (clean), Gold (modeled).

### Resposta curta (slide)
"Separação clara de responsabilidades. Reprocessamento isolado de qualquer camada. Padrão de mercado."

### Resposta longa (entrevista)
"Optei por medallion por três motivos: (1) separation of concerns — bronze é fonte da verdade, silver é qualidade, gold é modelo; (2) idempotência por camada — posso reprocessar Silver sem tocar Bronze; (3) é o padrão de fato em Databricks/Fabric, então qualquer DE entrando no projeto entende a estrutura imediatamente. Trade-off: 3x mais armazenamento que uma tabela única — mitigado pelo fato de que Delta usa compressão eficiente e OneLake não cobra por storage no Trial."

---

## 4. Schema drift na ingestão TLC

### Problema encontrado (dia 29/05)
Tentei ler todos os 36 parquets com `spark.read.parquet(...).option("mergeSchema", "true")` e o Spark falhou com erro `[CANNOT_MERGE_INCOMPATIBLE_DATA_TYPE]`. Ao investigar, descobri que a TLC tem inconsistências de schema entre os meses:

- `VendorID` aparece como `INT` em alguns meses e `BIGINT` em outros.
- `passenger_count`, `RatecodeID` mudam entre `DOUBLE` e `BIGINT`.
- `PULocationID/DOLocationID` mudam entre `INT` e `BIGINT`.
- **Mais sutil**: `airport_fee` em 2022-2023 vira `Airport_fee` (com A maiúsculo) em alguns arquivos de 2024.

### Solução adotada
Schema canônico explícito + cast file-by-file.

```python
TARGET_COLS = [
    ("VendorID", "bigint"),
    ("passenger_count", "double"),
    # ... lista completa em 01_bronze_ingest, Cell 4
]

def read_normalized(year, month):
    df = spark.read.parquet(path_for(year, month))
    col_lookup = {c.lower(): c for c in df.columns}   # case-insensitive
    exprs = []
    for tgt_name, tgt_type in TARGET_COLS:
        src_name = col_lookup.get(tgt_name.lower())
        if src_name is None:
            exprs.append(F.lit(None).cast(tgt_type).alias(tgt_name))
        else:
            exprs.append(F.col(src_name).cast(tgt_type).alias(tgt_name))
    return df.select(*exprs)
```

### Resposta curta (slide)
"Schema drift entre arquivos da TLC resolvido com schema canônico explícito + cast file-by-file."

### Resposta longa (entrevista)
"Logo na ingestão encontrei schema drift entre arquivos da TLC: VendorID aparece como INT em alguns meses e BIGINT em outros, e airport_fee muda capitalização em arquivos de 2024. O `mergeSchema=true` do Spark falha porque INT e BIGINT são tratados como incompatíveis e case-difference cria colunas duplicadas. Resolvi com schema-on-write controlado: defini os 19 campos com tipos canônicos, leio cada arquivo individualmente, faço lookup case-insensitive das colunas e converto via `cast()`. Vantagens: (a) trata case difference, (b) lida com colunas faltantes emitindo NULL no tipo certo, (c) deixa explícito qual tipo final esperamos — auditável e seguro em produção. É a mesma estratégia que aplico quando consumo extratos do SAP, onde campos numéricos vêm com tipo variável dependendo da configuração de batch."

### Por que isso é positivo na avaliação
Encontrar e resolver schema drift demonstra: (1) conhecimento de Spark, (2) atenção a dados reais (não brincando com dataset limpo de tutorial), (3) mentalidade de produção, (4) capacidade de transformar erro em decisão arquitetural.

---

## 4.1 Descobertas de qualidade de dados — TLC tem "vazamentos" temporais

### Problema observado (dia 29/05)
Após materializar Bronze (119.136.044 linhas), rodei `GROUP BY YEAR(pickup_datetime)` e descobri:

- **663 registros com data fora do escopo** (~0,00056% do total). Datas encontradas: 2001, 2002, 2003, 2008, 2009, 2012, 2014, 2021, **2025** e até **2026**.
- 2002 foi o ano "vazado" mais frequente (463 registros, provavelmente erro de meter num lote específico).
- **Datas futuras presentes**: `max(pickup_datetime) = 2026-06-26 23:53:12` — claramente erro de sistema de meter na hora do dropoff. 5 registros datados 2025 e alguns 2026.
- Volume normal por mês 2022-2024: entre 2,4M e 3,7M de viagens (consistente).

### Padrão de demanda anual descoberto (vai pro slide executivo)
| Ano | Trips | YoY |
|---|---|---|
| 2022 | 39.655.573 | — |
| 2023 | 38.310.138 | **-3,4%** |
| 2024 | 41.169.670 | **+7,5%** |

**Storytelling de página 1 do dashboard**: "Pós-pandemia, 2023 ainda mostrou queda de 3% antes da recuperação forte de 2024 (+7,5%). Em supply chain, mesmo padrão visto em demanda B2B em 2023 — desaquecimento global seguido de retomada."

### Resposta curta (slide)
"A TLC publica dados com 'vazamentos temporais' — registros datados antes de 2022 e até no futuro (2026), por erro no sistema de meter. Filtrados na camada Silver."

### Resposta longa (entrevista)
"Ao validar a camada Bronze descobri que a TLC tem dois tipos de ruído temporal: (1) registros com datas anteriores ao escopo do arquivo — encontrei trips com datas de 2001 a 2021 dentro dos arquivos mensais de 2022-2024; (2) datas no futuro — o `max(pickup_datetime)` apareceu como 2026-06-26, o que é impossível, claramente erro no sistema TPEP. Volume é pequeno (~656 registros pré-2022 em 119M), mas ignorar essa qualidade seria erro. Na Silver eu filtro estritamente `pickup_datetime BETWEEN '2022-01-01' AND '2025-01-01'`, descarto, e documento a métrica no relatório de Data Quality. Em supply chain, isso é equivalente a 'extrato com lançamento retroativo ou erro de data no SAP' — todo pipeline robusto precisa lidar com isso."

### Por que isso é positivo na avaliação
Mostra: (1) sempre validar dado antes de modelar, (2) construir Bronze como espelho da verdade (incluindo o ruído) e Silver como camada limpa — exatamente o que medallion prega, (3) capacidade de transformar quirky data em insight contado para o público.

---

## 4.2 Descobertas adicionais — Silver Layer (dia 30/05)

### Resumo numérico
- Bronze: **119.136.044 trips**
- Silver: **115.756.175 trips**
- Descartado: **3.379.869 (2,84%)**
- Anomalias flagged (mantidas com flag): **180.145 (0,16%)**

### Breakdown de descartes
| Motivo | Linhas | % bronze |
|---|---|---|
| Distância ≤ 0 | 2.123.821 | 1,78% |
| Tarifa negativa | 1.365.574 | 1,15% |
| Duração inválida (dropoff ≤ pickup) | 61.114 | 0,05% |
| Vazamento temporal (2001-2026) | 663 | 0,0006% |

**Insight de apresentação**: o "lixo" da TLC não é o que parecia. Os 663 vazamentos temporais (descoberta de ontem) são insignificantes perto dos 2,1M trips com distância zero (cancelamentos / erros de meter) e 1,4M com tarifa negativa (estornos / correções). Isso reforça uma boa prática de data engineering — **sempre quantifique antes de assumir**.

### Top anomalias detectadas
Trips flagged como `is_anomalous = 1` (180k registros). Top 4 por total_amount:

| Pickup | Duration | Distance | Total |
|---|---|---|---|
| 2022-01-07 | 10 min | 3.3 mi | **$401.092,32** |
| 2022-06-11 | 8 min | 1.2 mi | **$395.844,94** |
| 2023-06-12 | 11 min | 1.5 mi | **$386.983,63** |
| 2024-11-01 | 15 min | 2 mi | **$335.544,44** |

Claramente erros de sistema do meter TPEP (provavelmente leitura de 6 dígitos foi feita errada — um zero a mais). Em supply chain, esse tipo de outlier seria caso direto de auditoria/fraude.

### Resposta de entrevista — qualidade de dados
"Na camada Silver descartei 2,84% do volume Bronze (3,4M trips). Quebra interessante: a maior parte não é vazamento temporal como eu suspeitava no dia 1, e sim trips com distância zero (1,78%) — provavelmente cancelamentos onde o meter abriu mas não rodou — e tarifas negativas (1,15%) — estornos e correções. Mantive 180k anomalias com flag (não descartei) para análise no dashboard de Data Quality. Top anomalia: trip de US$401k em 10 minutos — erro óbvio de meter. Em supply chain, esse padrão de descobrir patterns de qualidade é o que diferencia 'limpar dado' de 'entender o sistema operacional por trás do dado'."

---

## 4.4 Insights validados pelo Business Sanity Query (dia 31/05)

Query analítica representativa rodada contra o Gold antes do Power BI. Confirma coerência do modelo e revela patterns de negócio.

### Distribuição Borough x Year (2024)

| Borough | Trips | % total | Avg fare USD |
|---|---|---|---|
| Manhattan | 35.270.447 | 95,4% | 23,94 |
| Queens | 3.650.417 | 9,9% | 73,21 |
| Brooklyn | 554.806 | 1,5% | 31,82 |
| Bronx | 114.001 | 0,3% | 35,88 |
| EWR (Newark) | 1.247 | <0,01% | 93,70 |
| Staten Island | 1.476 | <0,01% | 44,66 |

### Achados de negócio importantes

**1. Yellow Taxi é fenômeno de Manhattan** — 95%+ do volume. Em supply chain, equivalente à curva ABC clássica onde poucos pares O-D respondem pela maioria das movimentações.

**2. Avg fare por Borough refleta padrão operacional**:
- Manhattan: trips curtas urbanas ($18-24)
- Queens: contém JFK + LaGuardia → surcharge de aeroporto ($54-73)
- EWR: Newark Airport → distância longa + airport fee ($93-105)
- Outer boroughs: trips de médio porte ($30-66)

**3. Outer boroughs explodiram em demanda em 2024** (descoberta valiosa):
- Brooklyn: 245.894 (2023) → **554.806 (2024) = +125% YoY**
- Bronx: 51.863 (2023) → **114.001 (2024) = +120% YoY**
- Enquanto Manhattan teve recuperação modesta (+6,9%), outer boroughs dobraram.

**Storytelling para slide**: "Manhattan ainda domina, mas o crescimento de demanda em 2024 está nos outer boroughs. Padrão típico de supply chain quando hubs principais saturam e demanda migra para nodes secundários — em transporte e em distribuição B2B."

### Achado de qualidade — médias infladas em boroughs de baixo volume

Avg trip_distance em alguns boroughs aparece absurdo:
- Staten Island 2024: avg_distance **94,69 milhas** (impossível dentro de NYC)
- Bronx 2022: avg_distance **40,69 milhas**
- Brooklyn 2022: avg_distance **48,69 milhas**

**Causa**: em boroughs com baixo volume de trips (1k-260k), os 0,16% de trips anômalas (`is_anomalous = 1`) movem fortemente a média. Em Manhattan (35M trips), o mesmo % de anomalias é diluído.

**Decisão para o dashboard**:
- Visões executivas: **filtrar `is_anomalous = 0`** ou usar **mediana** em vez de média.
- Página de Data Quality: mostrar anomalias explicitamente para evidenciar o cuidado.

**Resposta de entrevista**: "Ao rodar uma query de sanidade descobri que averages nos outer boroughs eram absurdamente altos — Staten Island com avg_distance de 94 milhas. Investiguei e era inflation pela presença das 180k anomalias (`is_anomalous=1`) em pequenos volumes onde elas não diluem. Decisão para o dashboard: filtrar anomalias nas visões executivas e mostrar mediana. Isso ilustra exatamente o por quê de manter a flag `is_anomalous` em vez de descartar — o dado bruto tem valor para análise de qualidade, mas precisa ser filtrado para análise operacional."

---

## 4.3 Gold Layer — Star Schema (dia 31/05)

### Modelo final entregue
1 fato + 6 dimensões em Delta, particionado por year/month no fato.

| Tabela | Linhas | Observação |
|---|---|---|
| `dim_date` | 1.096 | Calendário 2022-2024 + clima diário (NOAA) |
| `dim_time` | 24 | Horas + day_part + is_rush_hour |
| `dim_zone` | 265 | Taxi zones + ACS demographics por Borough |
| `dim_vendor` | 5 | Provedores TPEP (4 + Unknown) |
| `dim_ratecode` | 7 | Códigos de tarifa (6 + Unknown) |
| `dim_payment` | 7 | Tipos de pagamento (6 + Unknown) |
| `fact_trips` | 115.756.175 | Surrogate keys + measures + flags |

**Integridade referencial validada**: zero foreign keys orfãs em todas as 7 colunas FK do fato (graças ao padrão COALESCE → Unknown).

### Padrão "API + Snapshot" para fontes externas

Decisão arquitetural relevante: trate dados externos com base em sua volatilidade.

| Fonte | Método | Por quê |
|---|---|---|
| NOAA Weather (diário) | REST API com paginação anual | Dados operacionais, refresh diário em prod |
| ACS Demographics (anual) | Snapshot hardcoded (Borough-level) | Master data estável, refresh anual em prod |

**Resposta de entrevista**: "Para enrichment usei dois padrões diferentes. NOAA é dado operacional (clima muda todo dia, refresh diário) — então consumi via REST API com paginação anual, respeitando o rate limit de 1 ano por request do CDO API. ACS é master data (renda e população mudam a cada censo, refresh anual) — então fiz snapshot dos valores Borough-level diretamente, com versionamento no repo. Isso é exatamente o padrão que vejo em pipelines reais: dado operacional via API, dado de referência via snapshot versionado. Confunde misturar os dois — um precisa de refresh frequente, o outro precisa de auditabilidade."

### Gotcha encontrado: NOAA CDO API limite de 1 ano

Primeira chamada com range 2022-01-01 → 2024-12-31 retornou HTTP 400. Documentação da NOAA tem essa restrição: **endpoint de Daily Summaries (GHCND) aceita range máximo de 1 ano por request**. Fix: loop por ano, 3 chamadas separadas (cada uma paginada).

**Resposta de entrevista**: "Encontrei um limite não-óbvio da NOAA CDO API: requests com range > 1 ano para dados diários retornam HTTP 400. A doc menciona isso mas não no lugar mais visível. Fix foi loopar por ano e paginar dentro de cada ano. É o tipo de problema que só descobre rodando — em prod, esse tipo de descoberta vai pra um runbook do pipeline."

### Decisão: enrichment ACS no nível Borough (não Zone)

ACS não publica dados granulados por Taxi Zone (que é divisão fiscal da TLC, não geográfica do Census). Tinha duas opções:
1. **Crosswalk espacial NTA ↔ Zone** via Geopandas + shapefile (mais preciso, mais complexo, mais tempo)
2. **Borough-level enrichment** (menos granular, instantâneo, totalmente reliable)

**Escolha**: opção 2, com justificativa explícita. Em produção, faria opção 1, mas para o teste a opção 2 entrega ~95% da insight com 5% do esforço.

**Resposta de entrevista**: "Optei por enrichment no nível Borough porque ACS não publica por Taxi Zone e o crosswalk espacial NTA-Zone consumiria tempo desproporcional ao ganho analítico. Documentei isso como decisão consciente — em produção 3M, com mais tempo, aplicaria spatial join via Geopandas. Para o teste, prioritizei entrega completa do star schema vs precisão geográfica."

### COALESCE → Unknown como padrão de referential integrity

Em todas as FKs do fact_trips, usei `COALESCE(dim.surrogate_key, <unknown_key>)`. Garante zero orfãs no fato mesmo se silver tem registros com vendor_id desconhecido (ex.: vendor 99 não previsto).

```sql
COALESCE(dv.vendor_key,   5)  AS vendor_key      -- 5 = Unknown row in dim_vendor
COALESCE(dr.ratecode_key, 7)  AS ratecode_key    -- 7 = Null/Unknown
COALESCE(dp.payment_key,  6)  AS payment_key     -- 6 = Unknown
```

Isso é **boa prática Kimball** — toda dimensão tem uma linha "Unknown" para acomodar dados sujos sem quebrar joins.

---

## 4.5 Insights descobertos via Conditional Formatting (dia 03/06)

Ao validar conditional formatting com visuais reais (data bars + ícones de semáforo), descobri 2 patterns que vão estruturar a narrativa executiva da apresentação.

### Insight 1 — Aeroportos respondem por 46% da receita do top 10 zonas

Ao rankear `dim_zone[zone_name]` por `[Total Revenue]` com Data Bars, o top 10 do projeto inteiro mostra:

| Rank | Zone | Total Revenue |
|---|---|---|
| 1 | JFK Airport | $425M |
| 2 | LaGuardia Airport | $222M |
| 3 | Midtown Center | $120M |
| 4 | Upper East Side South | $104M |
| 5 | Times Sq/Theatre District | $100M |
| 6 | Upper East Side North | $96M |
| 7 | Penn Station/Madison Sq West | $90M |
| 8 | Midtown East | $89M |
| 9 | Lincoln Square East | $78M |
| 10 | Midtown North | $77M |
| **Total top 10** | | **$1.404M** |

**JFK + LaGuardia = $647M = 46% do top 10 zonas.**

**Storytelling para slide**: "Yellow Taxi tem 265 zonas, mas só 2 (JFK + LaGuardia) respondem por 46% da receita do top 10. Em supply chain, equivale a ter 2 lanes que carregam quase metade do EBITDA — risco de concentração que pede contingência. Decisões estratégicas: onde investir em capacidade adicional, onde redirecionar fluxo em caso de disrupção, qual diversificação reduz dependência."

**Por que isso importa pro 3M**: padrão concentração ABC clássico, com os principais hubs do supply chain transportando o grosso do valor. Discussão típica de risk management.

### Insight 2 — Anomaly Rate revela "core operations vs exception lanes"

Ao aplicar ícones de semáforo na coluna `[Anomaly Rate %]` por borough, a distribuição revelou padrão muito claro:

| Borough | Anomaly Rate | Volume contexto |
|---|---|---|
| Manhattan | **0,1%** 🟢 | 95% do volume — core operations |
| Bronx | 0,3% 🟢 | Volume médio |
| Queens | 0,3% 🟢 | Médio (puxado pelos aeroportos) |
| Brooklyn | 0,3% 🟢 | Médio |
| Unknown | 0,6% 🟢 | Baixo volume |
| Staten Island | 1,0% 🟡 | Baixíssimo volume |
| EWR (Newark) | **16,7%** 🔴 | Trips longas, complexas |
| N/A | **23,2%** 🔴 | Outside-NYC, dados ruins |

**Storytelling para slide**: "Manhattan core opera em 99,9% de qualidade. Em zonas de fronteira (EWR, N/A), anomaly rate explode pra 17-23%. É o padrão clássico de 'core operations vs exception lanes' — operação consolidada vs operação de exceção. Em supply chain 3M, equivale a 'main shipping lanes vs special-case routes' — processo deve ser diferente, contratos diferentes, SLA diferente. Mostra que one-size-fits-all não funciona em operação de grande escala."

**Resposta de entrevista pronta**: "Ao construir o dashboard, percebi via conditional formatting que anomaly rate não é uniforme — Manhattan core opera a 0,1%, mas EWR e zonas fronteira chegam a 23%. Isso me levou a propor uma narrativa de 'core vs exception operations' que vou explorar nas próximas páginas. Mostra que dashboard bem feito gera **questões novas**, não só responde às existentes."

---

## 5. DirectLake vs Import vs DirectQuery

### Decisão
Modelo semântico Power BI conectado em modo DirectLake ao Lakehouse.

### Resposta curta (slide)
"DirectLake: lê Delta direto do OneLake sem copiar. Performance de Import sem duplicação."

### Resposta longa (entrevista)
"Tinha três opções no Power BI: Import duplica os dados (Lakehouse + modelo em memória), DirectQuery sofre com latência em fato de 110M linhas, e DirectLake — modo novo do Fabric que lê Delta nativamente do OneLake sem importar. Escolhi DirectLake por três motivos: (1) zero-copy — não tenho dois lugares onde os mesmos dados moram; (2) performance comparável a Import porque o Vertipaq engine consome Delta direto; (3) governança — quando o Lakehouse atualiza, o modelo enxerga imediatamente, sem refresh manual. Trade-off: ainda é tecnologia recente (preview features), então testei antecipadamente para garantir que Calc Groups, UDFs e Field Parameters funcionam em conjunto."

---

## 6. Star schema com role-playing dimension para Zone

### Decisão
1 fato + 6 dimensões. `dim_zone` participa duas vezes do modelo: uma como `dim_zone_pickup` (relacionamento ativo) e outra como `dim_zone_dropoff` (segunda cópia visível).

### Resposta curta (slide)
"Star schema clássico. Zone usada duas vezes via role-playing para legibilidade."

### Resposta longa (entrevista)
"Star schema é o padrão para modelos semânticos em ferramentas de BI — Vertipaq otimiza queries assumindo essa estrutura e features como Field Parameters e Calc Groups funcionam melhor com dimensões denormalizadas. Para Zone, considerei duas opções: USERELATIONSHIP ativando a relação dropoff sob demanda, ou role-playing dimension (duas cópias da tabela). Escolhi role-playing por três motivos: (1) legibilidade — quem abre o modelo vê duas tabelas com nomes claros (Pickup Zone, Dropoff Zone), (2) os usuários finais não precisam aprender USERELATIONSHIP nas medidas, (3) custo marginal de memória aceitável (265 zonas duplicadas). Trade-off: o modelo fica visualmente mais pesado, mas a clareza compensa."

---

## 7. Fonte externa de enriquecimento (critério do briefing)

### Decisão
NYC Open Data ACS Demographics como fonte primária. NOAA Weather como secundária opcional.

### Resposta curta (slide)
"Demografia ACS por bairro: correlaciona demanda de táxi com renda e densidade populacional — equivalente a demand drivers em supply chain."

### Resposta longa (entrevista)
"O critério do teste é trazer uma fonte externa que correlacione. Escolhi NYC ACS Demographics (American Community Survey) porque permite responder uma pergunta de negócio direta: a demanda por táxi é função de densidade populacional e renda? Esse é o exato paralelo de uma análise de demand drivers em supply chain — onde queremos entender que fatores estruturais explicam volume de pedidos por região. Trouxe também NOAA Weather como fonte secundária para análise de risco operacional ('shocks externos' como chuva e neve afetam lead time e demanda — mesmo padrão que vemos em supply chain quando há paralisações, eventos climáticos, picos sazonais). Crosswalk NTA ↔ Taxi Zone feito com mapping público + ajustes manuais nas zonas críticas (aeroportos)."

---

## 8. Features Power BI — implementação real (dia 1 + dia 2)

### 8.1 Composite Model (DirectQuery + remote semantic model)

**Decisão arquitetural** (não estava no plano original mas surgiu na execução):

Power BI Desktop não permite editar o modelo quando conectado em **Live connection** puro ao semantic model. Pra criar relacionamentos, hierarquias, medidas e RLS locais, foi necessário converter pra **composite model** clicando em "Fazer alterações neste modelo".

**Resposta de entrevista**: "Mantive o semantic model `sm_yellow_taxi_3m` como source-of-truth no Fabric (camada de dados, relacionamentos do star schema). No Power BI Desktop, converti pra composite model em modo DirectQuery — isso permite adicionar medidas, hierarquias e RLS locais sem importar os 115M de linhas pro `.pbix`. Trade-off conhecido: leve perda de performance vs DirectLake puro, mas ganho enorme de flexibilidade pra customizações no nível do relatório. Em produção 3M, esse padrão funciona bem porque o dado bruto fica no Lakehouse e cada equipe pode customizar a camada de relatório sem mexer no modelo central."

### 8.2 Time Intelligence + Calculation Group "Time Analytics"

**Como foi implementado**: via **Tabular Editor 2** (External Tools do Power BI Desktop). Criação visual no TE2, save direto no modelo local, refresh no PBI Desktop.

**Os 7 items criados** (com ordinal zero-based):
- `Current` (0) → `SELECTEDMEASURE()`
- `PY` (1) → `CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR(dim_date[full_date]))`
- `YoY` (2) → `SELECTEDMEASURE() - CALCULATE(...)`
- `YoY %` (3) → `DIVIDE(_Current - _PY, _PY)` com format string `"0.00%;-0.00%;0.00%"`
- `YTD` (4) → `TOTALYTD(SELECTEDMEASURE(), dim_date[full_date])`
- `PYTD` (5) → `CALCULATE(TOTALYTD(...), SAMEPERIODLASTYEAR(...))`
- `MAT` (6) → `CALCULATE(SELECTEDMEASURE(), DATESINPERIOD(dim_date[full_date], MAX(dim_date[full_date]), -12, MONTH))`

**Validação executada** — matriz `year × Time Analytics × Total Revenue`:

| Ano | Current | PY | YoY % |
|---|---|---|---|
| 2022 | $844M | (sem PY) | — |
| 2023 | $1.076M | $844M | **+27,4%** (explosão pós-pandemia) |
| 2024 | $1.137M | $1.076M | **+5,7%** (estabilização) |

**Resposta de entrevista**: "Implementei o Calc Group 'Time Analytics' com 7 calculation items via Tabular Editor 2. A grande vantagem: em vez de criar 7 versões de cada medida (Revenue YoY, Revenue YTD, Revenue MAT...), tenho 1 grupo que aplica time intelligence dinamicamente via `SELECTEDMEASURE()`. Para as 10 medidas base do modelo, isso significa que tenho a opção de 70 perspectivas analíticas com apenas 10 medidas e 7 calc items. Validei o padrão YoY: +27,4% em 2023 (recovery pós-pandemia) e +5,7% em 2024 (estabilização) — narrativa que vai pra slide executivo."

### 8.3 Field Parameters

**Como foi implementado**: PBI Desktop → Modelagem → Novo parâmetro → Campos.

**Os 2 Field Parameters criados**:
- `FP_Metric`: alterna entre 5 medidas (Total Trips, Total Revenue, Total Distance, Avg Duration, Total Tip).
- `FP_Dimension`: alterna entre 5 colunas categóricas (Borough, Zone, Vendor, Payment Type, Day Part).

**Validação executada** — gráfico com `FP_Metric` no eixo Y e `FP_Dimension` no eixo X + 2 slicers:

Resultado da combinação `Total Trips × day_part`:
- **Afternoon**: ~40M trips
- **Evening**: ~40M trips
- **Morning**: ~25M trips
- **Late Night**: ~10M trips

**Insight de negócio** (vai pra slide): "**Afternoon + Evening = 70% do volume total**". Padrão clássico de demanda urbana — ciclo trabalho/lazer dirige a demanda de táxi.

**Resposta de entrevista**: "Criei 2 Field Parameters: um pra métricas e um pra dimensões. Combinados, dão ao analista 25 análises possíveis (5 métricas × 5 dimensões) com 1 único gráfico no canvas. Quando descobri que Afternoon+Evening soma 70% do volume, percebi que o pico noturno é tão importante quanto o vespertino — info que sumiria num gráfico estático focado em só uma métrica. Em supply chain, equivale a entender quando a demanda B2B realmente pesa — não basta total mensal, precisa quebrar por janela operacional."

### 8.4 Conditional Formatting (planejado pro dia 3-4)

Aplicação prevista (não implementado ainda):
- Heat map em matriz `hora × borough` com background color (gradient verde→vermelho)
- KPI Cards com cor dinâmica (verde se YoY > 0, vermelho se < 0)
- Data bars em rankings de zonas por receita
- Ícones (semáforo) em tabela de qualidade de dados

### 8.5 Row-Level Security (RLS) — planejado pro dia 04/06

**Plano**: 5 roles, uma por Borough (Manhattan Manager, Brooklyn Manager, Queens Manager, Bronx Manager, Staten Island Manager). Filtro `[borough] = "<Borough>"` em `dim_zone`.

**Resposta de entrevista** (texto pronto): "Configurei 5 roles RLS — uma por borough — simulando o cenário 'regional manager só vê sua região'. Em supply chain 3M, equivale a 'site manager vê só sua planta', ou 'sales manager vê só sua região'. Demonstrei via View as no Power BI Desktop. Em produção, conectaria as roles via Azure AD groups."

### 8.6 UDFs (DAX User-Defined Functions) — implementadas

**Como foi implementado**: via **DAX Query View** do Power BI Desktop (não via Modelagem ribbon, que não tem UI específica pra UDF). Sintaxe `DEFINE FUNCTION nome = (params) => expression`. Save no modelo via "Atualizar o modelo com alterações".

**As 3 UDFs criadas**:

```dax
FUNCTION ClassifyTripDuration = ( minutes ) =>
SWITCH ( TRUE (),
    minutes < 5,   "Very Short",
    minutes < 15,  "Short",
    minutes < 30,  "Medium",
    minutes < 60,  "Long",
    minutes < 180, "Very Long",
                   "Outlier" )

FUNCTION ClassifyFareTier = ( fare ) =>
SWITCH ( TRUE (),
    fare < 10,   "Economy",
    fare < 25,   "Standard",
    fare < 60,   "Premium",
    fare < 150,  "Luxury",
                 "Anomalous" )

FUNCTION ComputeRevenuePerMile = ( revenue, distance ) =>
DIVIDE ( revenue, distance, 0 )
```

**Validação executada** — tabela `borough × tiers`:

| Borough | Avg Duration | Tier | Avg Fare | Tier | Rev/Mile |
|---|---|---|---|---|---|
| **EWR (Newark)** | 9 min | Short | $101 | **Luxury** | **$24,05** |
| **Manhattan** | 15 min | Medium | $22 | Standard | $5,67 |
| **Queens** | 38 min | Long | $67 | Luxury | $4,87 |
| **Bronx** | 34 min | Long | $34 | Premium | $1,26 |

**Insight de negócio**: EWR (Newark Airport) tem **Revenue per Mile de $24,05** — quase **5x a média de Manhattan**. Razão: trips curtas (9min) com tarifa altíssima (airport fee + distância de NJ). Vira destaque na apresentação: "Newark Airport é um nicho premium dentro da operação Yellow Taxi — alta margem por milha, baixo volume, alto valor unitário."

**Gotchas documentados** (descobertos durante implementação):

1. **Sintaxe sem aspas no DEFINE**: `FUNCTION ClassifyTripDuration = ...` funciona. Com aspas (`FUNCTION 'ClassifyTripDuration' = ...`) retorna erro de sintaxe.

2. **Sem anotação de tipo necessária**: `( minutes : NUMERIC )` não compila. `( minutes )` sem tipo funciona — DAX infere.

3. **Preview feature exige restart do PBI Desktop**: marcar a opção em "Recursos de visualização" e clicar OK não basta. Tem que fechar e reabrir o Power BI Desktop pra o engine ativar.

4. **BLANK → 0 coercion em comparações numéricas**: `BLANK < 5` é `TRUE` em DAX, porque BLANK é coerced para 0 em comparações numéricas. Isso fez aparecer linha "Very Short / Economy" no teste pra valores blank. Fix opcional: `IF(ISBLANK(x), BLANK(), SWITCH(...))`.

**Resposta de entrevista**: "Criei 3 UDFs DAX usando a sintaxe nova do Power BI 2025+. Encapsulei classificações reutilizáveis (Duration Tier, Fare Tier, Revenue per Mile). O ganho: troco a lógica em 1 lugar e propaga pra todas as medidas que usam. Encontrei 3 quirks durante o desenvolvimento — sintaxe sem aspas no DEFINE, ausência obrigatória de type hints, e necessidade de restart pra ativar preview features. Documentei todos no repo pra próxima pessoa não passar pela mesma curva. Também identifiquei o comportamento BLANK→0 do SWITCH — em produção, refinaria com `IF(ISBLANK(...))` explícito, mas pra portfólio mantive intencional pra mostrar o achado."

---

## 9. Plano B (Python + Polars local) — por que existe

### Decisão
Manter scripts Python locais (`python_fallback/`) em paralelo aos notebooks Fabric.

### Resposta curta (slide)
"Continuidade de negócio: se Fabric Trial tiver problema, pipeline equivalente roda em laptop em < 1h."

### Resposta longa (entrevista)
"Como o trial do Fabric pode ter limitação de capacity ou indisponibilidade, mantive em paralelo um pipeline equivalente em Python + Polars. Mesma lógica, mesmo schema canônico, saída em parquet. Mostra preocupação com continuidade de negócio — algo que em supply chain é mandatório. Em produção, esse fallback poderia ser um 'recovery path' caso o cluster Fabric estivesse em manutenção."

---

## 10. Versionamento Git desde o dia 1

### Decisão
Repo GitHub público criado e populado desde antes da primeira linha de código de pipeline.

### Resposta curta (slide)
"Versionamento desde dia 1. Cada decisão técnica tem um commit."

### Resposta longa (entrevista)
"Tratei o teste como projeto de produção. Repo público no GitHub desde o dia 1, conventional commits, branches limpas, README com badges, ARCHITECTURE.md, CONVENTIONS.md, DATA_SOURCES.md. Em um time como o de vocês — global, distribuído — versionamento desde o primeiro commit é o que separa código de scriptzinho. Quando o pipeline crescer ou outra pessoa precisar manter, todo o contexto está lá."

---

## 11. Página 1 — refinamentos finais e narrativa fechada (dia 4)

### Decisão
Página 1 (Executive Overview) finalizada no dia 4 após 3 polishes obrigatórios + 1 substituição estrutural de KPI. Tela passou de "funcional" para "client-ready" via micro-decisões de UX, cor semântica dinâmica e internacionalização para inglês (apresentação global).

### Os 4 ajustes aplicados no dia 4

**Polish 1 — Anomaly card com cor dinâmica + subtítulo dinâmico**
- Antes: valor "0,16%" preto, subtítulo estático "180k flagged • Saudável".
- Depois: valor verde (via `Anomaly Color`), subtítulo via `Anomaly Subtitle` que reage ao threshold (Healthy <3%, Warning <10%, Critical ≥10%).
- Razão: consistência semântica com o card de Revenue (que já era colorido). 2 cards-sinalizadores + 3 cards informativos.

**Polish 2 — Filtro N/A na tabela Top 10 O-D**
- Antes: linha "N/A → N/A" aparecia entre as top 10 (ruído de dados).
- Depois: filtro visual `borough <> "N/A"` AND `dropoff_borough <> "N/A"`.
- Razão: dashboard executivo não pode mostrar lixo de pipeline. Pareceria que a engenharia não foi cuidadosa.

**Polish 3 — Altura da tabela ampliada**
- Antes: 175px de altura, só 5 das 10 linhas visíveis (violação do "Top 10" prometido).
- Depois: 270px, todas as 10 linhas sem scroll.
- Razão: visual coerente com o título.

**Extra A — KPI "YoY Growth" substituído por "AVG LEAD TIME (min)"**
- Antes: card YoY Growth = 5,7% (redundante com o subtítulo do card de Revenue, que já dizia "+5,7% vs 2023").
- Depois: card Avg Lead Time = 17,56 min (média ponderada dos 3 anos).
- Razão: **Lead Time é o KPI hero universal de supply chain**. Conecta diretamente com a narrativa de Yellow Taxi como cadeia urbana — trip = shipment, duration = lead time. Para a vaga da 3M (Enterprise Supply Chain Analytics), esse KPI substitui um indicador redundante por um indicador estruturalmente alinhado com o domínio.

### Internacionalização (PT → EN)
Todos os subtítulos e textos dinâmicos passaram para inglês:
- "Volume 2022–2024", "Average ticket", "Average trip duration", "Healthy/Warning/Critical".
- `FORMAT(..., "+0.0%;-0.0%", "en-US")` — locale forçado, padrão recomendado pela Microsoft para modelos globais (multi-país). Resolveu o bug onde o ponto era interpretado como separador de milhar em locale PT-BR.

### Hierarquia visual cromática (decisão de design)
- **Cards-sinalizadores (coloridos)**: Total Revenue (verde via YoY Color), Anomaly Rate (verde/amarelo/vermelho via Anomaly Color).
- **Cards informativos (neutros)**: Total Trips, Avg Fare per Trip, Avg Lead Time — texto preto.
- Razão: dashboards executivos 2025 (PwC, KPMG, BCG) seguem o padrão "1-2 sinalizadores + 3-4 neutros". 5 cards todos coloridos vira "árvore de Natal" — o olho não sabe onde focar primeiro.

### 3 insights novos descobertos na finalização (ouro pra apresentação)

**Insight 1 — Fare boom 2022→2023 (+33,1%)**
Avg Fare saltou de $21,74 → $28,94 em 1 ano. Causa: NYC TLC implementou aumento de congestion surcharge em 2022 com efeito acumulado em 2023.
- **Tradução supply chain**: efeito equivalente a "peak season surcharge" — sobretaxa logística aplicada uniformemente eleva custo unitário sem alterar volume de remessas.
- **Slide narrative**: "Em 2023, a operação experimentou um choque tarifário de +33% no ticket médio — análogo a um peak season surcharge em supply chain. Esse choque foi absorvido e estabilizou em 2024."

**Insight 2 — Fare plateau 2024 (-1,0%)**
Avg Fare 2024 = $28,64 vs 2023 = $28,94. Estabilização pós-choque.
- **Tradução supply chain**: revenue growth via volume expansion, não via price increase. Crescimento saudável e sustentável — em supply chain, o oposto seria "passar inflação pro cliente", que aumenta receita mas erode market share.
- **Slide narrative**: "2024 mostra estabilização tarifária. O crescimento de receita de +5,7% veio inteiramente do volume — sinal de demand recovery genuíno, não de inflação repassada."

**Insight 3 — Lead Time variability < 1% (GOLD — slide próprio merecido)**
Avg Lead Time: 17,46 min (2022) → 17,59 min (2023) → 17,63 min (2024). Variação de 0,17 min em 3 anos.
- **Tradução supply chain**: lead time previsível habilita forecast confiável, reduz necessidade de safety stock, viabiliza takt time planning. É exatamente o tipo de KPI que um S&OP global vê com prioridade.
- **Slide narrative**: "Em 3 anos, o lead time médio variou menos de 1%. Em supply chain, essa estabilidade é o que separa uma operação madura de uma reativa: forecast confiável, safety stock otimizado, takt time previsível. Para vocês na 3M Global Supply Chain, esse é o KPI que mais diz sobre maturidade operacional."

### Resposta de entrevista (Q: "Por que substituiu YoY Growth por Lead Time?")
"Identifiquei redundância: o card YoY Growth duplicava informação que já estava no subtítulo do Revenue card. Decidi trocar por um KPI estruturalmente alinhado com a narrativa do projeto — Yellow Taxi como cadeia de suprimentos urbana, onde trip = shipment e duration = lead time. O Avg Lead Time tem o bônus de ser um dos indicadores mais valorizados em supply chain global: lead time estável habilita forecast confiável e otimização de safety stock. É um KPI que conecta diretamente com a realidade do dia-a-dia de quem vai apresentar esse dashboard internamente na 3M."

### Storytelling visual completo da Página 1 (slide de overview)

> "A Página 1 entrega a primeira leitura executiva em 30 segundos: receita acumulada de $3,06 Bi crescendo +5,7%, 116 milhões de viagens, ticket médio estabilizado em $26,42 após o choque tarifário de 2023, lead time previsível em 17,56 min e taxa de anomalia operacional em 0,16% — toda a rede está saudável. A trendline confirma um padrão de recovery em três ondas. A concentração da receita em Manhattan (~75%) revela uma curva ABC clara, e o Top 10 OD lanes mostra JFK e LaGuardia como hubs premium dominantes. Cor verde sinaliza saúde — Revenue e Anomaly Rate são os dois cards-sinalizadores; os três do meio são informativos. Toda a tela responde aos filtros Year/Month/Time View e respeita as 6 RLS roles configuradas no semantic model."

---

## 12. Página 2 — Demand Deep Dive (dia 8)

### A ideia (por que esta página existe)
A Página 1 responde "como está o negócio" para o C-level. A Página 2 desce um nível: responde **quando e onde a demanda acontece**, para o analista de capacity planning / S&OP — quem dimensiona frota, turnos e capacidade por nó. Tradução supply chain: cada janela horária de cada zona é o "perfil de demanda de um CD"; conhecer os picos = saber onde alocar capacidade.

### Decisão de design
- **Canvas 1920×1080** — novo padrão das páginas 2-5 (a Página 1 era 1600×900; padronizamos pra cima para ganhar densidade sem amontoar).
- **Hero = heat map hora×dia.** A pergunta "quando?" se responde em 2 segundos pela cor. É o visual-assinatura de análise de demanda (mesma linguagem do GitHub contribution graph e dos heatmaps de demanda do Uber).
- **KPIs neutros (sem cor condicional).** Diferente da Página 1 (2 cards-sinalizadores). Demanda não tem bom/ruim — pico às 18h é fato, não meta. Cor fica reservada para onde carrega significado.
- **Field Parameters como vitrine self-service** na base — 1 visual = 25 análises, em vez de 25 visuais.

### Os 4 visuais e o porquê
1. **Heat map (Matrix + color scale)** — Matrix nativa com formatação condicional de fundo, não custom visual: zero dependência externa, performático, mainstream/defensável. Gradiente `#EAF1FB`→`#1F3864`. (cobre a feature Conditional Formatting nesta página)
2. **Sazonalidade (linha por ano)** — 3 linhas (2022/2023/2024) sobre os 12 meses mostram **forma sazonal + crescimento** num visual só. Para S&OP, é o annual demand plan.
3. **Top 10 zonas** — responde "onde nasce a demanda"; mais acionável que um mapa de 5 boroughs para capacity planning.
4. **Explorador (Field Parameters)** — o analista troca métrica × dimensão sozinho, sem pedir tela nova.

### Pontos de fala (roteiro de ~75s na demo)
> "Se a Página 1 é o painel do executivo, esta é a do planejador. O heat map à esquerda responde 'quando' em 2 segundos: os corredores escuros mostram o pico de fim de tarde nos dias úteis (quinta às 18h é o ápice, ~1,3 milhão de viagens) e, surpreendentemente, um segundo turno de madrugada no fim de semana (sábado e domingo, meia-noite às 3h). São dois regimes de demanda diferentes, que exigem capacidade diferente — como ter um turno noturno num CD. À direita, a sazonalidade confirma um padrão estável e crescente, ou seja, forecastável. Embaixo, qualquer pessoa fatia a demanda por borough, zona, vendor, pagamento ou faixa do dia trocando dois filtros. E 31% de toda a demanda se concentra em só 4 horas do dia — é aí que a capacidade aperta."

### Insights para slide (ouro descoberto na construção)
1. **Dois regimes de demanda.** Dia útil = pico de commute fim de tarde (18h). Fim de semana = pico de madrugada (0-3h, nightlife). Tradução SC: planejamento precisa de dois perfis de capacidade, não um.
2. **Quinta é o pico, não sexta** (Peak day = Thursday). Demanda corporativa/eventos antecede o fim de semana.
3. **31% da demanda em 4 horas/dia** (rush 7-9h e 17-19h). Alta concentração → surge capacity / staffing dinâmico.
4. **Sazonalidade estável + crescente** = forecast confiável → menor safety stock (mesma tese da estabilidade de lead time da Página 1).

### Respostas de entrevista (Q&A)
**Q: Por que Matrix e não um custom visual de heatmap?**
"Matrix nativa + formatação condicional de fundo entrega o mesmo resultado sem dependência de custom visual (risco de governança e performance). Caminho mainstream e defensável."

**Q: Por que os KPIs da Página 2 não têm cor condicional, se a Página 1 tinha?**
"Decisão consciente. Na Página 1, Revenue e Anomaly têm bom/ruim, então cor sinaliza. Demanda é neutra — pico às 18h não é bom nem ruim. Verde/vermelho aqui seria ruído."

**Q: Como garantiu a ordem Seg→Dom no heat map?**
"O Spark numera o dia da semana com domingo=1. Criei a coluna `day_order_mon` (segunda=1…domingo=7) na camada Gold — não como calculated column no relatório, porque o modelo é composite sobre DirectLake e coluna calculada em tabela remota não é suportada. Empurrei a lógica pro Gold (modelo fino) e configurei Sort by column no modelo semântico."

**Q: Field Parameters — por quê?**
"Reduz 25 visuais a 1. O usuário escolhe métrica × dimensão. Menos manutenção, mais auto-serviço, melhor performance."

---

## 13. Página 3 — Operations & Cost (dia 8)

### A ideia (por que esta página existe)
Depois do "quando e onde" (Página 2), a Página 3 responde **quão eficiente e quão caro** é cada viagem — a tela do gerente de operações e custo. Tradução supply chain: lead time = tempo de ciclo; distância×tarifa = custo proporcional ao serviço; composição da tarifa = cost-to-serve; revenue/mile = custo unitário; surcharges = peak-season.

### Decisão de design
- **Hero = scatter distância × tarifa** (a relação custo↔serviço, com outliers visíveis).
- **Volta a cor de sinalização** (vermelho `#D32F2F`) só na barra "Outlier" do histograma — duração outlier é ruim, sinalizar é correto. Resto neutro.
- **`duration_class` materializado no Gold** (espelha o UDF `ClassifyTripDuration`: 5/15/30/60/180 min) — performance em vez de classificar 115M linhas em DAX.
- **Cost/Mile usa o UDF `ComputeRevenuePerMile` ao vivo** no relatório.

### Os 4 visuais + os ajustes de leitura que fizemos
1. **Scatter** — agregado por zona (~260 pontos, performático). Ajuste-chave: filtro `is_anomalous = 0` + eixo X capado, senão zonas com viagens de 300mi achatavam 95% das bolhas no canto. Depois do ajuste, a relação linear fare↔distância aparece e os aeroportos saltam como outliers premium.
2. **Histograma** (`duration_class`) — Outlier em vermelho.
3. **Composição** — **barra empilhada** (1 barra, 4 segmentos). Trocamos de barras separadas (que viravam fiapos por causa das escalas diferentes) → a proporção do custo fica óbvia.
4. **Trend** — fare por mês, mostra o choque tarifário 2022→2023.

### Pontos de fala (~75s)
> "Esta é a tela de eficiência e custo. O scatter mostra que a tarifa cresce de forma quase linear com a distância — operação previsível — e os aeroportos aparecem como outliers premium, tarifa fixa de rota. À direita, o histograma prova a consistência do lead time: 78% das viagens entre 5 e 30 minutos e só 0,09% de outliers — cadeia madura. A composição abre o custo-to-serve de $26,42: 69% tarifa base, 19% taxas e sobretaxas, 12% gorjeta. E o trend conta a história do choque tarifário de 2023, com platô em 2024 — o crescimento passou a vir de volume, não de preço. Detalhe estratégico: o custo por milha caiu 18% no ano enquanto a distância média subiu 20% — viagens mais longas trazem melhor economia por milha."

### Insights para slide (ouro)
1. **Lead time consistente**: 78% em 5–30 min, 0,09% outliers → cadeia madura, forecast confiável.
2. **Composição do cost-to-serve** ($26,42/trip): ~69% base fare ($18,1), ~19% surcharges & taxes ($4,9), ~12% tip ($3,2).
3. **Choque tarifário +33% 2022→2023**, platô em 2024 → crescimento via volume, não preço.
4. **Economies of distance**: Cost/Mile −17,8% YoY enquanto distância +20,4% → viagens mais longas, melhor custo unitário.
5. **Speed −4,5% YoY** (11,44 mph) → congestionamento piorando, risco de throughput.

### Respostas de entrevista
**Q: Por que filtrar anomalias só no scatter?** "O scatter agrega média por zona; poucas viagens de 200-300mi distorciam a média de zonas inteiras e esticavam o eixo. Filtrar `is_anomalous=0` revela a relação real. As anomalias continuam visíveis no histograma (Outlier) e na Página 5 de Data Quality — cada tela trata o outlier no contexto certo."

**Q: Por que barra empilhada na composição?** "Com 4 componentes de escalas muito diferentes ($18 vs $1), barras separadas ficam ilegíveis. Empilhada mostra a proporção do custo-to-serve numa leitura só."

**Q: Cost/Mile via UDF?** "`ComputeRevenuePerMile(revenue, distance)` é uma UDF DAX com guard de divisão por zero. Uso ela ao vivo na medida — UDF reaproveitável de verdade, não decorativa."

---

## 14. Página 4 — Demand vs Demographics (dia 8)

### A ideia
Peça obrigatória do briefing: fonte externa (US Census ACS) correlacionada com a demanda. Pergunta: a demanda acompanha tamanho/riqueza da região e onde há mercado sub-servido? Tradução SC: penetração de mercado; sub-indexado = oportunidade.

### Decisão de design (por que evitamos scatter E mapa)
Não repeti o scatter (já usado na Pág. 3) — variedade visual conta na avaliação. Tirei também o mapa: no grão borough (5 pontos) ele é pobre e arriscado no geocoding. Resultado: 4 tipos distintos — barra divergente, combo, treemap, matrix.

### Os 4 visuais
1. **Barra divergente** (Demand Index pp = Revenue share − Population share): hero / punchline de negócio.
2. **Combo** (revenue/capita coluna + median income linha): a demanda segue a renda?
3. **Treemap** (revenue share): concentração / curva ABC.
4. **Matrix scorecard**: detalhe por borough, data bars + índice colorido.

### Gotchas resolvidos
- Boroughs-lixo (Unknown/N/A/EWR) poluíam os 4 visuais → **filtro no nível da página**.
- Medidas têm de ser criadas **uma de cada vez** (colar o bloco inteiro num único measure dá erro de sintaxe "Median").

### Pontos de fala (~70s)
> "Esta peça cruza a operação com uma fonte externa, o Census. A barra divergente resume: Manhattan captura 55 pontos percentuais a mais de receita do que seu tamanho populacional sugere — over-served; Brooklyn é o oposto, −29pp, o maior mercado sub-servido. O combo mostra que renda explica só parte: a correlação é 0,43, moderada — a demanda é puxada por negócios e turismo, não pela riqueza do morador. O treemap fecha com a concentração: dois boroughs respondem por ~93% da receita, curva ABC clássica. Caveat honesto: a per-capita de Manhattan é inflada porque os passageiros lá não são residentes."

### Insights de ouro
1. Manhattan +55pp over-served; Brooklyn −29pp sub-servido (maior oportunidade).
2. r = 0,43 — demanda só parcialmente segue renda (negócios/turismo).
3. ABC: Manhattan + Queens ≈ 93% da receita.
4. Caveat: per-capita de Manhattan inflada por não-residentes.

### Q&A
- **Por que treemap, não mapa/scatter?** Scatter já usado na 3; mapa borough-grain é pobre; treemap traz part-to-whole + framing ABC.
- **Como mediu over/under-served?** Revenue share − Population share (pp).
- **Correlação em DAX?** Pearson manual sobre boroughs (n=5, direcional — limitação do grão ACS, documentada).

---

## 15. Página 5 — Data Quality & Governance (dia 8)

### A ideia
Tela "backstage" que prova **confiança e governança** — e fecha o 6º feature (RLS). Pergunta: posso confiar nesses dados e quem vê o quê? Sem ela o relatório é "bonito"; com ela vira "auditável".

### Os 4 visuais
1. **Discard breakdown** (bar): por que 2,84% saíram — cada motivo rastreável.
2. **Anomaly rate por borough** (column, cor condicional): onde está o ruído.
3. **RLS — visão por gestor** (bar): prova do Row-Level Security.
4. **Top anomalies by zone** (table agregada): evidência concreta.

### Pontos de fala (~70s)
> "Esta é a tela da confiança. Retivemos 97,16% das 119 milhões de linhas, e cada descarte tem motivo explícito — distância ≤ 0, tarifa negativa. As 0,16% de anomalias não foram apagadas: ficaram marcadas e mapeadas. Staten Island tem a maior taxa, 0,98%, oito vezes Manhattan — quanto menor o volume, mais ruído relativo. À direita, o RLS: 6 roles no Fabric; cada gerente só vê o próprio borough, o global vê os $3,06 Bi. Segurança na camada do modelo, não no visual."

### Insights de ouro
1. 97,16% retido; descarte ~95% explicado por distância ≤ 0 + tarifa negativa.
2. Anomalia inversamente proporcional ao volume: Staten Island 0,98% vs Manhattan 0,12%.
3. Dois perfis de anomalia: aeroportos = muitas e pequenas; Manhattan = poucas e extremas ($401k em Gramercy).
4. RLS escopa cada gestor; soma dos boroughs ≈ global (gap = zonas não-NYC).

### Q&A
- **Como o RLS funciona no composite?** Roles no semantic model do Fabric; o composite não mostra roles remotas no "Exibir como", então uso medidas "Manager View" pra demonstrar o efeito.
- **Por que a soma do descarte (3,55M) > líquido (3,38M)?** Overlap: linhas que falham mais de um critério.
- **Por que agregou a tabela de anomalias por zona?** Sem ID único de trip, não dá pra listar trips individuais nativamente; mostro a maior anomalia por zona.

---

## Página 6 — Weather Impact & Operational Risk

### A ideia
Tela que fecha o loop das fontes externas: os dados NOAA ingeridos no Bronze, enriquecidos no Gold, finalmente aparecem como análise de negócio. Duas perguntas em uma tela: *qual estação cria mais pressão operacional?* e *eventos climáticos extremos realmente afetam o ciclo?*

### Os visuais
1. **V1 — Season combo chart (hero)**: barras de throughput (Avg Daily Trips) + linha de lead time por estação. Spring lidera em volume (113k/dia), Fall é a estação de estresse — 2º volume COM o pior lead time (18,46 min).
2. **V2 — Extreme weather mini-charts** (3 charts lado a