# 3M Technical Case — NYC TLC Yellow Taxi Analytics

[![Status](https://img.shields.io/badge/status-in_development-yellow)]()
[![Stack](https://img.shields.io/badge/stack-Microsoft_Fabric-blue)]()
[![Power BI](https://img.shields.io/badge/Power_BI-Desktop-F2C811)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

> Teste técnico para vaga de **Data / Analytics / BI** no time global de **Enterprise Supply Chain Analytics** da 3M.

## Visão geral

Solução end-to-end de engenharia analítica sobre os dados públicos do **NYC TLC Yellow Taxi (2022–2024)**, traduzindo a operação de táxis para uma narrativa de **cadeia de suprimentos urbana** (demanda, lead time, lanes O-D, eficiência operacional, custos e exception management).

A entrega cobre todo o fluxo:

1. **Data Intake** — ingestão programática de 36 arquivos Parquet (3 anos × 12 meses) da CDN pública da TLC.
2. **Harmonization & Cleansing** — limpeza, tipagem, derivação de campos e flags de qualidade em Spark SQL.
3. **Enrichment** — join com fonte externa: **NYC ACS Demographics** (renda, população, densidade por bairro) e **NOAA Weather** (clima histórico).
4. **Data Model** — star schema em Delta Lake (1 fato + 6 dimensões) materializado no Lakehouse.
5. **Front-end** — relatório Power BI conectado via **DirectLake** com Time Intelligence, Field Parameters, Calculation Groups, Conditional Formatting, RLS e UDFs DAX.

## Stack

| Camada | Ferramenta |
|---|---|
| Storage & Compute | Microsoft Fabric Lakehouse (OneLake + Delta) |
| Ingestão | Fabric Notebook (PySpark) |
| Transformação | Fabric Notebook (Spark SQL) |
| Modelo Semântico | Power BI semantic model em DirectLake |
| Front-end | Power BI Desktop |
| DAX Avançado | Tabular Editor 2 |
| Versionamento | GitHub |
| Plano B local | Python + Polars |

## Estrutura do repositório

```
3m-techcase-nyc-tlc/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md         # arquitetura técnica
│   ├── CONVENTIONS.md          # convenções de nomes e código
│   ├── DATA_SOURCES.md         # fontes de dados e como acessá-las
│   ├── DATA_DICTIONARY.md      # dicionário das tabelas Gold (será preenchido)
│   └── PRESENTATION_NOTES.md   # decisões técnicas + storytelling p/ apresentação
├── notebooks/                  # exportados do Fabric
│   ├── 01_bronze_ingest.ipynb     # ✅ ingestão + persistência Delta
│   ├── 02_silver_clean.ipynb      # ⏳ próximo
│   └── 03_gold_model.ipynb        # ⏳
├── sql/                        # scripts SQL auxiliares
├── powerbi/
│   ├── 3m_techcase.pbix
│   ├── theme.json
│   └── measures_reference.md
├── python_fallback/            # ingestão e ETL local (plano B)
├── images/                     # diagramas e screenshots
├── .env.example                # template das variáveis de ambiente
├── .gitignore
├── requirements.txt
└── LICENSE
```

## Pré-requisitos

- Microsoft 365 Developer Program tenant
- Microsoft Fabric Trial ativo
- Power BI Desktop ≥ 2.140 (com preview features: UDFs, Calculation Groups, DirectLake)
- Tabular Editor 2
- Python 3.10+

## Configuração de ambiente

```bash
git clone https://github.com/Nicolas-arch/3m-techcase-nyc-tlc.git
cd 3m-techcase-nyc-tlc

# venv para o plano B local
python -m venv .venv
.venv\Scripts\activate         # Windows
pip install -r requirements.txt

# variáveis de ambiente
copy .env.example .env         # preencher tokens
```

Tokens necessários (cadastro gratuito):

- **NOAA Climate Data Online**: https://www.ncdc.noaa.gov/cdo-web/token
- **NYC Open Data App Token**: https://data.cityofnewyork.us/profile/app_tokens

## Status do projeto

| Etapa | Status |
|---|---|
| Setup de infraestrutura (Fabric Workspace, Lakehouse, GitHub, venv) | ✅ concluído (28/05) |
| Ingestão Bronze (36 parquets + lookup zones) | ✅ concluído (29/05) |
| Limpeza Silver (filtros, derivações, flags) | 🟡 em andamento (30/05) |
| Modelagem Gold + Enrichment (ACS + NOAA) | ⏳ |
| Modelo semântico Power BI (DirectLake) | ⏳ |
| Features DAX (Calc Groups, UDFs, RLS, Field Parameters) | ⏳ |
| Dashboard (5 páginas) | ⏳ |
| Apresentação (.pptx) | ⏳ |

## Highlights do desenvolvimento

- **119.136.044 trips** ingeridas em Delta (Bronze) a partir de 36 arquivos parquet (~1.85 GB).
- **Schema canônico explícito** para resolver schema drift entre arquivos da TLC (INT vs BIGINT, `airport_fee` vs `Airport_fee`).
- **Idempotência** na ingestão: re-execução não duplica arquivos nem dados.
- **663 registros descobertos com data fora do escopo** (datas de 2001 a 2026) — filtrados na camada Silver, documentados como achado de qualidade.
- **Padrão YoY identificado**: -3.4% em 2023, +7.5% em 2024 — narrativa de demand recovery pós-pandemia.

Decisões técnicas e respostas-padrão para apresentação em [`docs/PRESENTATION_NOTES.md`](docs/PRESENTATION_NOTES.md).

## Critérios do teste

- ✅ Escopo: Yellow Taxi 2022–2024
- ✅ Fonte externa correlacionada (ACS Demographics + NOAA Weather)
- ✅ Features obrigatórias planejadas: Time Intelligence, Field Parameters, Calculation Groups, Conditional Formatting, RLS, UDFs

## Autor

Nicolas Augusto — Campinas/SP — [LinkedIn](#)

## Licença

[MIT](LICENSE)
