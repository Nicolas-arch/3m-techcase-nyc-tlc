# 🎒 Parte 2 (opcional) — Pitch de 3 min: Projeto HAND (frota / logística)

> **Formato:** ~3 min falados + 1 diagrama de arquitetura (`Parte2_Diagrama_HAND.svg`).
> **Confidencialidade:** você **não** mostra dado nem tela real do cliente. Reconta o problema, redesenha a arquitetura (isso é conhecimento seu) e cita os resultados — que já estão no seu CV/LinkedIn. Se o nome do cliente for sensível, diga *"um cliente do setor de transporte"*.
> **⚠️ Pontos a confirmar/ajustar** estão marcados com `[confirmar]` — alinhe antes do ensaio.

---

## Estrutura com tempo (decore a espinha, fale natural)
| Bloco | Tempo | Ideia central |
|---|---|---|
| 1. Contexto | ~20s | frota cara, zero visibilidade de onde o dinheiro vaza |
| 2. O que construí | ~40s | camada analítica que une fontes separadas; cruzar planejado × realizado |
| 3. Arquitetura | ~40s | aponta o diagrama: fontes → ETL → modelo → dashboards |
| 4. Resultados | ~40s | os 3 ganhos com cifra |
| 5. Por que importa pra 3M | ~30s | frota = logística = supply chain |
| 6. Fecho | ~10s | como eu gosto de trabalhar |

---

## Script pronto

**1. Contexto**
> "Como analista de BI numa operação de transporte, atendi um cliente que tinha uma frota cara de operar e quase nenhuma visibilidade sobre onde o dinheiro vazava — combustível, pneus e pedágio. A pergunta era direta: onde estão as ineficiências, e quanto dá pra economizar?"

**2. O que construí**
> "Montei uma camada analítica em Power BI que consolidou fontes operacionais que viviam separadas: a telemetria de rastreamento dos caminhões via API, os dados de abastecimento, os de pedágio e as rotas planejadas. O movimento-chave foi cruzar **planejado versus realizado** — porque é no desvio entre os dois que o custo se esconde."

**3. Arquitetura** *(aponta o diagrama)*
> "O fluxo é este: as fontes entram via API e arquivos, passam por um tratamento de limpeza e padronização, viram um modelo estrela em Power BI com os KPIs de custo por caminhão, por rota e por período, e desembocam em dashboards que os gestores usavam na reunião de resultado." `[confirmar: ETL via Power Query e/ou Python]`

**4. Resultados**
> "Três entregas concretas. Primeiro, identifiquei janelas e volumes de abastecimento mais vantajosos — isso deu cerca de **6% de economia de combustível por mês**. Segundo, mapeei aproximadamente **R$ 60 mil por mês em pedágio excedente**, cruzando o rastreamento com as rotas — e isso virou um **novo modelo de controle de pedágio** dentro da operação. Terceiro, detectei **desvios de rota** dos motoristas cruzando o GPS com a rota planejada, cortando exposição a custo não autorizado."

**5. Por que importa pra 3M**
> "Isso é supply chain na veia: frota é distribuição, combustível e pedágio são *freight cost*, e 'planejado versus realizado' é análise de desvio de plano — o mesmo raciocínio de OTIF e de *cost-to-serve*. Peguei dado operacional cru e transformei em decisão de eficiência com cifra. É exatamente o que um time de Supply Chain Analytics faz no dia a dia."

**6. Fecho**
> "Foi um projeto pequeno em time e grande em impacto — e resume como eu gosto de trabalhar: conectar dado, operação e dinheiro."

---

## Pontes pra supply chain (munição nas perguntas)
- Frota / transporte → **distribuição e logística**.
- Combustível + pedágio → **freight cost / cost-to-serve**.
- GPS × rota planejada → **desvio de plano (plan vs. actual)**, base de OTIF e controle de custo.
- Janela/volume de abastecimento → **timing de compra / strategic sourcing**.
- Reunião de resultado com gestores → **decisão executiva baseada em dado**.

---

## Q&A provável
- **"Como você pegava os dados de rastreamento?"** → "Via API REST da plataforma de telemetria `[confirmar qual]`, com coleta periódica; tratava e padronizava antes de modelar."
- **"Qual foi o maior desafio técnico?"** → "Unir fontes com granularidades diferentes — evento de GPS, abastecimento, pedágio — numa chave comum de caminhão/rota/data. `[confirmar]`"
- **"Como garantiu confiança no número da economia?"** → "Comparei baseline antes/depois e validei com o cliente nas reuniões de resultado. `[confirmar como mediu]`"
- **"Esse projeto não é confidencial?"** → "Os dados são do cliente, então não trago dados nem telas reais. O que apresento é a arquitetura e a abordagem, que são meu conhecimento. Os números de impacto são os que constam no meu currículo."

---

## Menção de apoio (solte numa pergunta da Parte 1)
> "Inclusive, na EuroChem eu já trabalhei com a **mesma stack de vocês** — Microsoft Fabric com OneLake e Modelo Semântico, integrando SAP e Oracle pra governança de dados financeiros, com análises de Curva ABC e Pareto. Então o que construí no case do táxi não é teoria — é o que eu já faço na prática num ambiente corporativo."

---

*⚠️ Antes do ensaio: confirmar os 3 `[confirmar]` (método de ETL, fonte de telemetria, como mediu a economia) pra você responder sem hesitar.*
