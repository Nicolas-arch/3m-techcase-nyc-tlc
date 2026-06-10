# Home / Menu Page — Build & Navigation Guide

Executive landing page for the **3M Tech Case — NYC Yellow Taxi as an Urban Supply Chain** report.
Matches the existing 5 pages: **1920×1080, navy `#1F3864`, Segoe UI, English.**

Asset: `home_menu_3M_taxi_1920x1080.png` (drop-in page background).

---

## A) Diagnóstico

- **Tipo de dado:** navegação / landing executiva (não analítica). Resume o report e direciona para as 5 telas.
- **Público:** banca técnica da 3M (C-level + analistas de Supply Chain Analytics). Primeira tela que eles veem.
- **Pergunta que a tela responde:** "O que é este report e por onde eu começo?"
- **Tom visual:** dark navy executivo + amarelo táxi como cor de ação. Consistente com as 5 telas.
- **Referências de mercado:** dashboards dark fintech do Dribbble/Behance (navy + 1 accent), launchers estilo bento (Apple TV / iOS), e o padrão "navigation hub" recomendado pela Microsoft para relatórios Power BI multi-página (botões + Page Navigation).

---

## B) Design system da página

```
CANVAS        1920 × 720? -> 1920 × 1080 (16:9, igual às outras telas)
FUNDO         gradiente navy #0A152B (topo) -> #172D52 (base) + glows suaves
CARDS         vidro (branco ~6%) , raio 18px, borda branca ~12%, sombra suave

TIPOGRAFIA (Segoe UI no PBI; Carlito/Calibri no mockup)
  Título hero      62px  Bold   #F4F8FD
  Subtítulo        23px  Reg    #96AACA
  Valor de KPI     38px  Bold   #F4F8FD (verde nos positivos)
  Título de card   21px  Bold   #F4F8FD
  Descrição/label  13–14px Reg   #96AACA
  Eyebrow (CAPS)   14px  Reg    #FFC20E (tracking)

PALETA (5 cores)
  Navy  #1F3864  -> fundo / base
  Amarelo táxi #FFC20E -> ação, destaque, ícones, arrows
  Verde #4AD68E -> métrica positiva (+5.7% YoY)
  Branco #F4F8FD -> texto primário
  Azul-acinzentado #96AACA -> texto secundário
```

---

## C) Wireframe — mapa de posições (px, canvas 1920×1080)

```
+---------------------------------------------------------------------------+
| [logo] NYC YELLOW TAXI                      3M · ENTERPRISE SUPPLY CHAIN   |  Header  y0–126
|        Supply Chain Analytics               NYC TLC · 2022–2024            |
+---------------------------------------------------------------------------+
| EXECUTIVE REPORT · NAVIGATION          +-----------------------------+     |
| Every trip is a shipment.              | How we read the data        |     |  Hero   y150–430
| Turning 116M NYC taxi rides into       |  Trip        -> Shipment    |     |
| supply-chain intelligence...           |  Duration    -> Lead time   |     |
|                                        |  Zone        -> DC/Customer |     |
|                                        |  Fare        -> Logistics $ |     |
+---------------------------------------------------------------------------+
| 116M        | $3.06B      | 17.6 min   | +5.7%       | 97.16%            |  KPIs   y470–588
| trips       | logistics $ | lead time  | YoY demand  | data retention    |
+---------------------------------------------------------------------------+
| EXPLORE THE REPORT                                       5 pages           |
| +--------+  +--------+  +--------+  +--------+  +--------+                  |
| | 01 ⭐  |  | 02     |  | 03     |  | 04     |  | 05     |                  |  Nav    y648–960
| | Exec   |  | Demand |  | Ops &  |  | Demo-  |  | Data   |                  |
| | Open-> |  | Open-> |  | Open-> |  | Open-> |  | Open-> |                  |
| +--------+  +--------+  +--------+  +--------+  +--------+                  |
+---------------------------------------------------------------------------+
| Sources: NYC TLC · NOAA · Census ACS        Fabric · Power BI · GitHub     |  Footer y996+
+---------------------------------------------------------------------------+
```

Grid de 5 colunas: cada coluna `x = [72, 432, 792, 1152, 1512]`, largura **336px**, gap **24px**, margens laterais **72px**. KPIs e cards de navegação compartilham o mesmo grid (alinhamento perfeito).

---

## D) Os 5 botões de navegação

| # | Página destino | Ícone | Microcopy |
|---|---|---|---|
| 01 | Executive Overview | dashboard | "The whole story on one screen — revenue & KPIs." (START HERE) |
| 02 | Demand Deep Dive | clock | "When & where demand peaks across the city." |
| 03 | Operations & Cost | gauge | "Lead time, efficiency and cost bottlenecks." |
| 04 | Demand vs Demographics | people | "Who is under- or over-served (ACS data)." |
| 05 | Data Quality & Governance | shield | "Trust, security, RLS & exception management." |

Coordenadas de cada card — **versão simples** `home_menu_3M_taxi_simple_1920x1080.png` (use esta para posicionar os botões transparentes):
`y = 452`, `altura = 348`, `x ∈ {72, 432, 792, 1152, 1512}`, `largura = 336`.
(Versão completa `home_menu_3M_taxi_1920x1080.png`: `y = 648`, `altura = 312`, mesmas colunas e largura.)

---

## E) Como montar no Power BI (caminho recomendado)

**Técnica:** imagem de fundo + botões transparentes com Page Navigation. É o padrão profissional — visual 100% controlado, e os botões ficam só com a ação.

1. **Nova página** → renomeie para `Home` (ou `Menu`). Arraste-a para ser a 1ª página.
2. **Tamanho do canvas:** View → Page view → *Actual size*. Format page → Canvas settings → **Type: 16:9** (1920×1080, igual às outras telas).
3. **Fundo:** Format page → **Canvas background** → Browse → selecione `home_menu_3M_taxi_1920x1080.png` → **Image fit: Fit** → **Transparency: 0%**.
   - Dica: deixe *Wallpaper* (fora do canvas) na mesma cor navy `#0A152B` para a tela não "vazar" branco.
4. **Botões de navegação (×5):** Insert → Buttons → **Blank**.
   - Para cada card, posicione o botão exatamente sobre ele (use o painel Format → General → Properties: X/Y/Largura/Altura com os valores da seção D).
   - Format do botão → **Fill: Off** e **Border: Off** (botão invisível — o desenho já está no fundo).
   - **Action → Type: Page navigation → Destination:** a página correspondente (01→Executive Overview, etc.).
   - (Opcional) Em **Style → On hover**, ligue um Fill amarelo com ~12% de transparência para dar feedback no hover.
5. **Botão Home/Voltar nas 5 telas:** em cada página de conteúdo, Insert → Buttons → **Blank** (ou ícone), Action → Page navigation → `Home`. Coloque no canto superior esquerdo. Assim o menu vira um sistema de navegação fechado.
6. **Esconder as abas para o usuário final:** ao publicar/apresentar, use View → desligue *Page tabs* (ou no Service: Settings do report). A navegação passa a ser só pelos botões.

### KPIs ao vivo (opcional, recomendado para impressionar)
Os números do mockup (116M, $3.06B, 17.6 min, +5.7%, 97.16%) estão "queimados" na imagem. Para deixá-los **dinâmicos** (e respeitando o filtro de ano, por exemplo):

- Não pinte os valores no fundo: gere uma 2ª versão do PNG só com os rótulos (sem os números), ou cubra a faixa de KPI com 5 **Card** visuals transparentes posicionados em `y≈470`, alturas 118, mesmas 5 colunas.
- Cada Card usa as medidas já existentes: `[Total Trips]`, `[Total Revenue]`, `[Avg Lead Time (min)]`, `[YoY % Revenue]`, `[Retention %]`.
- Format do Card → Background: Off, Border: Off, Callout value e label nas cores do design system.

> Para a apresentação: KPIs estáticos no menu são aceitáveis (é uma capa). Mas Cards ao vivo mostram domínio técnico e mantêm o menu correto se os dados forem atualizados. Recomendo os Cards ao vivo se sobrar tempo.

---

## F) Por que estas escolhas (decisões de design)

- **Dark navy + amarelo táxi:** o amarelo é a cor icônica do yellow cab — vira a cor de "ação/navegação" sem precisar inventar paleta. Navy mantém a consistência com as 5 telas e o tom executivo (padrão em dashboards financeiros 2024–2025).
- **Painel "How we read the data":** coloca a narrativa central (trip = shipment) logo na capa. É o gancho de storytelling que a banca vai lembrar — e abre a apresentação sozinho.
- **KPIs como teaser:** dão um "executive snapshot" antes mesmo de clicar, e provam que o report tem substância.
- **Card 01 destacado (START HERE):** reduz a carga cognitiva — diz ao avaliador por onde começar. Bento/launcher layout é o padrão moderno de hub de navegação.
- **Imagem de fundo + botões transparentes:** performático (zero visuals pesados na capa), escalável (trocar o PNG troca o visual todo) e fácil de manter.

---

## G) Checklist de qualidade

```
[x] Objetivo claro: capa + navegação para 5 telas
[x] Hierarquia visual topo->base respeitada
[x] <= 6 blocos visuais (header, hero, painel, KPIs, nav, footer)
[x] Cores com significado (amarelo=ação, verde=positivo)
[x] Tipografia consistente com o report (Segoe UI)
[x] Grid de 5 colunas alinhando KPIs e botões
[x] Sem tabela gigante, sem pizza, sem 3D
[x] Tendências de mercado aplicadas (dark, glass, bento)
[x] Contraste adequado (texto claro sobre navy)
[x] Performance: fundo é imagem + botões (sem DAX na capa)
```
