# Changelog — Nível 9: Transparência Institucional & Auditabilidade

> **Data:** 2 de Maio de 2026  
> **Commits:** `22c9b41` → `82abf81` → `67d4bb1`  
> **Status:** ✅ Em Produção  

---

## Objetivo

Elevar o NEXUS ao standard de **maturidade operacional institucional**, introduzindo um rasto de auditoria imutável, relatórios de performance diários automáticos e monitorização visual da correlação entre estratégias — eliminando as últimas "caixas negras" do sistema.

---

## 3 Componentes Implementados

---

### 1. Audit Trail Formalizado — `engine/audit_logger.py`

**Problema anterior:** As decisões da `CentralGovernance` (vetos de IA, multiplicadores de VIX/DD/Correlação) eram aplicadas mas nunca registadas de forma permanente e estruturada.

**Solução:**
- Nova base de dados isolada: `nexus_audit.db` (separada da `quant_platform.db` para evitar contenção de I/O no *order engine*)
- Tabela `audit_log` com campos: `timestamp`, `action_type`, `strategy_id`, `symbol`, `user`, `decision_json` (estado completo serializado), `outcome`
- **Injeção em `engine/central_governance.py`**: cada saída do `evaluate_trade()` grava automaticamente o estado completo da `GovernanceDecision` (dataclass via `asdict`)
- **Fail-Safe**: erros de escrita no audit log nunca bloqueiam a execução de trades

**Decisão de Design:**  
> A base de dados de auditoria é propositadamente separada da principal. Uma escrita lenta no audit log **não pode, em nenhuma circunstância, aumentar a latência** de uma decisão de trade ao vivo.

---

### 2. Night Reports — `engine/night_reports.py`

**Problema anterior:** Não havia forma sistemática de saber: *"quanto perdemos em slippage esta semana?"* ou *"qual estratégia gerou o alpha real vs. o paper?"*

**Solução:**

```
NightReporter
├── generate_daily_report()    → Agrega trades do dia por estratégia
├── pnl_attribution()          → PnL Real por Estratégia + Regime
├── live_vs_paper_reconcile()  → Divergência: Live PnL vs Paper Expected
└── log_and_alert()            → Telegram alert se divergência > 5%
```

**Proteção LTI:** Os assets LTI (`VWCE`, `SXR8`, `IMAE`, `EUNA`, `4GLD`, `SPYD`) são **hardcoded como exclusões** — nunca contaminarão as métricas de alpha tático do NEXUS.

**Integração:** O `ops_agent.py` chama o `NightReporter` dentro do `_send_digest()` diário (enviado por Telegram todas as noites de semana). O Daily Digest agora inclui:
- Live Net PnL do dia
- Total Slippage
- Divergência Live vs Paper (alerta `⚠️` se > 5%)
- Contagem de legs LTI ignoradas

---

### 3. Strategy Correlation Heatmap (Real) — Torre de Controlo

**Problema anterior:** O card *"Strategy Correlation"* no tab Risk & LTI da Torre mostrava **valores aleatórios** gerados por pseudo-correlação de temperatura. Dados completamente fictícios.

**Solução:**

#### Backend — `api/routes.py`
Novo endpoint: `GET /api/analytics/strategy-correlation?days=30`
- Calcula **Correlação de Pearson** entre os vectores de PnL diário de cada estratégia
- Algoritmo puro Python (sem NumPy/Pandas) — zero dependências novas
- Zero chamadas à API IBKR — apenas SQLite
- LTI-safe: filtra apenas tabela `trades` por `strategy_id`
- Retorna: matriz NxN + lista de pares críticos (> 0.80)

#### Proxy — `torre.py`
Novo endpoint: `GET /torre/api/strategy-correlation` → proxy para o QM

#### Frontend — `torre_app.js`
Função `renderStratCorrHeatmap()` completamente reescrita:
- **Escala de cor real**: Verde (< 0.20) → Amber (0.40–0.60) → Vermelho (> 0.80)
- Diagonal sempre Cyan (correlação consigo mesma = 1.0)
- Hover interativo com tooltip (valor exato a 3 casas decimais)
- Lista de pares em alerta abaixo do grid com ícones 🔴/🟡

---

## Separação de Responsabilidades Atualizada

| Módulo | Responsabilidade |
|---|---|
| `audit_logger.py` | Registo imutável de todas as decisões de governance |
| `night_reports.py` | Atribuição de PnL e reconciliação Live vs Paper |
| `central_governance.py` | Motor de decisão quantitativa (agora auditável) |
| `api/routes.py` | Endpoint de correlação de estratégias (Pearson real) |
| `torre_app.js` | Visualização do heatmap com dados reais |

---

## Git Commits (quant-mentor)

| Hash | Mensagem |
|---|---|
| `22c9b41` | feat: Institutional Phase 1 - Audit Trail and Night Reports |
| `82abf81` | feat: Strategy Correlation Matrix API endpoint |
| `67d4bb1` | fix: ops_agent digest syntax error - resources block outside except |

## Git Commits (torre-controlo)

| Hash | Mensagem |
|---|---|
| `450b2d8` | feat: Strategy Correlation Heatmap - real data via Pearson PnL correlation |

---

## Estado de Serviços Verificado (20:09 CET)

| Serviço | Estado |
|---|---|
| `quant-mentor` | ✅ active — a aguardar IB Gateway (fim de semana) |
| `torre-controlo` | ✅ active |
| `ops-agent` | ✅ active (corrigido syntax error do digest) |
| `lti-engine` | ⏸️ inactive (normal ao fim de semana) |

---

## Regra Arquitectural Chave (Nível 9)

> O NEXUS opera agora com **full observability**: cada decisão de risco fica gravada para sempre, cada euro de slippage é contabilizado, e a concentração de risco entre estratégias é visível em tempo real na Torre de Controlo.

---

## Adenda — Nível 9.2 + 9.3 (2 Maio 2026)

### Commits: `59c6056` (quant-mentor/main)

### 9.2 — Parameter Versioning com Config Canary

**Ficheiro:** `engine/param_versioning.py`

**BD:** `nexus_config.db` (nova, isolada da quant_platform.db)

**O que rastreia (16 parâmetros):**
- Todos os limites de risco: `MAX_DRAWDOWN_LIMIT=0.08`, `DAILY_LOSS_LIMIT=0.03`, `VIX_HALT_LEVEL=45`, `MAX_RISK_PER_TRADE=0.01`, etc.
- Hashes SHA256 dos 7 YAMLs de estratégia (detecta edições de ficheiro)
- Hash do commit git activo

**Comportamento:**
- Snapshot #1 gravado com sucesso no arranque (trigger: `supervisor_start`, commit `59c6056`)
- Config Canary: alerta Telegram se `error_rate_post > 3× error_rate_pre` após mudança
- Rollback **nunca automático** com motor live — apenas alerta manual

**API (disponível quando Gateway UP):**
- `GET /api/config-snapshot` — snapshot activo + diff do anterior
- `GET /api/config-history?param=MAX_DRAWDOWN_LIMIT&n=20` — historial por parâmetro

### 9.3 — Backtesting as a Service (BaaS)

**Ficheiros:**
- `engine/bar_cache.py` — Cache SQLite de barras históricas + Yahoo Finance fallback
- `engine/backtest_service.py` — Fila FIFO de jobs + worker thread `nice+15`

**Protecções activas:**
- Worker corre em `nice +15` → OS priva-o de CPU vs order engine
- Yahoo Finance (`yfinance`) instalado no venv como fonte de dados primária
- IB Gateway **nunca consultado** durante RTH (09:30–16:00 ET)
- Jobs Outside RTH prioritários; RTH → QUEUED até after-hours
- Máximo 1 job simultâneo (sem race conditions)
- Timeout por job: 300 segundos
- Usa `StrategyFactory.GENETIC_CLASS_MAP` para instanciar estratégias (mesmo mecanismo do Incubator genético)

**API:**
- `POST /api/backtest/submit` — `{strategy_id, symbol, start, end, params_override}`
- `GET /api/backtest/status/<job_id>` — estado + resultado completo
- `GET /api/backtest/list` — últimos 50 jobs
- `POST /api/backtest/cancel/<job_id>` — cancela jobs em QUEUED

**Resultado de um job DONE inclui:**
`sharpe`, `max_drawdown`, `win_rate`, `profit_factor`, `total_trades`, `total_return_pct`, `expectancy`, `final_equity`

### Integração no Supervisor
`run_forever.py` chama `ParameterStore.snapshot_current()` no arranque — garante auditoria de todas as mudanças de config entre deploys.
