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

## Página 2 — Demand Deep Dive (a construir)

**Objetivo**: explorar **quando e onde** a demanda acontece. Para analistas de capacity planning.

Wireframe será adicionado na Etapa E (após Página 1 finalizada).

---

## Página 3 — Operations & Cost (a construir)

**Objetivo**: lead time, eficiência operacional, breakdown de tarifa.

Wireframe será adicionado no dia 4.

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
