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

## 8. Features Power BI obrigatórias — justificativa de cada uma

### Time Intelligence + Calculation Group
**Por que Calc Group em vez de criar medidas YoY, YTD, PY separadas?**

"Sem Calculation Group, eu teria que duplicar cada KPI 7 vezes (Current, PY, YoY, YoY%, YTD, PYTD, MAT) — para 8 KPIs base = 56 medidas. Com Calc Group, mantenho 8 medidas e o grupo aplica time intelligence dinamicamente via SELECTEDMEASURE. Modelo enxuto, manutenção drástica. É o padrão de bibliotecas DAX maduras."

### Field Parameters
**Por que Field Parameters em vez de criar 1 visual por métrica?**

"Field Parameters permitem que o usuário final escolha qual métrica está em foco (Trips, Revenue, Distance, Duration) e qual dimensão analisa por (Borough, Zone, Vendor, Payment, Day Part). Em vez de duplicar visuais, eu tenho 1 visual reativo a 2 slicers. Reduz drasticamente o número de páginas e dá poder de exploração ao analista."

### Conditional Formatting
**Aplicação:** heat map zona × hora, semáforos em KPI cards com YoY, data bars em rankings.

### Row-Level Security (RLS)
**Por que 5 roles por Borough?**

"Para simular o cenário Supply Chain de 'regional manager só vê sua região'. Em produção 3M seria 'site manager vê só sua planta'. Governança por padrão, demonstrada via View as no Power BI Desktop."

### UDFs (DAX User-Defined Functions)
**Por que UDFs em vez de copiar lógica em cada medida?**

"UDFs (recurso 2025+ do Power BI) permitem encapsular lógica reutilizável. Criei 3: `ClassifyTripDuration`, `ClassifyFareTier`, `ComputeRevenuePerMile`. Reduz duplicação de DAX e centraliza regras de negócio (ex.: o threshold do que é 'long trip' fica em UMA função, não em N medidas). É a mesma filosofia de DRY que aplico em Python."

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

## Storytelling final (fechamento da apresentação)

> "O briefing pediu um relatório Power BI. Eu entendi que estava pedindo algo mais — uma demonstração de como eu trabalho. Por isso entreguei: pipeline end-to-end no Fabric (mesma stack da 3M), schema canônico controlado, star schema documentado, 6 features Power BI obrigatórias mais 3 UDFs reutilizáveis, fonte externa correlacional, RLS para governança, repo público versionado desde o dia 1 e plano B funcional para continuidade. Cada decisão técnica está justificada e rastreável. O que vocês vão ver no relatório é só a ponta do iceberg — a fundação está toda no repo."
