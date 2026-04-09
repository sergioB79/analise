# Chart Analysis Pipeline

Sistema de análise automática de gráficos TradingView por asset e timeframe, com publicação em GitHub Pages.

---

## O que faz

1. Lê screenshots de gráficos TradingView da pasta `screenshots/`
2. Faz upload das imagens para GitHub (sem passar pelo AI)
3. Classifica cada screenshot por asset e timeframe via vision (Haiku)
4. Analisa cada gráfico com as instruções do template correspondente (Sonnet)
5. Consolida a análise por asset (3 TFs → 1 report por asset)
6. Produz visão macro com todos os assets
7. Publica página HTML com thumbnails clicáveis, análise e custo do run
8. Move screenshots para `processadas/YYYY-MM-DD/`

**Um comando. ~2 minutos. ~$0.33 por run com 3 assets e 9 gráficos.**

---

## Stack

| Componente | Tecnologia | Custo |
|---|---|---|
| Script local | Python + Modal SDK | Grátis |
| Pipeline cloud | Modal.com | ~$0/mês compute |
| Classificação | Claude Haiku (vision) | ~$0.015/run |
| Análise de gráficos | Claude Sonnet (vision) | ~$0.30/run |
| Hosting página | GitHub Pages | Grátis |

---

## Estrutura

```
chart-pipeline/
    screenshots/          ← pões os .png aqui antes de correr
    processadas/          ← imagens movidas automaticamente após run
        2026-04-09/
        2026-04-10/
    instrucoes/           ← templates de análise por asset/TF
        EURUSD_scalp.md
        EURUSD_swing.md
        EURUSD_position.md
        GER40_scalp.md
        GER40_swing.md
        GER40_position.md
        XAUUSD_scalp.md
        XAUUSD_swing.md
        XAUUSD_position.md
        OUTPUT_FORMAT.md
    run.py                ← script local — único ponto de entrada
    pipeline.py           ← pipeline Modal (deploy uma vez)
    config.json           ← keys locais (não commitar)
```

---

## Configuração inicial

### 1. Instalar dependências

```bash
pip install modal
```

### 2. Autenticar Modal

```bash
python -m modal token new
```

### 3. Criar segredos no Modal

```bash
modal secret create chart-pipeline \
  ANTHROPIC_API_KEY=sk-ant-... \
  GITHUB_PAT=ghp_...
```

### 4. Criar volume de instruções

```bash
modal volume create chart-instrucoes
modal volume put chart-instrucoes instrucoes/
```

### 5. Criar `config.json` local

```json
{
    "GITHUB_PAT": "ghp_...",
    "GITHUB_REPO": "sergioB79/analise"
}
```

> **Nota:** `config.json` está no `.gitignore` — nunca commitar keys.

### 6. Deploy da pipeline

```bash
python -m modal deploy pipeline.py
```

---

## Uso diário

```bash
# 1. Tirar screenshots no TradingView e guardar em screenshots/
# 2. Correr o script
python run.py
```

Output no terminal:

```
Chart Analysis Pipeline
────────────────────────────────────────
Data: 2026-04-09
Screenshots encontrados: 9
  · EURUSD_2026-04-09_...png
  · GER40_2026-04-09_...png
  · XAUUSD_2026-04-09_...png
────────────────────────────────────────
→ A fazer upload das imagens para GitHub...
→ Custo estimado: ~$0.22
A enviar para Modal...
────────────────────────────────────────
Pipeline concluida.
Página actualizada: https://sergiob79.github.io/analise/2026-04-09.html
Assets processados: EURUSD, GER40, XAUUSD
Duração: 136s
────────────────────────────────────────
CUSTO DO RUN:
  Haiku:   $0.015
  Sonnet:  $0.318
  ──────────────────────────
  TOTAL:   ~$0.333
────────────────────────────────────────
9 ficheiros movidos para processadas/2026-04-09/
```

---

## Templates de análise

Cada asset tem 3 ficheiros `.md` em `instrucoes/`:

| Categoria | Timeframes | Ficheiro |
|---|---|---|
| scalp | 5M · 15M · 30M | `{ASSET}_scalp.md` |
| swing | 1H · 2H · 4H | `{ASSET}_swing.md` |
| position | 12H · 1D | `{ASSET}_position.md` |

O ficheiro `OUTPUT_FORMAT.md` define o formato universal de report — injectado em todos os subagentes.

Para actualizar instruções após editar os `.md`:

```bash
modal volume put chart-instrucoes instrucoes/
```

---

## Indicador TradingView (identificação de TF)

Cada template no TradingView deve ter este indicador Pine Script adicionado ao gráfico:

```pine
//@version=5
indicator("Chart Label", overlay=true)
asset = input.string("EURUSD", "Asset", options=["EURUSD", "GER40", "XAUUSD", "SP500", "WTI", "USDJPY"])
tf = input.string("1D", "Timeframe", options=["15M", "30M", "1H", "4H", "12H", "1D"])
corner = input.string("Top Right", "Posição", options=["Top Right", "Top Left", "Bottom Right", "Bottom Left"])
pos = corner == "Top Right" ? position.top_right : corner == "Top Left" ? position.top_left : corner == "Bottom Right" ? position.bottom_right : position.bottom_left
var table t = table.new(pos, 1, 1)
table.cell(t, 0, 0, asset + " | " + tf, text_color=color.white, bgcolor=color.new(color.black, 20), text_size=size.large, text_halign=text.align_center)
```

Fica fixo no canto do gráfico independentemente do scroll ou zoom. Configura o asset e TF nos inputs e guarda no template.

---

## Assets suportados

`EURUSD` · `GER40` · `XAUUSD` · `SP500` · `WTI` · `USDJPY`

---

## Página de output

**Índice:** `https://sergiob79.github.io/analise/`
**Análise por data:** `https://sergiob79.github.io/analise/YYYY-MM-DD.html`

Cada página inclui:
- Custo real do run (tokens Haiku + Sonnet)
- Visão macro consolidada
- Card por asset com thumbnails clicáveis dos gráficos
- Análise multi-TF com setups, níveis e invalidações

---

## Notas

- O sistema processa o que existir — dias sem screenshots ou com assets incompletos funcionam normalmente
- Screenshots são movidos para `processadas/` após cada run — a pasta `screenshots/` fica sempre limpa
- O histórico de análises é preservado — cada run cria uma página nova, o índice acumula
- As imagens são hospedadas no próprio repositório GitHub — sem custo adicional de storage

---

*S. Batalha · Nazaré · 2026*
