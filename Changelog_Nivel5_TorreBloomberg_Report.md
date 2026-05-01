# Changelog — Nível 5 (Torre Bloomberg, Heatmap & Report Patch)

**Data:** 1 de Maio de 2026
**Tags:** #nexus #torre #ui #bloomberg #heatmap #bugfix

## 🎯 Resumo Operacional
Elevação da Torre de Controlo ao nível de "Terminal Profissional" (Bloomberg-style), resolvendo simultaneamente o falso alerta de deadlock no Watchdog e o bug de contagem de trades no relatório semanal.

Toda a infraestrutura permanece imaculada e alinhada com o **Git Drift Guard v2.0**.

---

## 🛠️ O que foi implementado

### 1. Torre de Controlo — "Bloomberg Terminal Dock"
- **CLI Tática Integrada:** Linha de comandos responsiva e focável no rodapé (`nexus >`).
- **NEXUS Command Interface:** Implementação nativa de atalhos e relatórios instantâneos via terminal (ex: `/status TSLA`, `/vix`, `/help`, `/liquidate`).
- **Glassmorphism Expandable Dock:** A barra inferior de métricas rápidas converte-se fluidamente no terminal principal ao clique (UX otimizado).

### 2. Market Heatmap & AI Newswire
- **Heatmap (S&P 500 Sectors):** Novo painel visual integrado no separador "Overview", renderizando a performance percentual e cromática de ETFs setoriais base (`XLK`, `XLF`, `XLE`, etc.).
- **AI Newswire:** A infraestrutura processa agora o feed de RSS do Yahoo Finance, aplicando uma camada rudimentar de análise de sentimento e rotulagem (`🟢 BULLISH`, `🔴 BEARISH`, `⚪ NEUTRAL`). Notícias pertinentes aos ativos controlados recebem a etiqueta de vigilância `⚡ NEXUS`.

### 3. Patch Crítico no Motor de Relatórios & Watchdog Deadlock Fix
- **Bug `Total Trades: 0`:** O `report_generator.py` foi patcheado para deixar de ler o obsoleto `trades.log` de texto. Os relatórios semanais são agora gerados através de consultas diretas em SQL à nova infraestrutura `quant_platform.db` (tabela `trades`), discriminando com precisão vitórias, derrotas e win-rate baseando-se no `created_at` e no lucro/prejuízo real (`pnl`).
- **Resolução de Deadlock (Watchdog):** Foi orquestrado um arranque isossíncrono (`simultaneous restart`) entre o `quant-mentor` e o `ops-agent`, evitando falsos-positivos de "Stale Heartbeat" durante os deployments de manutenção.

---

## 🔒 Status Git Guard & Daemons

| Ecossistema | Último Commit | Status |
|---|---|---|
| `quant-mentor` (NEXUS) | `0b434a3` — SQLite db for weekly reports | ✅ Running |
| `torre_controlo` (Dashboard) | `bc194fe` — Heatmap + AI Newswire | ✅ Running |

Todos os serviços cumprem os limites de `StartLimitBurst` e estão 100% monitorizados pelo Operations Agent.
