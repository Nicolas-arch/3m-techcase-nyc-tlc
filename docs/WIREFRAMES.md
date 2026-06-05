# Wireframes do Dashboard Power BI

> Layout textual de cada página do relatório, com posicionamento de visuais, dimensões esperadas e DAX/campos atribuídos. Documento serve como blueprint de construção e referência durante a apresentação.

## Convenções

- **Tamanho da página**: 16:9 padrão Power BI Desktop (1280×720 px).
- **Grid**: 12 colunas, ~107px cada.
- **Tema**: aplicado via `powerbi/theme.json` (azul corporativo).
- **Tipografia**: Segoe UI (default do tema).
- **Cores principais**: `#1F3864` (escuro), `#2E75B6` (médio), `#22A06B` (positivo), `#D32F2F` (negativo), `#F29111` (alerta).

---

## Página 1 — Executive Overview ✅ FINALIZADA (dia 4)

**Status**: tela completa, polishes aplicados, todos os subtítulos dinâmicos em inglês. Canvas final: **1600×900px**.

**Objetivo**: telinha-resposta para o C-level. Em **5 segundos**, o leitor sabe: volume, receita, tendência, geografia, top lanes.

**Layout**:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ HEADER (altura 80px)                                                     │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│  Título: "NYC TLC Yellow Taxi — Executive Overview"                      │
│  Subtítulo: "2022-2024 — Pipeline end-to-end via Fabric + Power BI"      │
│  Slicers: [Year ▼]  [Month ▼]  [Time Analytics ▼]                        │
└──────────────────────────────────────────────────────────────────────────┘

┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ KPI Card 1  │ KPI Card 2  │ KPI Card 3  │ KPI Card 4  │ KPI Card 5  │
│             │             │             │             │             │
│ Total       │ Total Trips │ Avg Fare    │ YoY %       │ Anomaly Rate│
│ Revenue     │             │             │             │             │
│             │             │             │             │             │
│ $3,06B      │ 115,7M      │ $26,42      │ +5,7%       │ 0,16%       │
│ (verde)     │             │             │ (verde)     │ (verde)     │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
   altura ~130px

┌────────────────────────────────────────┬─────────────────────────────────┐
│                                        │                                 │
│  VISUAL 1: TRENDLINE                   │  VISUAL 2: REVENUE BY BOROUGH   │
│  Title: "Revenue Trend (Monthly)"      │  Title: "Revenue by Borough"    │
│                                        │                                 │
│  Tipo: Line Chart                      │  Tipo: Bar Chart (horizontal)   │
│  X: dim_date[year_month]               │  Y: dim_zone[borough]            │
│  Y: [Total Revenue]                    │  X: [Total Revenue]              │
│  Cor: #1F3864 (azul escuro)            │  Cor: gradient azul              │
│                                        │  Sort: descendente por valor     │
│                                        │  Data labels: visíveis            │
│  altura ~360px  largura 60%            │  altura ~360px  largura 40%      │
│                                        │                                 │
└────────────────────────────────────────┴─────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ VISUAL 3: TOP 10 O-D LANES                                               │
│ Title: "Top 10 Origin-Destination Lanes by Revenue"                      │
│                                                                          │
│ Tipo: Tabela com Data Bars                                               │
│ Colunas:                                                                 │
│   - Pickup Zone (dim_zone[zone_name])                                    │
│   - Dropoff Zone (dim_zone_dropoff[dropoff_zone_name])                   │
│   - Total Trips                                                          │
│   - Total Revenue (com data bars)                                        │
│   - Avg Fare                                                             │
│ Filtro: Top 10 by Total Revenue                                          │
│                                                                          │
│ altura ~200px                                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

### Composição visual da Página 1

| Visual | Posição (grid) | Tamanho aprox. | Campos |
|---|---|---|---|
| Header (texto) | Linha 1, 12 colunas | 1280×80 | Texto estático + 3 slicers |
| KPI 1 — Total Revenue | Linha 2, col 1-2 | 250×130 | `[Total Revenue]` + `YoY Color` |
| KPI 2 — Total Trips | Linha 2, col 3-4 | 250×130 | `[Total Trips]` |
| KPI 3 — Avg Fare | Linha 2, col 5-6 | 250×130 | `[Avg Fare]` |
| KPI 4 — YoY % | Linha 2, col 7-8 | 250×130 | `[Total Revenue]` × `'Time Analytics'[Time Analytics] = "YoY %"` |
| KPI 5 — Anomaly Rate | Linha 2, col 9-10 | 250×130 | `[Anomaly Rate %]` |
| Trendline | Linha 3, col 1-7 | 750×360 | X=`year_month`, Y=`[Total Revenue]` |
| Revenue by Borough | Linha 3, col 8-12 | 530×360 | Y=`borough`, X=`[Total Revenue]` |
| Top 10 O-D Lanes | Linha 4, col 1-12 | 1280×200 | Tabela com data bars |

### Storytelling da Página 1 (durante a demo)

"Esta tela responde 4 perguntas em 10 segundos:

1. **Quanto vale o negócio?** $3,06B em 3 anos.
2. **Está crescendo?** YoY de +5,7% em 2024 — recovery pós-pandemia confirmado pela trendline.
3. **Onde está concentrado?** Manhattan domina, Queens segundo (puxado por aeroportos).
4. **Quais são as lanes críticas?** JFK e LaGuardia aparecem no top 10 com data bars proeminentes.

Em supply chain, equivale ao painel executivo onde CFO + COO + VP Operations conseguem fechar o status do trimestre em 1 olhada."

### Conditional Formatting aplicado nesta página

- **KPI Card 1 (Total Revenue)**: cor do valor via `YoY Color` medida (verde se YoY > 0, vermelho se < 0).
- **KPI Card 4 (YoY %)**: cor verde fixa para destacar crescimento.
- **KPI Card 5 (Anomaly Rate %)**: cor verde fixa (0,16% é healthy).
- **Revenue by Borough**: data labels habilitados + barras em gradient azul.
- **Top 10 O-D Lanes**: data bars em `[Total Revenue]`, intensity proporcional ao valor.

### Insights esperados visíveis nesta página

1. Manhattan + Queens = ~96% do volume (visual borough bar chart).
2. Tendência claramente positiva de 2022 → 2024.
3. JFK ⇄ Manhattan vai aparecer no top de O-D lanes — corredor premium clássico.
4. YoY positivo destacado em verde = mensagem de recovery.

### Estado final aplicado no dia 4 (mudanças vs versão inicial)

| Item | Versão inicial (dia 3) | Versão final (dia 4) |
|---|---|---|
| KPI 4 | YoY Growth (5,7%) — redundante | **AVG LEAD TIME (min)** = 17,56 — alinha narrativa supply chain |
| Anomaly card | Valor preto, subtítulo estático | Cor dinâmica via `Anomaly Color` + subtítulo via `Anomaly Subtitle` |
| Tabela Top 10 | 175px altura, mostrava N/A → N/A | 270px altura, filtro `borough <> "N/A"` em ambas as pontas |
| Subtítulos | Mistura PT/EN, alguns estáticos | Todos dinâmicos em inglês, FORMAT com locale `"en-US"` |
| Hierarquia cromática | 4 cards coloridos | 2 sinalizadores (Revenue + Anomaly) + 3 neutros |
| Idioma | Mistura PT/EN | 100% inglês (apresentação global 3M) |

### Medidas DAX usadas (referência rápida)

| Card | Valor (Data) | Cor (fx) | Subtítulo (Reference label fx) |
|---|---|---|---|
| Total Revenue | `[Total Revenue]` | `[YoY Color Revenue]` | `[Revenue Subtitle]` |
| Total Trips | `[Total Trips]` | — (neutro) | `[Trips Subtitle]` |
| Avg Fare per Trip | `[Avg Fare]` | — (neutro) | `[Fare Subtitle]` |
| Avg Lead Time (min) | `[Avg Lead Time (min)]` | — (neutro) | `[Lead Time Subtitle]` |
| Anomaly Rate | `[Anomaly Rate]` | `[Anomaly Color]` | `[Anomaly Subtitle]` |

Definição completa de cada medida está em `docs/CODE_EXPLAINED.md` seção 9.

---

## Página 2 — Demand Deep Dive 🟡 EM CONSTRUÇÃO (dia 8)

**Status**: wireframe + medidas definidos. Canvas: **1920×1080px** (novo padrão das páginas 2-5).

**Objetivo**: responder *quando e onde* a demanda se concentra. Público: analista de capacity planning / S&OP.
**Tradução supply chain**: heat map = perfil de demanda por janela horária (dimensionamento de turno/capacidade); sazonalidade = planejamento anual; top zonas = nós de maior throughput; Field Parameter = self-service de fatiamento.

### Faixas do canvas (Y)
- Header `0–88` · KPIs `104–224` · Linha principal `240–800` · Base `816–1056` · Margens 24px.

### Mapa de posições (X, Y, W, H)

| # | Componente | Tipo | X | Y | W | H |
|---|---|---|---|---|---|---|
| Header | Faixa navy + título + subtítulo | Texto/forma | 0 | 0 | 1920 | 88 |
| S1 | Slicer Year | Slicer dropdown | 1422 | 24 | 150 | 40 |
| S2 | Slicer Month | Slicer dropdown | 1584 | 24 | 150 | 40 |
| S3 | Slicer Borough | Slicer dropdown | 1746 | 24 | 150 | 40 |
| K1 | Total Trips | Card | 24 | 104 | 360 | 120 |
| K2 | Avg Daily Trips | Card | 402 | 104 | 360 | 120 |
| K3 | Peak Hour | Card | 780 | 104 | 360 | 120 |
| K4 | Peak Day | Card | 1158 | 104 | 360 | 120 |
| K5 | Rush Hour Share | Card | 1536 | 104 | 360 | 120 |
| V1 | **Heat map hora×dia (HERO)** | Matrix + color scale | 24 | 240 | 1130 | 560 |
| V2 | Sazonalidade mensal | Line chart | 1170 | 240 | 726 | 272 |
| V3 | Top 10 zonas de demanda | Bar horizontal | 1170 | 528 | 726 | 272 |
| V4 | Explorador de demanda | Column (Field Param) | 24 | 816 | 1430 | 240 |
| S4 | Slicer FP Metric | Slicer (FP_Metric) | 1470 | 816 | 426 | 112 |
| S5 | Slicer FP Dimension | Slicer (FP_Dimension) | 1470 | 944 | 426 | 112 |

### Dados por visual
- **V1 Heat map**: Linhas `dim_time[hour_label]` (sort by `hour`) · Colunas `dim_date[day_name]` (sort by `day_order_mon`) · Valores `[Total Trips]` · Conditional formatting Background gradiente `#EAF1FB`→`#1F3864`. (cobre feature **Conditional Formatting**)
- **V2 Sazonalidade**: X `dim_date[month_name]` (sort by `month`) · Y `[Total Trips]` · Legenda `dim_date[year]` (2022 `#9DC3E6`, 2023 `#2E75B6`, 2024 `#1F3864`).
- **V3 Top zonas**: Y `dim_zone[zone_name]` (Pickup) · X `[Total Trips]` · Top N 10 · filtro `borough <> "Unknown"` · cor `#2E75B6`.
- **V4 Explorador**: Eixo `FP_Dimension` · Valor `FP_Metric` · slicers S4/S5 trocam ambos → 1 visual = 25 análises. (cobre feature **Field Parameters**)

### Medidas DAX criadas no dia 8 (Página 2)
```dax
Peak Hour =
VAR _byHour = ADDCOLUMNS ( VALUES ( dim_time[hour_label] ), "@t", [Total Trips] )
RETURN MAXX ( TOPN ( 1, _byHour, [@t], DESC ), dim_time[hour_label] )

Peak Day =
VAR _byDay = ADDCOLUMNS ( VALUES ( dim_date[day_name] ), "@t", [Total Trips] )
RETURN MAXX ( TOPN ( 1, _byDay, [@t], DESC ), dim_date[day_name] )

Avg Daily Trips = AVERAGEX ( VALUES ( dim_date[full_date] ), [Total Trips] )

Rush Hour Trip % = DIVIDE ( CALCULATE ( [Total Trips], dim_time[is_rush_hour] = 1 ), [Total Trips] )

Weekend Trip % = DIVIDE ( CALCULATE ( [Total Trips], dim_date[is_weekend] = 1 ), [Total Trips] )
```
Coluna calculada em `dim_date` (ordena heat map Seg→Dom; modelo tem `day_of_week` 1=domingo):
```dax
day_order_mon = IF ( dim_date[day_of_week] = 1, 7, dim_date[day_of_week] - 1 )
```

### Subtítulos dinâmicos dos 5 cards (texto, locale en-US)
- K1 Total Trips → reaproveita `[Trips Subtitle]` (Pág. 1). Os 4 abaixo são novos:
```dax
Avg Daily Trips Subtitle =
VAR _maxDay = MAXX ( VALUES ( dim_date[full_date] ), [Total Trips] )
RETURN "peak day " & FORMAT ( _maxDay, "#,##0", "en-US" )

Peak Day Subtitle =
VAR _byDay = ADDCOLUMNS ( VALUES ( dim_date[day_name] ), "@t", [Total Trips] )
VAR _hi = MAXX ( _byDay, [@t] )
VAR _lo = MINX ( _byDay, [@t] )
RETURN FORMAT ( DIVIDE ( _hi - _lo, _lo ), "+0.0%;-0.0%", "en-US" ) & " vs quietest day"

Peak Hour Subtitle =
VAR _byHour = ADDCOLUMNS ( VALUES ( dim_time[hour_label] ), "@t", [Total Trips] )
VAR _share  = DIVIDE ( MAXX ( TOPN ( 1, _byHour, [@t], DESC ), [@t] ), [Total Trips] )
RETURN FORMAT ( _share, "0.0%", "en-US" ) & " of all trips"

Rush Hour Subtitle =
VAR _rushM = DIVIDE ( CALCULATE ( [Total Trips], dim_time[is_rush_hour] = 1 ), 1000000 )
RETURN FORMAT ( _rushM, "#,##0.0", "en-US" ) & "M in rush windows"
```
> Gotcha: o truque de escala `,,` no FORMAT é frágil — para "M"/"K" divida explícito (`/ 1000000`) e formate o resultado. KPIs desta página NÃO têm cor condicional (demanda é neutra), só subtítulo dinâmico.

### Storytelling (demo)
Olho cai no heat map → corredores escuros = picos (manhã útil + 18h). Direita: sazonalidade confirma estabilidade + crescimento YoY; top zonas dá o "onde". Base: explorador self-service por Field Parameters. Ação dirigida: **onde e quando alocar capacidade**.

### Gotchas reforçados
- Flags (`is_rush_hour`, `is_weekend`) comparadas com `= 1` (DAX não coage INT↔BOOL).
- `FORMAT` com locale `"en-US"` nos subtítulos.
- KPIs desta página são neutros (demanda não tem bom/ruim) — diferente da Página 1 (2 sinalizadores).

---

## Página 3 — Operations & Cost 🟡 EM CONSTRUÇÃO (dia 8)

**Status**: wireframe + medidas definidos. Canvas **1920×1080**.

**Objetivo**: lead time, eficiência operacional e cost-to-serve. Público: gerente de operações/custo.
**Pergunta-foco**: *quão eficiente e quão caro é cada "shipment", e onde está a ineficiência e o custo?*
**Tradução supply chain**: lead time = tempo de ciclo; distância×tarifa = custo proporcional ao serviço; composição da tarifa = cost-to-serve; surcharges = peak-season; revenue/mile = custo unitário.
**Cor semântica nova**: `#D32F2F` só na barra "Outlier" do histograma (duração outlier é ruim → sinalizar é correto).

### Mapa de posições (X, Y, W, H)

| # | Componente | Tipo | X | Y | W | H |
|---|---|---|---|---|---|---|
| Header | navy + título | — | 0 | 0 | 1920 | 88 |
| S1–S3 | Year / Month / Time View | Slicers | 1422 / 1584 / 1746 | 24 | 150 | 40 |
| K1 | Avg Lead Time (min) | Card | 24 | 104 | 360 | 120 |
| K2 | Avg Distance (mi) | Card | 402 | 104 | 360 | 120 |
| K3 | Avg Speed (mph) | Card | 780 | 104 | 360 | 120 |
| K4 | Avg Cost / Trip ($) | Card | 1158 | 104 | 360 | 120 |
| K5 | Cost / Mile ($) — UDF | Card | 1536 | 104 | 360 | 120 |
| V1 | **Distance × Fare (HERO)** | Scatter (bubble) | 24 | 240 | 1130 | 560 |
| V2 | Lead time distribution | Column (histograma) | 1170 | 240 | 726 | 272 |
| V3 | Fare composition | Bar horizontal (multi-medida) | 1170 | 528 | 726 | 272 |
| V4 | Cost-to-serve trend | Line + área (mensal) | 24 | 816 | 1872 | 240 |

### Dados por visual
- **V1 Scatter**: X `[Avg Distance (mi)]` · Y `[Avg Fare]` · Tamanho `[Total Trips]` · Detalhes `dim_zone[zone_name]` · Legenda `dim_zone[borough]`. ~260 pontos (1/zona) → performático. Aeroportos = outliers.
- **V2 Histograma**: X `fact_trips[duration_class]` (coluna nova Gold) · Y `[Total Trips]`. Cond. format: barra "Outlier" em `#D32F2F`. (espelha UDF `ClassifyTripDuration`)
- **V3 Composition**: Valores = `[Avg Base Fare]`, `[Avg Surcharges & Taxes]`, `[Avg Tip]`, `[Avg Tolls]`.
- **V4 Trend**: X `dim_date[year_month]` · Y `[Avg Fare]`. Choque tarifário 2022→2023. (Time Intelligence + Calc Group)

### Medidas DAX novas (Página 3)
```dax
Avg Distance (mi) = DIVIDE ( [Total Distance], [Total Trips] )
Avg Speed (mph)   = CALCULATE ( AVERAGE ( fact_trips[avg_speed_mph] ), fact_trips[is_anomalous] = 0 )
Avg Cost per Trip ($) = DIVIDE ( SUM ( fact_trips[total_amount] ), [Total Trips] )
Cost per Mile ($) = ComputeRevenuePerMile ( [Total Revenue], [Total Distance] )   -- UDF ao vivo

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
Subtítulos: K1 reaproveita `[Lead Time Subtitle]`; K2–K5 usam template YoY (en-US), trocando a medida base.

### Coluna nova no Gold — fact_trips (alinhar limites com o UDF ClassifyTripDuration)
```sql
CASE
  WHEN s.duration_min IS NULL THEN '0 · Unknown'
  WHEN s.duration_min < 5     THEN '1 · Very Short'
  WHEN s.duration_min < 15    THEN '2 · Short'
  WHEN s.duration_min < 30    THEN '3 · Medium'
  WHEN s.duration_min < 60    THEN '4 · Long'
  WHEN s.duration_min < 180   THEN '5 · Very Long'
  ELSE '6 · Outlier'
END AS duration_class   -- limites = UDF ClassifyTripDuration (5/15/30/60/180)
```
Prefixo `1 ·`… ordena o eixo sem coluna de sort. Re-roda a célula do fato → sincroniza `duration_class` no modelo semântico (igual ao `day_order_mon`).

### Features obrigatórias cobertas aqui
UDF (Cost/Mile ao vivo + duration_class espelhado) · Conditional Formatting (Outlier vermelho) · Time Intelligence (trend + Calc Group).

### Storytelling (demo)
Scatter = tarifa cresce com distância, aeroportos como outliers premium. Histograma = consistência do lead time (cauda Outlier mínima). Composition = base fare domina, surcharges pesam em aeroporto. Trend = choque de preço 2022→2023. Ação: onde atacar custo e onde o lead time arrisca o SLA.

---

### Build final — field wells + ajustes de leitura (Página 3)

**Slicers (header):** `dim_date[year]` · `dim_date[month]` · `'Time Analytics'[Time Analytics]`

**Cards** (Callout value + Reference label):

| Card | Valor | Subtítulo |
|---|---|---|
| K1 Avg Lead Time | `[Avg Lead Time (min)]` | `[Lead Time Subtitle]` |
| K2 Avg Distance | `[Avg Distance (mi)]` | `[Avg Distance Subtitle]` |
| K3 Avg Speed | `[Avg Speed (mph)]` | `[Avg Speed Subtitle]` |
| K4 Avg Cost / Trip | `[Avg Cost per Trip ($)]` | `[Avg Cost Subtitle]` |
| K5 Cost / Mile | `[Cost per Mile ($)]` | `[Cost per Mile Subtitle]` |

**V1 Scatter:** X `[Avg Distance (mi)]` · Y `[Avg Fare]` · Size `[Total Trips]` · Details `dim_zone[zone_name]` · Legend `dim_zone[borough]` · **Filtro `is_anomalous = 0` + eixo X capado (~30)** ← essencial pra leitura.
**V2 Histograma:** X `fact_trips[duration_class]` · Y `[Total Trips]` · cor `6 · Outlier` = `#D32F2F`.
**V3 Composição:** **Barra EMPILHADA** · Valores = `[Avg Base Fare]`, `[Avg Surcharges & Taxes]`, `[Avg Tip]`, `[Avg Tolls]` · categoria vazia · rótulos on.
**V4 Trend:** X `dim_date[year_month]` · Y `[Avg Fare]` (+opcional `[Avg Surcharges & Taxes]`).

**Títulos como pergunta:** "O custo é proporcional à distância?" · "Quanto dura uma viagem?" · "Para onde vai a tarifa?" · "Como o custo-to-serve está evoluindo?"

---

## Página 4 — Demand vs Demographics (a construir)

**Objetivo**: correlação entre demanda e fonte externa (ACS demographics).

Wireframe será adicionado no dia 4.

---

## Página 5 — Data Quality / Backstage (a construir)

**Objetivo**: transparência sobre tratamento de dados, anomalias detectadas, qualidade por borough.

Wireframe será adicionado no dia 4.

---

## Página 6 — Security & Governance (a construir)

**Objetivo**: demonstrar visualmente o efeito das RLS roles. Não está nos requisitos originais, mas adicionada para evidenciar funcionalidade.

Wireframe será adicionado no dia 4 com 6 KPI cards (uma por role).
