# 🎯 Kit de Domínio — Apresentação 3M TechCase (NYC TLC)

> **Objetivo:** te deixar com o projeto na ponta da língua para falar com fluência e segurança, com ou sem o dashboard na tela.
> **Como usar:** leia em voz alta a Seção 0 (cola) + Seções 1 e 2 até decorar números e narração. Treine os flashcards (Seção 3) tampando as respostas. Faça o simulado com a bateria de Q&A (Seção 4). Na véspera, só a Seção 0 + Seção 5 (frases de abrir/fechar).
> **Regra de ouro:** nada vai pra sua boca que você não saiba explicar em 1 frase. Se decorar só uma coisa, decore a Seção 0.
> ⚡ **ATUALIZADO (P6):** agora são **6 telas**. A Página 6 (Weather/NOAA) é a **2ª fonte externa**.

---

## 0. A COLA — os números que você NUNCA pode errar

**Headline executivo (Página 1):**

| KPI | Valor | Como falar |
|---|---|---|
| Receita total (2022–2024) | **US$ 3,06 Bi** | "três bilhões e sessenta milhões" |
| Viagens | **116 milhões** | (115,76M exatos) |
| Crescimento de receita 2024 | **+5,7% YoY** | vs. 2023 |
| Ticket médio | **US$ 26,42** | estabilizado após o choque de 2023 |
| Lead time médio | **17,56 min** | variou <1% em 3 anos |
| Taxa de anomalia | **0,16%** | 180k viagens sinalizadas, não apagadas |
| Retenção de dados | **97,16%** | de 119M brutas → 116M limpas |
| Concentração (ABC) | **Manhattan ≈ 75%** da receita | JFK + LaGuardia lideram |

**Receita por ano:** 2022 = $844M → 2023 = $1.076M (**+27,4%**, recovery pós-pandemia) → 2024 = $1.137M (**+5,7%**, estabilização).

**Ticket médio por ano:** $21,74 (2022) → **$28,94 (2023, +33,1% = choque tarifário)** → $28,64 (2024, platô).

**Lead time por ano:** 17,46 → 17,59 → 17,63 min (variação de **0,17 min em 3 anos**).

**Pipeline em 3 números:** Bronze **119,1M** linhas → Silver **115,76M** (descarte 2,84%) → Gold **1 fato + 6 dimensões**.

**🌦️ Clima (Página 6, NOAA — 2ª fonte externa):** Spring **113k/dia** (17,44 min) · Summer **101k** (17,50) · **Fall 109k (18,46 — pior lead time)** · Winter **100k** (16,75). Total 105,6k/dia · 17,56 min. Extremo: demanda **−36%** (106k→68k), lead time **−3,8 min** (mais rápido), só **4 dias extremos em 1.096**.

**Os 6 "insights de ouro"** (se só puder citar seis):
1. **Lead time estável <1% em 3 anos** → forecast confiável, menos safety stock. *(o KPI mais "supply chain" do projeto)*
2. **Choque tarifário +33% em 2023, platô em 2024** → crescimento por volume, não por preço.
3. **ABC:** Manhattan + Queens ≈ **93% da receita**; JFK + LaGuardia = **46% do top 10 zonas**.
4. **Dois regimes de demanda:** rush de fim de tarde (dia útil) + madrugada de fim de semana; **31% em 4h/dia**.
5. **Demanda × renda = 0,43** (correlação moderada) → puxada por negócio/turismo, não pela renda do morador.
6. **🌦️ Clima (NOAA, 2ª fonte externa):** Outono = **estação de estresse** — 2º volume (109k/dia) com o **pior lead time (18,46 min)**; em clima extremo a demanda cai **36%** mas o ciclo fica **mais rápido** → **auto-seleção** da demanda.

---

## 1. Pitch geral (decore os dois)

### ⏱️ Versão 30 segundos (abertura / elevator)
> "O briefing pedia um relatório de Power BI sobre os táxis de Nova York. Eu li como um pedido pra mostrar *como eu trabalho* de ponta a ponta. Então tratei o táxi como uma cadeia de suprimentos urbana — cada viagem é uma remessa, a duração é o lead time, as zonas são os nós da rede. Construí um pipeline completo no Microsoft Fabric, a mesma stack de vocês, modelei um star schema com 116 milhões de viagens e entreguei 6 telas que vão do executivo ao backstage de governança — com **duas** fontes externas correlacionadas, demografia e clima. Tudo versionado no GitHub desde o dia 1."

### ⏱️ Versão 2 minutos (resumo da reunião)
> "O problema: a TLC publica dados públicos de 116 milhões de corridas, mas crus — schema inconsistente entre meses, datas vazadas até de 2026, tarifas negativas. Meu primeiro trabalho foi transformar esse ruído em dado confiável.
>
> Pra isso montei um pipeline medallion no Fabric: **Bronze** guarda o dado cru intacto, **Silver** limpa e tipa — retive 97,16% e documentei cada descarte —, e **Gold** é o modelo dimensional, um star schema com 1 fato e 6 dimensões, enriquecido com duas fontes externas: demografia do Census e clima da NOAA. O Power BI lê esse modelo via **DirectLake**, sem copiar dado.
>
> Em cima disso, entreguei as 6 features que o teste pedia — time intelligence, field parameters, calculation groups, conditional formatting, RLS e UDFs em DAX — mais 3 funções reutilizáveis minhas. As 6 telas contam uma história: visão executiva, onde e quando a demanda acontece, eficiência e custo, demanda versus demografia, governança com segurança por linha, e o impacto do clima no risco operacional.
>
> E o tempo todo eu traduzo pra lógica de supply chain: lead time estável habilita forecast; a concentração em Manhattan é uma curva ABC clássica; o choque tarifário de 2023 é um peak-season surcharge; o clima extremo é uma disrupção que reduz volume e muda o perfil da rede. O dashboard é a ponta do iceberg — a fundação inteira, com cada decisão justificada, está no repositório."

---

## 2. Roteiro da arquitetura (narração fluida do Fabric)

**O fluxo em 1 frase:** *"Fontes públicas → ingestão PySpark (Bronze) → limpeza Spark SQL (Silver) → star schema (Gold) → modelo semântico DirectLake → Power BI com as features e a governança."*

### O pipeline, camada por camada
- **Bronze (cru, intacto):** ingestão em **PySpark** dos 36 parquets mensais da TLC + lookups + enrichment. Princípio: nada se descarta nem se altera; é a fonte da verdade. Idempotente — re-roda sem duplicar.
- **Silver (limpo, tipado):** **Spark SQL**. Filtro temporal (2022–2024), descartes de qualidade (distância ≤ 0, tarifa negativa, duração inválida), campos derivados (`duration_min`, `avg_speed_mph`) e a flag `is_anomalous` (sinaliza, não apaga). De 119,1M → **115,76M** linhas.
- **Gold (modelado):** **star schema** — `fact_trips` (115,76M) + 6 dimensões: `dim_date` (1.096 dias, com clima NOAA), `dim_time` (24h), `dim_zone` (265 zonas, com demografia ACS), `dim_vendor` (5), `dim_ratecode` (7), `dim_payment` (7). Zero foreign keys órfãs.

### Os "porquês" que a banca vai cobrar (1 linha cada)
- **Por que Fabric?** É a stack da 3M. Demonstrar fluência nela > entregar em ferramenta isolada. E Lakehouse + Delta + DirectLake é arquitetura zero-copy moderna.
- **Por que PySpark e não Pandas?** 116M linhas não cabem em Pandas single-node. Spark escala de MB a TB sem reescrever código.
- **Por que medallion?** Separação de responsabilidades + reprocessar uma camada sem tocar nas outras + padrão de mercado (qualquer DE entende).
- **Por que DirectLake (não Import/DirectQuery)?** Import duplica dado; DirectQuery sofre latência em 116M linhas; **DirectLake lê o Delta direto do OneLake — performance de Import, sem cópia.**
- **Por que star schema (não snowflake/flat)?** O motor Vertipaq otimiza pra star; Field Parameters e Calc Groups funcionam melhor com dimensões denormalizadas.
- **Por que role-playing na Zone (não USERELATIONSHIP)?** Legibilidade — quem abre o modelo vê "Pickup Zone" e "Dropoff Zone"; usuário final não precisa aprender USERELATIONSHIP nas medidas.

### Os 6 features obrigatórios — onde cada um está
| # | Feature | Implementação | Onde aparece |
|---|---|---|---|
| 1 | **Time Intelligence** | Calc Group "Time Analytics", 7 items (Current/PY/YoY/YoY%/YTD/PYTD/MAT) via Tabular Editor 2 | Página 1 |
| 2 | **Field Parameters** | `FP_Metric` (5 medidas) × `FP_Dimension` (5 dims) = 25 análises em 1 visual | Página 2 |
| 3 | **Calculation Groups** | o próprio "Time Analytics" — 10 medidas × 7 items = 70 perspectivas | Página 1 |
| 4 | **Conditional Formatting** | heat map, data bars, ícones de semáforo, season matrix | Todas |
| 5 | **RLS** | 6 roles (5 borough managers + global) | Página 5 |
| 6 | **UDFs DAX** | 3 funções: `ClassifyTripDuration`, `ClassifyFareTier`, `ComputeRevenuePerMile` | Página 3 |

### Enrichment — o padrão "API + Snapshot" (ótima resposta)
- **NOAA (clima, diário)** → REST API com paginação anual. É dado **operacional** (muda todo dia).
- **ACS (demografia, anual)** → snapshot versionado por borough. É **master data** (estável).
- *Frase:* "Dado operacional via API, dado de referência via snapshot versionado — misturar os dois é erro comum."
- *Jogada de avaliação:* "O briefing pedia **uma** fonte externa. Entreguei **duas**, cada uma com tela própria — ACS na Página 4 (renda/densidade = demand drivers) e NOAA na Página 6 (clima = risco operacional)."

### O composite model (a decisão que surgiu na execução)
> "Mantive o `sm_yellow_taxi_3m` como fonte da verdade no Fabric. No Power BI, em Live connection puro não dá pra editar o modelo, então converti pra **composite** em DirectQuery — isso me deixou criar medidas, UDFs e RLS locais **sem importar os 116M de linhas** pro .pbix. Trade-off: leve perda de performance vs. DirectLake puro, em troca de flexibilidade na camada de relatório. Em produção, é o padrão que deixa cada time customizar sem mexer no modelo central."

---

## 3. Flashcards das 6 telas (tampe a resposta e treine)

### 🟦 Tela 1 — Executive Overview
- **Para quem / pergunta:** C-level. "Como está o negócio em 30 segundos?"
- **Pitch 30s:** "Receita acumulada de $3,06 Bi crescendo +5,7%, 116M de viagens, ticket estabilizado em $26,42 após o choque de 2023, lead time previsível de 17,56 min e anomalia de 0,16%. A rede está saudável. A trendline mostra recovery em três ondas e a receita concentra ~75% em Manhattan — curva ABC clara."
- **Números:** $3,06 Bi · 116M · +5,7% · $26,42 · 17,56 min · 0,16% · Manhattan ~75%.
- **Gancho SC:** trip = shipment, duration = **lead time** (substituí o KPI "YoY Growth", redundante, por Lead Time — o KPI hero universal de supply chain).
- **Visuais:** 5 KPI cards (2 sinalizadores coloridos + 3 neutros), trendline, ABC Manhattan, Top 10 O-D lanes.
- **Q prováveis:**
  - *Por que trocou YoY Growth por Lead Time?* → "YoY já estava no subtítulo do Revenue; troquei por um KPI estruturalmente alinhado: lead time = lead time de supply chain, habilita forecast e otimiza safety stock."
  - *Por que só 2 cards coloridos?* → "Padrão de dashboard executivo: 1–2 sinalizadores + 3–4 neutros. Tudo colorido vira 'árvore de Natal', o olho não sabe onde focar."

### 🟦 Tela 2 — Demand Deep Dive
- **Para quem / pergunta:** planejador de capacity / S&OP. "Quando e onde a demanda acontece?"
- **Pitch 30s:** "O heat map hora×dia responde 'quando' em 2 segundos: pico de fim de tarde nos dias úteis — quinta às 18h é o ápice, ~1,3M de viagens — e um segundo turno de madrugada no fim de semana. Dois regimes de demanda, duas capacidades diferentes. 31% de toda a demanda cabe em 4 horas do dia."
- **Números:** quinta 18h = pico (~1,3M) · Afternoon+Evening = 70% do volume · 31% em 4h/dia.
- **Gancho SC:** cada janela horária de cada zona = perfil de demanda de um CD; conhecer os picos = saber onde alocar capacidade.
- **Visuais:** heat map (Matrix + cond. formatting), sazonalidade (3 linhas/ano), top 10 zonas, explorador Field Parameters.
- **Q prováveis:**
  - *Por que Matrix e não custom visual de heatmap?* → "Mesmo resultado, sem dependência de custom visual (risco de governança/performance). Caminho mainstream e defensável."
  - *Como garantiu a ordem Seg→Dom?* → "O Spark numera domingo=1; criei `day_order_mon` no **Gold**, não como calc column no relatório, porque coluna calculada em tabela remota não é suportada no composite/DirectLake."

### 🟦 Tela 3 — Operations & Cost
- **Para quem / pergunta:** gerente de operações e custo. "Quão eficiente e quão caro é cada viagem?"
- **Pitch 30s:** "O scatter mostra a tarifa crescendo quase linear com a distância — operação previsível — e os aeroportos como outliers premium. 78% das viagens entre 5 e 30 min, só 0,09% outliers: cadeia madura. O custo-to-serve de $26,42 se abre em 69% tarifa base, 19% taxas, 12% gorjeta. E o trend conta o choque tarifário de 2023 com platô em 2024 — crescimento por volume, não por preço."
- **Números:** 78% em 5–30 min · composição 69/19/12 · choque +33% · Cost/Mile −17,8% YoY com distância +20,4% · EWR Rev/Mile $24,05 (~5× Manhattan).
- **Gancho SC:** lead time = tempo de ciclo; composição da tarifa = cost-to-serve; revenue/mile = custo unitário; surcharge = peak-season.
- **Visuais:** scatter dist×fare, histograma `duration_class`, barra empilhada de composição, trend de tarifa.
- **Q prováveis:**
  - *Por que filtrar anomalias só no scatter?* → "O scatter agrega média por zona; poucas viagens de 200–300 mi distorciam zonas inteiras. Filtrei `is_anomalous=0` ali; as anomalias continuam visíveis no histograma e na Tela 5."
  - *Cost/Mile via UDF?* → "`ComputeRevenuePerMile(revenue, distance)`, com guard de divisão por zero. Uso ao vivo — UDF reaproveitável de verdade, não decorativa."

### 🟦 Tela 4 — Demand vs Demographics (fonte externa #1 · ACS)
- **Para quem / pergunta:** estratégia de mercado. "A demanda acompanha tamanho/riqueza da região? Onde há mercado sub-servido?"
- **Pitch 30s:** "A barra divergente resume: Manhattan captura 55 pontos percentuais a mais de receita do que seu tamanho populacional sugere — over-served; Brooklyn é o oposto, −29pp, o maior mercado sub-servido. A renda explica só parte: correlação de 0,43, moderada — a demanda é puxada por negócio e turismo, não pela renda do morador."
- **Números:** Manhattan +55pp · Brooklyn −29pp · r = 0,43 · ABC Manhattan+Queens ≈ 93%.
- **Gancho SC:** penetração de mercado; sub-indexado = oportunidade; é a análise de **demand drivers** (que fatores estruturais explicam volume por região).
- **Visuais:** barra divergente (Demand Index = Revenue share − Population share), combo (revenue/capita + renda), treemap (ABC), matrix scorecard.
- **Q prováveis:**
  - *Correlação em DAX?* → "Pearson manual sobre os boroughs. n=5, então é **direcional** — limitação do grão do ACS, que documentei."
  - *Por que treemap e não mapa?* → "Mapa no grão borough (5 pontos) é pobre e arriscado no geocoding; treemap traz part-to-whole + framing ABC. Variedade visual também conta."
  - *Caveat honesto:* "A per-capita de Manhattan é inflada porque os passageiros lá não são residentes — eu digo isso antes de perguntarem."

### 🟦 Tela 5 — Data Quality & Governance
- **Para quem / pergunta:** auditoria / governança. "Posso confiar nesses dados e quem vê o quê?"
- **Pitch 30s:** "Retivemos 97,16% das 119M de linhas, e cada descarte tem motivo explícito — distância ≤ 0, tarifa negativa. As 0,16% de anomalias não foram apagadas, foram marcadas e mapeadas. À direita, o RLS: 6 roles no Fabric; cada gerente vê só o próprio borough, o global vê os $3,06 Bi. Segurança na camada do modelo, não no visual."
- **Números:** 97,16% retido · anomalia inversa ao volume (Staten Island ~1% vs Manhattan 0,1%) · EWR 16,7% / N/A 23,2% (core vs exception lanes) · top anomalia $401k em 10 min (Gramercy).
- **Gancho SC:** anomalia = transação suspeita/auditoria; core vs exception lanes = main lanes vs special routes (SLA/contrato diferente); RLS = "site manager vê só sua planta".
- **Visuais:** discard breakdown, anomaly rate por borough (cond. formatting), RLS por gestor, top anomalies por zona.
- **Q prováveis:**
  - *Como o RLS funciona no composite?* → "Roles no semantic model remoto não aparecem no 'Exibir como'; uso medidas 'Manager View' pra demonstrar o efeito. Em produção, ligaria as roles a grupos do Azure AD."
  - *Por que a soma dos descartes > líquido?* → "Overlap: linhas que falham mais de um critério ao mesmo tempo."

### 🟦 Tela 6 — Weather Impact & Operational Risk (2ª fonte externa · NOAA)
- **Para quem / pergunta:** risco operacional / S&OP. "Choques externos (clima) afetam throughput e lead time?"
- **Pitch 30s:** "Esta é a segunda fonte externa, o clima do NOAA. Duas descobertas. Sazonalidade: o Outono é a estação de estresse — 2º maior volume, 109 mil viagens/dia, com o pior lead time, 18,46 min, +1,71 acima do inverno. Alto volume × ciclo lento = cost-to-serve máximo. A Primavera é o pico real, 113 mil/dia, mas eficiente. E o contraintuitivo: em dias de clima extremo — só 4 em 1.096 — a demanda cai 36%, mas o lead time fica mais rápido e a tarifa cai. É auto-seleção: só as viagens essenciais acontecem, e com menos trânsito o ciclo encurta."
- **Números:** Spring 113k/dia (17,44) · Summer 101k (17,50) · **Fall 109k (18,46, pior)** · Winter 100k (16,75) · Total 17,56 · Extremo: −36% demanda · −3,8 min lead time · 4 dias em 1.096.
- **Gancho SC:** sazonalidade = peak-season / planejamento de capacidade (Outono = janela de reforço, Inverno = janela de manutenção); choque extremo = disrupção que reduz volume e muda o perfil operacional.
- **Visuais:** season combo (throughput em barras + lead time em linha), 3 mini-charts Normal vs Extreme (demanda, lead time, tarifa), season matrix e insight boxes. (Conditional formatting na matrix.)
- **Q prováveis:**
  - *Por que clima é boa 2ª fonte externa?* → "Correlaciona com lead time e demanda — paralelo direto de uma disrupção em supply chain. E tem padrão de ingestão diferente do ACS: API diária vs snapshot anual."
  - *Lead time MENOR em clima extremo não é estranho?* → "É o ponto: a demanda despenca, o trânsito some, e só ficam as viagens essenciais — o ciclo encurta. Auto-seleção. Em SC, é a disrupção que tira volume mas deixa a rede mais fluida pra quem resta."
  - *Por que enriquecer `dim_date` e não criar `dim_weather`?* → "Clima é atributo do dia, não entidade própria. Grão diário por estação meteorológica de NYC, ligado na `dim_date`."

---

## 4. Bateria de Q&A (técnico + negócio)

### 🔧 Técnicas
1. **Por que Fabric e não Snowflake/Databricks/DuckDB?** → "Fabric é a stack da 3M; mostrar fluência nela é o mais relevante. Lakehouse+Delta+DirectLake é zero-copy moderno, e o trial cobre o teste."
2. **DirectLake vs Import vs DirectQuery?** → "Import duplica, DirectQuery tem latência em 116M linhas, DirectLake lê Delta direto do OneLake — performance de Import sem cópia. Trade-off: é preview, então testei antes que Calc Groups, UDFs e Field Parameters convivem."
3. **Como tratou o schema drift da TLC?** → "VendorID alterna INT/BIGINT entre meses e `airport_fee` muda capitalização em 2024. `mergeSchema` falha. Defini um schema canônico de 19 campos, leio arquivo a arquivo com lookup case-insensitive e `cast()`. Auditável e seguro — mesma estratégia que uso com extrato de SAP."
4. **Como garante integridade referencial?** → "Padrão Kimball: cada dimensão tem uma linha 'Unknown' e uso `COALESCE(key, unknown_key)` em toda FK. Zero órfãs mesmo com vendor desconhecido."
5. **RLS num composite model — como?** → "As roles vivem no semantic model do Fabric; o composite não as mostra no 'Exibir como', então demonstro com medidas 'Manager View'. Produção: Azure AD groups."
6. **Performance e escalabilidade?** → "Star schema + DirectLake + partição por year/month no fato. Empurrei classificações pesadas (`duration_class`, `day_order_mon`) pro Gold em vez de calcular 116M linhas em DAX. Spark escala horizontal sem reescrever código."
7. **Por que UDFs em DAX?** → "Encapsulei lógica reutilizável (faixas de duração, tier de tarifa, revenue/mile). Troco em 1 lugar, propaga pra todas as medidas. Achei 3 quirks da sintaxe nova e documentei no repo."
8. **Calculation Group — qual o ganho?** → "Em vez de criar 7 versões de cada medida, 1 grupo aplica time intelligence via `SELECTEDMEASURE()`. 10 medidas × 7 items = 70 perspectivas com manutenção mínima."
9. **Como escalaria isso na 3M de verdade?** → "Ingestão via Fabric Data Pipeline/ADF schedulado, transformação em PySpark ou dbt, orquestração com Azure DevOps (XMLA endpoint pra CI/CD do .pbix) e governança/linhagem no Microsoft Purview."
10. **E se o Fabric cair no dia?** → "Tenho plano B: pipeline equivalente em Python+Polars local, mesmo schema, saída em Parquet, Power BI em Import. Continuidade de negócio — mandatório em supply chain."

### 💼 Negócio / Supply Chain
11. **A base é de táxi, e daí pra 3M?** → "Táxi é uma cadeia de suprimentos urbana: viagem = remessa, duração = lead time, zonas = nós, tarifa = cost-to-serve. As perguntas são as mesmas — demanda, eficiência, custo, gargalo, concentração de risco."
12. **Qual o insight mais valioso pra um time de supply chain?** → "Lead time estável <1% em 3 anos. Estabilidade de lead time é o que separa operação madura de reativa: habilita forecast confiável, reduz safety stock, viabiliza takt time. É o KPI que um S&OP global olha primeiro."
13. **Como usaria a concentração ABC?** → "Manhattan + Queens = 93% da receita; JFK + LaGuardia = 46% do top 10. É risco de concentração — onde investir capacidade, onde ter contingência se uma lane cair."
14. **'Core vs exception lanes' — o que muda na prática?** → "Manhattan core opera a 0,1% de anomalia; zonas de fronteira (EWR, N/A) chegam a 17–23%. Operação consolidada vs. de exceção pede processo, SLA e contrato diferentes. One-size-fits-all não funciona em escala."
15. **O choque tarifário de 2023 — o que ensina?** → "+33% no ticket, depois platô. É um peak-season surcharge: sobretaxa eleva custo unitário sem mudar volume. Em 2024 o crescimento veio de volume, não de preço — recovery genuíno, não inflação repassada."
16. **Por que enriquecer com Census e clima?** → "ACS responde 'demanda segue renda/densidade?' — análise de demand drivers. NOAA é risco operacional: choques externos (clima) afetam lead time e demanda, igual a disrupções em supply chain. O briefing pedia uma fonte externa; entreguei duas."
17. **🌦️ O que o clima ensina pra supply chain?** → "Sazonalidade tem perfis operacionais distintos: Outono é estresse (volume alto + ciclo lento = cost-to-serve máximo → reforçar capacidade), Inverno é janela de manutenção. E o choque extremo reduz volume mas muda o perfil da rede — em SC, é planejar para disrupção, não só para o pico."

### ⚠️ Limitações (diga você, antes que perguntem — passa maturidade)
- **Correlação n=5 boroughs** → direcional, não estatisticamente robusta. Limitação do grão do ACS, documentada.
- **ACS no nível borough, não zone** → decisão consciente; em produção faria spatial join NTA↔Zone via Geopandas (95% da insight com 5% do esforço).
- **Per-capita de Manhattan inflada** por passageiros não-residentes.
- **Composite model** tem leve perda de performance vs. DirectLake puro — troca deliberada por flexibilidade.
- **Clima no grão diário / estação meteorológica de NYC** (não por zona) → suficiente pra sazonalidade e extremos; granularidade fina exigiria estações por região.

---

## 5. Frases de abrir e fechar (decore palavra por palavra)

**Abertura (framing):**
> "O briefing pedia um relatório de Power BI. Eu entendi como um pedido pra mostrar *como eu trabalho*. Então tratei os táxis de Nova York como uma cadeia de suprimentos urbana — e construí a solução de ponta a ponta na mesma stack que vocês usam."

**Fechamento:**
> "Cada decisão técnica está justificada e rastreável no repositório. O que vocês viram no relatório é a ponta do iceberg — pipeline no Fabric, schema canônico controlado, star schema documentado, as 6 features mais 3 UDFs, duas fontes externas correlacionadas, RLS pra governança e plano B pra continuidade. A fundação inteira está versionada desde o dia 1."

---

## 6. Mapa de tradução Táxi → Supply Chain (cola de bolso)

| No projeto (táxi) | Na 3M (supply chain) |
|---|---|
| Viagem | Remessa / ordem |
| Duração da viagem | **Lead time / tempo de ciclo** |
| Zona de pickup/dropoff | Origem-destino (O-D lane) / nó da rede |
| Demanda por hora/zona | Perfil de demanda por CD / capacity planning |
| Composição da tarifa | Cost-to-serve |
| Revenue per mile | Custo unitário |
| Congestion surcharge | Peak-season surcharge |
| Viagem anômala ($401k) | Transação suspeita → auditoria/fraude |
| Concentração Manhattan/aeroportos | Curva ABC — lanes/SKUs que carregam o EBITDA |
| Manhattan core vs EWR/N/A | Main lanes vs special-case routes (SLA/contrato distintos) |
| Lead time estável | Forecast confiável → menor safety stock, takt time |
| ACS (renda/densidade) | Demand drivers estruturais por região |
| NOAA (clima) | Choques externos (clima, disrupção) → planejamento de risco |
| Estação de estresse (Fall) | Pico sazonal que exige reforço de capacidade |
| Clima extremo (auto-seleção) | Disrupção que reduz volume e muda o perfil da rede |

---

*Fonte de tudo: `docs/PRESENTATION_NOTES.md`, `ARCHITECTURE.md`, `CODE_EXPLAINED.md`. Números da Página 6 (Weather) conferidos via screenshot do `.pbix` em 08/06.*

---

## 7. Defesa técnica dos 6 features (slide 8) — como desenvolvi + como testo

> Esta seção é pra **banca técnica**. No slide 8 não leia os 6 cards: diga *"cobri os 6 exigidos, e validei cada um — não só liguei a opção, testei o resultado"* e deixe a banca puxar. Tenha estes detalhes prontos pra ir fundo em qualquer um.

### 1. Time Intelligence
- **Onde uso:** Página 1 (Exec) — reage ao girar o filtro de ano.
- **Como desenvolvi:** dentro do Calculation Group "Time Analytics" (Tabular Editor 2). Pré-requisito: `dim_date` marcada como **Date Table**, coluna de data = `full_date` (evitei `date`, palavra reservada).
- **Como funciona:** cada item transforma a medida ativa via `SELECTEDMEASURE()`:
  - `PY = CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR(dim_date[full_date]))`
  - `YTD = TOTALYTD(SELECTEDMEASURE(), dim_date[full_date])`
  - `MAT = CALCULATE(SELECTEDMEASURE(), DATESINPERIOD(dim_date[full_date], MAX(dim_date[full_date]), -12, MONTH))`
- **Testado? Sim. Retestar:** matriz `dim_date[year]` (linhas) × `Time Analytics` (colunas) × `[Total Revenue]` → 2022 $844M · 2023 $1.076M (PY=844, **YoY% +27,4%**) · 2024 $1.137M (**+5,7%**). Se PY vier em branco ao isolar 1 ano, é o gotcha do SAMEPERIODLASTYEAR — por isso a medida avulsa de YoY usa o padrão `MAX(year)`.

### 2. Field Parameters
- **Onde uso:** Página 2 (Demand) — explorador self-service.
- **Como desenvolvi:** PBI Desktop → Modelagem → Novo parâmetro → Campos. `FP_Metric` (5 medidas) e `FP_Dimension` (5 colunas).
- **Como funciona:** cada parâmetro é uma tabela calculada trocada por slicer; o mesmo gráfico re-renderiza. 5×5 = **25 análises em 1 visual**.
- **Testado? Sim. Retestar:** `FP_Metric` no Y, `FP_Dimension` no X + 2 slicers; selecione Total Trips × Day Part → Afternoon ~40M · Evening ~40M · Morning ~25M · Late Night ~10M (**Afternoon+Evening ≈ 70%**). Troque os slicers e veja mudar sem quebrar.

### 3. Calculation Groups
- **Onde uso:** é o **motor do #1** — mesmo objeto "Time Analytics".
- **Como desenvolvi:** Tabular Editor 2 (o PBI Desktop não tem UI boa pra isso). 7 calculation items com ordinal.
- **Como funciona:** em vez de 10 medidas × 7 variações = 70 medidas, mantenho **10 medidas + 7 items = 17 objetos**. Ganho: manutenção e consistência.
- **Gotcha (bom citar):** o calc group liga "desencorajar medidas implícitas" → toda agregação implícita precisa virar medida explícita.
- **Testado? Sim** (mesma matriz do #1). **Retestar:** troque a medida base (ex.: Total Trips no lugar de Revenue) na matriz e confirme que os 7 cálculos propagam sem reescrever nada.

### 4. Conditional Formatting
- **Onde uso:** P1 (cor dinâmica nos cards Revenue/Anomaly), P2 (heat map), P5 (semáforo de anomalia), P6 (season matrix).
- **Como desenvolvi:** painel Formatar → Formatação condicional. Cor dinâmica = "Formatar por: valor do campo" apontando pra uma medida DAX que devolve hex (ex.: `Anomaly Color`). Heat map = Matrix nativa com escala de fundo (`#EAF1FB`→`#1F3864`), sem custom visual.
- **Como funciona:** a medida devolve a cor por threshold de negócio (verde/amarelo/vermelho).
- **Gotchas (bom citar):** (a) flag do lakehouse é INT → `is_anomalous = 1`, nunca `= TRUE()` (DAX não coage INT↔BOOL); (b) `FORMAT(..., "...", "en-US")` com locale forçado, senão o ponto vira separador de milhar em PT-BR.
- **Testado? Sim. Retestar:** mude o filtro de ano e veja o card de Revenue trocar de cor conforme o YoY; passe o mouse no heatmap e confira valor a valor.

### 5. Row-Level Security (RLS)
- **Onde uso:** Página 5 (Governance).
- **Como desenvolvi:** **6 roles** no semantic model do Fabric — 5 borough managers (`dim_zone[borough] = "Manhattan"`…) + 1 Global Manager (sem filtro).
- **Como funciona / gotcha-chave:** o modelo é **composite** (DirectQuery sobre o semantic model remoto), e o "Exibir como" do PBI Desktop **não enxerga roles remotas**. Por isso demonstro o efeito com medidas "Manager View": `Manhattan Manager View = CALCULATE([Total Revenue], dim_zone[borough]="Manhattan")`. A soma das 5 ≈ global (gap = zonas fora de NYC).
- **Testado? Sim. Retestar:** no Fabric/Service use "Testar como função" em cada role; ou no relatório confirme que as medidas Manager View somam ~$3,06 Bi. Produção: ligar as roles a grupos do Azure AD.

### 6. UDFs (DAX User-Defined Functions)
- **Onde uso:** Página 3 (Operations) — `duration_class` no histograma e `ComputeRevenuePerMile` no KPI de custo/milha.
- **Como desenvolvi:** via **DAX Query View** (o ribbon de Modelagem não tem UI de UDF), sintaxe `DEFINE FUNCTION nome = (params) => expressão`, salvo com "Atualizar o modelo com alterações". 3 funções: `ClassifyTripDuration`, `ClassifyFareTier`, `ComputeRevenuePerMile` (DIVIDE com guard de zero).
- **Como funciona:** encapsulo lógica reutilizável — troco em 1 lugar, propaga. Obs.: `duration_class` foi **materializado no Gold** (espelha a UDF) por performance, em vez de classificar 116M linhas em DAX ao vivo.
- **Gotchas (ouro pra banca técnica):** (a) sem aspas no DEFINE; (b) sem type hint (`(minutes)`, não `(minutes : NUMERIC)`); (c) é preview feature → exige **reiniciar** o PBI Desktop; (d) `BLANK < 5` é TRUE (BLANK coage a 0) → guardar com `IF(ISBLANK(x), …)`.
- **Testado? Sim. Retestar:** tabela `borough` × tiers → EWR Rev/Mile **$24,05** (≈5× Manhattan $5,67) · Queens $4,87 · Bronx $1,26.

### Frase-resumo pro slide 8 (credibilidade técnica)
> "Cobri os 6 features e validei cada um — não só liguei a opção, testei o resultado: o calc group contra uma matriz year × measure, os field parameters trocando 25 combinações, o RLS com medidas Manager View que somam o global, e as UDFs contra uma tabela de tiers por borough. Tudo bate com o business sanity query que rodei no Gold antes do Power BI."

### Os 2 pra ir fundo se a banca deixar escolher
**Calculation Groups** (mostra maturidade de modelagem: 70 perspectivas com 17 objetos) e **UDFs** (feature nova de 2025 + os 4 gotchas que você resolveu) — são os que mais impressionam time técnico. Conditional Formatting e Field Parameters são os mais fáceis de demonstrar ao vivo.

---

## 8. RLS — como demonstrar certo + troubleshooting (vivido na prática)

### ⚠️ "Test as role" NÃO funciona aqui (confirmado)
O modelo é composite/DirectQuery com **SSO** → o Service retorna *"Testar como função não funciona com o logon único (SSO)"*. É limitação da arquitetura, não erro. Use um dos 2 caminhos abaixo.

### Caminho 1 — demo segura (RECOMENDADO): medidas "Manager View"
Na tela de Governança, as medidas `Manhattan Manager View = CALCULATE([Total Revenue], dim_zone[borough]="Manhattan")` (uma por borough) mostram a receita que cada gestor veria. Rodam como você (dono) → **sempre funcionam**, sem SSO/role/login. Mostra Manhattan vs Global lado a lado.

### Caminho 2 — prova REAL com login (o "uau", faça e grave antes)
1. Workspace **tlc-3m-techcase** → **Gerenciar acesso**.
2. Adiciona `nicolasRLS` como **Viewer/Visualizador** — NUNCA Member/Admin (esses ignoram RLS; Viewer respeita).
3. Mantém ele na role **Manhattan Manager**.
4. Abre o relatório logado como `nicolasRLS` → mostra **só Manhattan**.
5. **Screenshot / grava 20s** e mostra na banca.
(Isso resolve o erro anterior: faltava o `nicolasRLS` ter acesso ao Lakehouse, que o papel Viewer concede.)

**Plano B sempre:** screenshots Manhattan vs Global num slide — risco zero.

### Troubleshooting que eu realmente vivi (ótima resposta de banca)
Tentei testar com uma conta Azure AD separada (`nicolasRLS`), adicionada à role Manhattan + Build no modelo. Dois erros, em ordem:

1. **"Uma conexão não pode ser feita à fonte de dados sm_yellow_taxi_3m"** → o relatório é **composite** (DirectQuery pro modelo remoto); quem consome precisa de **Build no modelo de origem**, não só o relatório compartilhado.
2. Depois do Build, ainda falhou (só o card "RAW ROWS" carregava) → causa real: o modelo é **DirectLake**, que por padrão lê o OneLake com a **identidade do usuário (SSO)**. O `nicolasRLS` tinha Build no modelo mas **não tinha acesso ao Lakehouse `lh_yellow_taxi`** → a leitura DirectLake falha.

**Como resolver:**
- **Rápido (teste):** dar ao usuário acesso ao Lakehouse (Viewer no workspace / acesso de dados no OneLake).
- **Produção (o certo):** trocar a conexão do modelo de SSO para **identidade fixa** (Service Principal / Workspace Identity). Aí o consumidor só precisa do modelo + a role de RLS; não toca no Lakehouse, e o RLS vira o único guardião.

**Fala de banca:** *"RLS num modelo DirectLake: descobri que Build no modelo não basta — DirectLake com SSO exige acesso ao OneLake. A solução de produção é identidade fixa na conexão, aí o RLS do modelo é o único controle. Pra demonstrar, uso Test as role."*

### ✅ SOLUÇÃO QUE FUNCIONOU (login real com RLS) — receita confirmada
O que destravou o RLS ao vivo com uma conta separada:
1. **Identidade fixa** na conexão do `sm_yellow_taxi_3m`: Configurações → Conexões de nuvem → fonte OneLake → **Criar uma conexão** com **OAuth 2.0 (minha conta)** → mapear a fonte pra ela (em vez de "logon único/SSO").
2. **Acesso ao relatório** pro usuário: Viewer no workspace **ou** Compartilhar só o relatório (ambos respeitam RLS).
3. Usuário na role **Manhattan Manager**.
→ Resultado: `nicolasRLS` abre o relatório e vê **só Manhattan**.

**Por que funcionou:** com SSO, o DirectLake tentava ler o OneLake com a conta do usuário (que não tinha acesso ao Lakehouse) → erro. Com **identidade fixa**, o modelo lê sempre com a minha conta; o usuário só precisa do relatório + a role. **Member/Admin/Contributor furam o RLS; Viewer e "compartilhado" respeitam.**

**Fala de banca (decore):** *"Configurei RLS em 6 roles num modelo DirectLake. Descobri na prática que DirectLake com SSO exige acesso do usuário ao OneLake — então troquei a conexão pra identidade fixa (OAuth). Aí o RLS do modelo vira o único controle: compartilho o relatório, ponho o gestor na role, e ele vê só o borough dele. Testei com uma conta real e funcionou."*
