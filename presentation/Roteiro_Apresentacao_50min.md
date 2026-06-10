# 🎬 Roteiro da Apresentação — 3M TechCase (~50 min)

> **Seu mapa passo a passo.** Em cada ponto: o que você **FAZ** (slide ou tela) e o que você **FALA** (a essência). As falas completas estão no `Kit_de_Dominio_Apresentacao_3M.md`.
> **Slides em inglês, você fala em português.** Tempo total ~50 min — núcleo de ~30 min + Q&A.
> ⚡ **ATUALIZADO (P6):** agora são **6 telas**. A **Página 6 — Weather (NOAA)** entra como a **2ª fonte externa**, junto com a Página 4 (ACS Demographics).

---

## ⚙️ Antes de começar (checklist, 5 min antes da call)
- [ ] Power BI aberto no arquivo certo, **já logado no Fabric**.
- [ ] Internet ok; o `.pbix` abre e os visuais carregam (sem erro de conexão).
- [ ] Deck aberto em modo apresentação, na ordem certa.
- [ ] **Screenshots das 6 telas salvas** (plano B se travar) — principalmente das 2 que vão ao vivo.
- [ ] `.pbix` já enviado pra 3M (o briefing pede antes da apresentação).
- [ ] Água por perto, notificações desligadas.

## 📌 O que é "demo ao vivo" (pra não ter dúvida)
Demo ao vivo = em vez de mostrar uma **imagem** do dashboard no slide, você **abre o Power BI de verdade** e interage: clica num filtro e os números mudam na frente da banca. Prova que o negócio é real.
**Recomendação:** ao vivo só as **2 telas mais fortes** (Executive Overview + Demand). As outras 4 (Operations, Demographics, **Weather**, Governance) você mostra como **imagem no slide**, resumindo o insight. Controla tempo e risco de travar sem perder impacto.

## 🌐 A jogada das DUAS fontes externas (decore isto)
O briefing pede **uma** fonte externa que correlacione. Você trouxe **duas**, cada uma com tela própria e padrão de ingestão diferente:

- **Página 4 — ACS Demographics** (renda, população, densidade) → fonte externa **#1**, *master data* via **snapshot** versionado.
- **Página 6 — NOAA Weather** (clima, risco operacional) → fonte externa **#2**, *dado operacional* via **REST API** com paginação anual.

Fale isso explicitamente no slide de Enrichment: *"O briefing pedia uma fonte externa; eu trouxe duas, e tratei cada uma conforme sua volatilidade — demografia é master data, clima é operacional."* Isso ataca direto o critério de avaliação.

---

## PARTE 1 — O CASE (~30 min)

### `0:00–0:03` · Abertura — **Slide 1 (Título)**
- **FAZ:** slide de título no ar. Se apresenta em 2 frases (nome + "mais de 4 anos em dados, BI e engenharia analítica").
- **FALA:** *"O briefing pedia um relatório de Power BI. Eu entendi como um pedido pra mostrar como eu trabalho — então tratei os táxis de Nova York como uma cadeia de suprimentos urbana."*
- **TRANSIÇÃO:** "Deixa eu começar pelo porquê dessa leitura."

### `0:03–0:06` · Framing: táxi = supply chain — **Slide 2**
- **FAZ:** slide do framing.
- **FALA:** a tradução — viagem = remessa, duração = **lead time**, zona = nó da rede, tarifa = cost-to-serve. Reforça: como o briefing **não deu um problema**, esse foi **o propósito que eu propus**.
- **TRANSIÇÃO:** "Com essa lente, vamos ao caminho do dado."

### `0:06–0:11` · Pipeline — **Slides 3 a 6**
- **Slide 3 (Approach):** os 5 blocos — Intake, Harmonização, Enriquecimento, Modelo, Front-end.
- **Slide 4 (Intake):** 119,1M de linhas; **schema drift** (VendorID INT/BIGINT, `airport_fee`/`Airport_fee`) resolvido com schema canônico.
- **Slide 5 (Harmonização):** **97,16%** retido, cada descarte rastreável (distância ≤ 0, tarifa negativa).
- **Slide 6 (Enrichment):** ⭐ **AQUI você apresenta as DUAS fontes externas** — ACS (#1) + NOAA (#2) — com o padrão "API + snapshot".
- **TRANSIÇÃO:** "Dado limpo e enriquecido, parti pro modelo."

### `0:11–0:15` · Modelo + Features + Medidas — **Slides 7 a 9**
- **Slide 7 (Model):** star schema, 1 fato + 6 dimensões, via **DirectLake**. (Mencione: `dim_date` enriquecida com clima NOAA; `dim_zone` com ACS.)
- **Slide 8 (6 features):** passa rápido confirmando que cobriu **os 6 exigidos** + 3 UDFs suas.
- **Slide 9 (Measure layer):** calc group = 10 medidas × 7 items = **70 perspectivas** com manutenção mínima.
- **TRANSIÇÃO:** "Chega de slide — deixa eu mostrar funcionando." → **abre o Power BI**.

### `0:15–0:29` · 🔴 DEMO + telas — **Slides 10 a 13**
- **Slide 10 → Power BI · Executive Overview (AO VIVO, ~5 min):**
  - Lê os 5 KPIs ($3,06 Bi · 116M · +5,7% · **17,56 min** lead time · 0,16%).
  - **Gira o filtro de ano** (mostra que reage ao vivo → Time Intelligence / Calc Group).
  - Aponta o **ABC de Manhattan** (~75%) e o **Top 10 O-D** (JFK + LaGuardia).
- **Slide 11 → Power BI · Demand (AO VIVO, ~4 min):**
  - **Heat map** hora×dia: pico **quinta 18h**, e os **2 regimes** (fim de tarde útil + madrugada de fim de semana).
  - **Troca métrica × dimensão no Field Parameter** ao vivo. Cita: 31% da demanda em 4h/dia.
- **Operations (imagem, ~1,5 min):** 78% das viagens em 5–30 min (cadeia madura); composição do custo 69/19/12; **UDFs** (`ClassifyTripDuration`, `ComputeRevenuePerMile`).
- **Slide 12 → Demographics — fonte externa #1 / ACS (imagem, ~2 min):**
  - Manhattan **+55pp** over-served / Brooklyn **−29pp** sub-servido; correlação demanda×renda = **0,43**.
- **🌦️ Slide 13 → Weather — fonte externa #2 / NOAA (imagem, ~2,5 min):** *(NOVO)*
  - **Sazonalidade:** **Fall = estação de estresse** — 2º maior volume (**109 mil/dia**) com o **pior lead time (18,46 min, +1,71 vs inverno)** → alto volume × ciclo lento = cost-to-serve máximo. **Spring** é o pico real (**113 mil/dia**) mas eficiente (17,44 min). **Winter** = baixa demanda + ciclo mais rápido (16,75 min) = janela de manutenção/capacidade.
  - **Extremos (contraintuitivo):** em dias de clima extremo (só **4 em 1.096**), a demanda **cai 36%** (106k→68k), mas o lead time fica **mais rápido (−3,8 min)** e a tarifa cai → **auto-seleção**: só as viagens essenciais acontecem, e com menos trânsito o ciclo encurta.
  - **Tradução SC:** sazonalidade = *peak-season surcharge / planejamento de capacidade*; choque extremo = *disrupção que reduz volume mas muda o perfil operacional*.
- **Governance + RLS (imagem, ~1,5 min):** **97,16% retido** + **RLS, 6 roles**: cada gestor vê só seu borough, o global vê os $3,06 Bi.
- **⚠️ SE O DEMO TRAVAR:** volta pro slide com o screenshot e segue contando. O conteúdo é o mesmo.
- **TRANSIÇÃO:** "Juntando tudo, os insights que mais importam."

### `0:29–0:33` · Insights + Next steps + Fecho — **Slides 14 a 15**
- **Slide 14 (Key insights):** os de ouro — lead time estável <1%, ABC (Manhattan+Queens 93%), over/under-served, 2 perfis de anomalia, e **🌦️ Seasonal & weather stress (Fall + extremos → planejamento de capacidade e disrupção)**.
- **Slide 15 (Next steps + close):** próximos passos (forecast de demanda **weather-aware**, escalar na 3M com Fabric Pipelines + Azure DevOps + Purview). Fecho → *"O que vocês viram é a ponta do iceberg; a fundação está toda no repositório, versionada desde o dia 1."*

---

## 🎯 Mapa das 6 features — onde cada uma aparece
| # | Feature | Onde | O que você faz / fala |
|---|---|---|---|
| — | **as 6 de uma vez** | Slide 8 | "Os 6 exigidos, todos entregues." Rede de segurança. |
| 1 | Time Intelligence | Slide 10 · Exec (ao vivo) | Gira o filtro de ano → +5,7% YoY reage. |
| 2 | Calculation Groups | Slide 9 + Exec | "Motor: 1 calc group × 7 itens = 70 perspectivas." |
| 3 | Field Parameters | Slide 11 · Demand (ao vivo) | Troca métrica × dimensão. "1 visual = 25 análises." |
| 4 | Conditional Formatting | Heat map (Demand) + cards Exec + **season matrix (Weather)** | "Cor = regra de negócio, não enfeite." |
| 5 | Security roles (RLS) | Governance (imagem) | "6 roles: cada gestor vê só seu borough." |
| 6 | UDFs | Operations (imagem) | "`ClassifyTripDuration` + `ComputeRevenuePerMile` ao vivo." |

**Ao vivo você só clica em 4** (features 1–4, nas telas Exec + Demand). RLS e UDFs você **narra sobre a imagem**. O slide 8 garante que nenhuma passou batida.

---

## PARTE 2 (opcional) — só se convidarem · ~3 min
Se perguntarem *"tem algum projeto seu pra mostrar?"*, puxa o **HAND** (`Parte2_Pitch_HAND.md` + diagrama). Não gaste o tempo do case com isso — carta na manga.

## Q&A — ~15–18 min
É aqui que a banca te avalia de verdade. Respostas prontas no Kit (seção 4). Se travar: *"boa pergunta — minha decisão foi X por causa de Y."* **Não invente número**: se não souber, diga como descobriria. Admita as limitações você mesmo (correlação n=5, ACS por borough, weather por dia) — passa maturidade.

---

## ⏱️ Resumo do tempo (guia, não cronômetro)
| Bloco | Tempo | Slides |
|---|---|---|
| Abertura | 3 min | 1 |
| Framing (táxi = supply chain) | 3 min | 2 |
| Pipeline (com 2 fontes externas) | 5 min | 3–6 |
| Modelo + features + medidas | 4 min | 7–9 |
| **Demo ao vivo (Exec + Demand) + 4 telas em imagem** | **14 min** | 10–13 |
| Insights + fecho | 4 min | 14–15 |
| **Q&A** | **~15–18 min** | — |
| *(Parte 2, só se pedirem)* | *+3 min* | — |

**Núcleo ~33 min + Q&A.** Sobra folga proposital: melhor terminar com tempo do que correr. Se a banca te cortar pra perguntar, ótimo — significa que engajaram.
