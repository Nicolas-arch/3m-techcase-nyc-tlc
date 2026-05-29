# Checkpoint Review — Meio do Projeto (31/05/2026)

> Revisão intermediária executada antes de partir para a camada de visualização. Objetivo: validar aderência ao Tech Case, identificar gaps e mitigar riscos antes que mudanças se tornem caras.

## Contexto

Após 4 dias de desenvolvimento (28–31/05), o projeto está aproximadamente em **50% do escopo total**: toda a engenharia de dados (Bronze, Silver, Gold) está concluída e validada; o modelo semântico e o relatório Power BI são as próximas etapas. Este é o ponto de máximo retorno para validação — antes que o Power BI seja construído sobre uma fundação potencialmente incorreta.

## 1. Cobertura do Tech Case (Part 1 – obrigatório)

| Requisito do briefing | Status | Como entreguei |
|---|---|---|
| Escopo Yellow Taxi 2022-2024 apenas | ✅ | 119.136.044 trips no Bronze, todas do escopo |
| Construir Power BI report | 🟡 | Modelo dimensional pronto, semantic model próximo |
| Sem instruções específicas — candidato decide métricas | ✅ | KPIs propostos em PRESENTATION_NOTES seção 10.1 |
| Sem problem statement formal — candidato propõe | ✅ | Supply chain framing documentado |
| Discutir Data Intake / Ingestion | ✅ | Notebook 01 + ARCHITECTURE.md |
| Discutir Harmonization / Cleansing | ✅ | Notebook 02 + 2,84% descartado documentado |
| Discutir Enrichment | ✅ | ACS + NOAA com justificativa |
| Discutir Data Model | ✅ | Star schema 1 fato + 6 dim |
| Discutir Front End | ⏳ | Próxima fase |
| Time Intelligence functions | ⏳ | Pendente Calc Group "Time Analytics" |
| Field Parameters | ⏳ | Pendente FP_Metric + FP_Dimension |
| Calculation Groups | ⏳ | Pendente via Tabular Editor 2 |
| Conditional formatting | ⏳ | Pendente nos visuais |
| Security roles (RLS) | ⏳ | Pendente 5 roles por Borough |
| UDFs | ⏳ | Pendente 3 UDFs DAX |
| Fonte externa correlacionada | ✅ **EXCEDIDO** | 2 fontes (ACS + NOAA), brief pedia 1 |
| Justificar fonte + insights | ✅ | PRESENTATION_NOTES seção 7 + Gold layer |
| .pbix enviado antes da reunião | ⏳ | Buffer reservado para 08/06 |

**Veredito da Part 1**: 9 de 17 itens entregues (53%). Os 8 pendentes estão concentrados na fase de visualização — todos viáveis nos 7 dias restantes.

## 2. Critérios de avaliação (judgment criteria)

| Critério | Status atual | Como sustentar |
|---|---|---|
| Perspectiva analítica e abordagem sobre os dados | ✅ | Supply Chain framing, 12 mapeamentos taxi↔SC documentados |
| Proposição de propósito (storytelling) | ✅ | Narrativa "demand recovery pós-pandemia" suportada pelos dados (-3,4% YoY 2023, +7,5% 2024) |
| Facilidade de entender dados e narrativa | 🟡 | Vai depender muito do design do Power BI |
| Design do relatório servindo à proposição | ⏳ | Próxima fase |
| Aplicação de habilidades técnicas | ✅ | Fabric Lakehouse, schema canônico, PySpark + Spark SQL, REST API paginado, COALESCE para integridade, GitHub versionado |

## 3. Strengths (pontos fortes para destacar na apresentação)

1. **Schema drift na ingestão TLC encontrado e resolvido** — solução com schema canônico explícito + cast file-by-file. Mostra atenção a dados reais.
2. **Achados de qualidade de dados quantificados** — 663 vazamentos temporais + 2,1M zero-distance + 1,4M tarifa negativa + 61K duração inválida. Total descartado 2,84%, documentado.
3. **Top anomalia identificada**: trip de US$ 401.092 em 10min — caso clássico de exception management.
4. **Padrão YoY identificado**: -3,4% em 2023, +7,5% em 2024 — base para narrativa de demand recovery.
5. **Padrão híbrido para fontes externas**: NOAA via REST API (dados operacionais), ACS via snapshot (master data). Padrão típico de produção.
6. **Limite de 1 ano da NOAA CDO encontrado e documentado** — fix via loop por ano. Tipo de gotcha que só se aprende rodando.
7. **Integridade referencial validada**: zero foreign keys orfãs no fato (graças ao padrão COALESCE → Unknown).
8. **Documentação contínua**: ARCHITECTURE, CONVENTIONS, DATA_SOURCES, PRESENTATION_NOTES, CODE_EXPLAINED — total ~30K palavras versionadas.
9. **Repo público no GitHub desde o dia 1** — versionamento real, conventional commits, badges, README atualizado a cada milestone.

## 4. Gaps (a corrigir antes do Power BI)

| Gap | Severidade | Ação |
|---|---|---|
| `DATA_DICTIONARY.md` referenciado no README mas vazio | Média | Gerar com schema completo do Gold |
| `NOAA_TOKEN` ainda hardcoded em Cell 8 do Notebook 03 | **Alta** (vai vazar no commit) | Substituir por placeholder antes de exportar |
| Sem "business sanity query" rodada | Baixa | Executar 1 query analítica representativa contra o Gold |
| Versão Power BI Desktop não revalidada | Média | Confirmar ≥ 2.140 |
| Preview features (UDFs, Calc Groups, DirectLake) não revalidadas | Média | Confirmar habilitados |

## 5. Riscos para a fase Power BI (01–04/06)

| Risco | Impacto | Probabilidade | Mitigação |
|---|---|---|---|
| DirectLake + Calc Groups + Field Parameters incompatíveis em preview | Alto | Média | Testar combinação no início do dia 02/06; se conflitar, manter Calc Group e usar slicer tradicional |
| UDFs DAX (preview) com bug bloqueando feature obrigatória | Alto | Baixa | Testar 1 UDF simples antes; fallback é usar SWITCH/IF inline |
| 115M linhas via DirectLake lentas em interação | Médio | Média | Já temos particionamento year/month; se precisar, criar aggregation table dia/borough |
| .pbix com semantic model DirectLake estoura limite de tamanho do PBI Free | Médio | Baixa | DirectLake só guarda metadados; não importa dados; OK |
| Crosswalk Borough-level pode ser questionado pelo avaliador | Baixo | Média | Resposta pronta em PRESENTATION_NOTES (decisão consciente, opção 2 documentada) |

## 6. Plano dos 7 dias restantes (01–08/06)

| Data | Foco | Horas | Entrega |
|---|---|---|---|
| Dom 01/06 | Semantic model + Power BI Desktop + medidas base | 4h | Modelo conectado, 10 medidas base |
| Ter 02/06 | Calc Groups + Field Parameters + UDFs | 4h | Features DAX preview validadas |
| Qua 03/06 | Páginas 1-2 (Executive Overview + Demand Deep Dive) | 5h | 2 páginas com conditional formatting |
| Qui 04/06 | Páginas 3-5 (Operations, Demographics, Data Quality) + RLS | 5h | Relatório completo + RLS testada |
| Sex 05/06 | Apresentação PPTX estrutura (slides 1-8) | 4h | Deck v1 |
| Sáb 06/06 | Apresentação polimento + revisão PBI | 6h | Deck v2 final + .pbix candidate |
| Dom 07/06 | Buffer + ensaio | 4h | Ensaio cronometrado validado |
| Seg 08/06 | Entrega | 4h | .pbix enviado + tag v1.0 |

## 7. Resposta de entrevista pronta sobre o checkpoint

> "No meio do projeto fiz uma revisão formal contra o briefing — confere o `CHECKPOINT_REVIEW.md` no repo. Identifiquei que toda a camada de dados estava aderente (Bronze/Silver/Gold + enrichment + integridade referencial validada) e priorizei a fase de visualização e DAX nos 7 dias restantes. Também fiz um stress-test mental dos riscos da próxima fase — DirectLake + UDFs são preview features, então testei combinações cedo no dia 2 antes de empilhar mais 4 dias de trabalho em cima. Esse tipo de checkpoint a meio caminho é o que diferencia projeto que entrega no prazo de projeto que descobre problemas faltando 1 dia."

## 8. Decisão

**Prosseguir para a fase Power BI** após executar os 5 gaps acima (estimativa: 1h total). Nenhum gap impede o avanço; todos são mitigáveis e melhoram a qualidade final.
