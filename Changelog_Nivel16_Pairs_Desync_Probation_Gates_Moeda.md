# Changelog Nível 16 — Incidente PAIRS CAT/XOM (12/10/2026), gates de probação e moeda

**Data:** 12/10/2026 · **Commits:** `77e5ce9`, `f167c8f`, `4d0d946` (+ commit dos gates/moeda do mesmo dia) · **Repo:** `quant-mentor`

## Incidente
Às 15:45 CEST o PAIRS abriu SELL 9 XOM / BUY 2 CAT. Às 15:46 o `PositionGuardian` emitiu `STATE_STORE_DESYNC` (alarme legítimo).
Em paralelo, a `BAC` (-32, short órfão de 09-10/10) foi liquidada: stop `#166139` cancelado, 32 recompradas a 54.50 (PnL -$9.72).

## Causas-raiz (confirmadas com logs + código)
1. **Race condition em `engine/order_pairs.py`** — os handlers de fill só eram ligados ~3-5 s após `placeOrder`; fills em <500 ms nunca chegavam ao `state_store.json` (`ib_insync` não repete eventos passados). Fix: handlers ligados imediatamente + verificação `if trade.fills:`.
2. **`engine/reconciler.py::_get_account_value`** exigia `currency=='USD'`; a conta tem moeda-base **EUR** → devolvia 0.0 desde 08/04/2026 (commit `8353d335`) → `EXTERNAL-UNVERIFIED` + `GhostSweeper` cancelou o stop da CAT. Fix: aceitar EUR/BASE/USD + fallbacks.
3. **Auto-saneamento do `PairsMonitor`** (15:52:50) removeu o par de `open_pairs` por o `state_store` estar vazio (consequência de 1) → par aberto no broker sem gestão de saída.
4. **Bypass da probação:** o gate de entrada de pares só consultava `SafetyManager`; o PAIRS estava em probação no `ProbationEngine` (tabela `strategy_probation`) e operou em LIVE. Idem no `RapidScalper` (RSCALP/SCALP).
5. **Scanner de pares** (`engine/pairs_scanner.py`): combinava símbolos de qualquer setor e aceitava correlação negativa via `abs(corr)` → `XOM/CAT` (corr -0.83), `CVX/CAT`, `MSFT/CVX`.

## Correções (single source of truth)
- `order_pairs.py`, `reconciler.py` (causas 1-2).
- `engine/pairs_guard.py::is_pairs_quarantined(safety, probation_engine)` + novo `engine/quarantine_gate.py::is_strategy_blocked` (usado pelo RapidScalper).
- `pairs_scanner.py`: só pares **co-setor** (`SECTOR_MAP`), correlação **com sinal** ≥ 0.80. Re-execução: 0 pares aprovados de 23 testados (todos rejeitados por p-value) → PAIRS inativo.
- `config.py`: `MAX_PORTFOLIO_EXPOSURE` 0.80 → **0.70**; `MIN_RR_DEFAULT=1.0`, `MIN_RR_BY_STRATEGY={'PAIRS':0.0}`; gate central `OrderValidator` **CHECK 8.6** (`min_rr_violation`).
  - 1.5 foi avaliado e rejeitado: bloquearia 38/75 trades do MR (única estratégia lucrativa em Outubro) e 50/74 da `ROT_G12_pmip`.
- Moeda: novo `engine/currency.py::currency_symbol`; `IBConnector.get_trading_equity()` devolve `currency`; relatórios Telegram (`core/reporting.py`) deixam de assumir `$`.
- Par CAT/XOM fechado canonicamente (serviço parado → `clientId=2` cancela stops → market): XOM -$4.09, CAT +$8.06, comissões $2.03 (total +$3.97). BD #740/#741 `CLOSED` (`MANUAL_EXIT_PAIRS_UNTRACKED`); PnL calculado sobre fills reais.

## Testes
Suite completa: 983 passed (antes dos testes de moeda/gate). Novos: `test_rr_gate_and_pairs_scanner.py`, `test_pairs_scanner_sector.py`, `test_pairs_guard_probation.py`, `test_currency_and_quarantine_gate.py`, + 2 em `test_pairs_execution.py`.
`verify_system_integrity.py` → exit 0 após cada intervenção.

## Factos sobre a conta (verificados 12/10)
- Contas `DUP958757` (paper) e `U25744098` (live LTI): **moeda-base EUR** (NetLiquidation 89 303 / 20 068 EUR).
- Paper: caixa EUR +69 252, caixa **USD ≈ -10 762** (financiamento de margem); ExchangeRate USD→EUR 0.8963.

## Pendente / decisões abertas
- **Unidades mistas no sizing/exposição:** preços/risco em USD divididos por `account_value` em EUR (viés ~+11.6%, conservador). Corrigir exige converter equity para USD num ponto único — altera dimensionamento, **requer decisão do utilizador**.
- Entry price na BD é o preço do sinal, não o fill (afeta análise de slippage).
- PAIRS em probação sem saída automática (libertar manualmente após validar universo).
- Símbolos fora do `SECTOR_MAP` (MU, QCOM, NFLX, INTC) ficam excluídos do scanner.
- Torre de Controlo (repo separado) ainda pode rotular EUR como `$`.
- Incubadora: efeito do fix Nível 15 só visível no run de 18/10.

## Adendo (12/10/2026 tarde) — Scanner, MAE/MFE, Resolver

- **Scanner PAIRS** (b658b95): cointegração em log-preços, 252d, p<0.05. Pares: JPM/BAC, NVDA/AMD, NVDA/INTC (este e o mais fragil). PAIRS LIVE a observar durante a semana.
- **MAE/MFE/holding_bars a zero** (84bdbea): causa-raiz = `on_price_update` nunca tinha chamador (codigo morto). Agora `TradeAnalytics.update_excursions_from_bars` e chamado no fim de cada ciclo do `strategy_evaluator` com as barras do ciclo. Validado em producao: AVGO #742 mae -0.26%, mfe +0.55%, holding 28. Historico antigo fica a zero.
- **`scripts/signal_outcome_resolver.py`**: resolve sinais MOM/GAP desde 14/09 em trades virtuais (barras IB 5min, SL primeiro, EOD mark-to-close, sem custos). 1377 resultados.
  - MOM n=421 avgR -0.048; GAP n=38 avgR -0.20.
  - Variantes MOM_G3/G4 e GAP_G13/G14 sao clones exatos do base (resultados identicos).
  - MOM: BUY em TRENDING_UP avgR -0.126 (negativo nas duas metades temporais). Sem esses: +0.025; + hora ET<=10: +0.103 (positivo nas duas metades). Edge pequeno, antes de custos.
  - Proposta pendente: variante MOM em PAPER com esses filtros; retirar clones.

## Adendo (12/10/2026 noite) — Auditoria 14 estrategias, Fase 1 (P0)
- PairsStrategy: fair value sem alpha do Kalman (TP +174% em teste sintetico; +48% medio em producao). Corrigido (76eb300). Execucao de uma perna ja estava bloqueada desde 02/10 (signal_aggregator/order_engine); so contaminava `signals`.
- RSCALP: `record_trade` nunca era chamado -> limites diarios inoperantes. Agora `check_daily_limits` le a BD (entradas/P&L de hoje, UTC).
- ROT_G12_pmip: lifecycle LIVE -> PAPER (+ gene is_live=False), -334 USD, PF 0.57-0.69. Reversivel via set_stage. Continua em CANONICAL_LIVE_PERMANENT (so impede auto-demotion).
- Genes MOM/GAP traduzidos para parametros reais (af73ad8); MOM_F em PAPER.
- Pendente (Fase 2): GAP/SQZ regime 'TRENDING' invalido; METALS/M6E feed stale; AI deadlock; SCALP fantasma; despromocao de LIVE sem trades.
- Nota: estrategias em PAPER nunca acumulam trades na BD -> lifecycle nunca as promove (so signal_outcome_resolver mede).

## Adendo (12/10/2026 19:33) — Implementação Concluída: Fases 2 e 3 (Commit 63326be)
- **METALS & M6E (Futuros e Dados 24h Globex):** `data/ib_connector.py` corrigido para passar `useRTH=False` nos futuros (ouro, prata, euro, índices). Permite barras 24h contínuas (sessão asiática e Londres), eliminando o falso positivo de "dados stale de 64h" que bloqueava as estratégias. `engine/strategy_base.py` atualizado para reconhecer MGC, SIL e M6E como futuros. `circuit_breaker.py` e `data_quality.py` ajustados para tolerar a cadência normal de barras de 15m (até 1800s / 30m).
- **GAP & SQZ (Regimes Inválidos):** substituído o regime inexistente `'TRENDING'` por `{'TRENDING_UP', 'TRENDING_DOWN'}` em `engine/strategy_gap.py`, `engine/strategy_squeeze.py` e `strategy_sqz_config.yaml`. Ambas as estratégias passam a operar normalmente em dias de tendência.
- **AI Strategy (Deadlock do Modelo Sanado):** `engine/ai_learning.py` atualizado para carregar automaticamente de `ai_model_backup.joblib` caso o primário não exista. Ficheiro `models/ai_model.joblib` restaurado com sucesso. Log oficial confirma: `[AI] Learning system active — model=loaded`.
- **SCALP (Limpeza de Execução):** estratégia legada desativada no loop de ações do `core/scheduler.py` em favor do `RSCALP` (que opera o ciclo independente de 60s em micro-futuros).
- **MR (Recalibração de Confluência):** `strategy_config.yaml` atualizado para `min_signal_score: 2`, exigindo pelo menos uma confirmação real adicional (Volume, Bollinger ou SMA) além do RSI básico.
- **Governança Constitucional:** mantida a regra de auditoria manual para estratégias de `CANONICAL_LIVE_PERMANENT`, com emissão de alertas críticos de sub-desempenho no log quando métricas forem violadas.
- **Validação:** suite completa com 996 testes aprovados (0 falhas). Raio-X com Exit 0 (sem divergências entre Broker, BD, state_store e Torre).

## Adendo (12/10/2026 20:25) — Incubadora NEXUS v2: Fase P0 Concluída (Commits 6ab1dc7 e 427dd4e)
- **Contrato de Métricas Universal (P0.1 & P0.2):**
  - `data/database.py::get_strategy_stats`: atualizado para emitir o contrato completo de chaves (`REQUIRED_STATS_KEYS` em `engine/incubator_contract.py`). Adicionadas as métricas `total_trades`, `total_signals`, `sharpe_ratio` (anualizado via retornos trade-a-trade) e `max_drawdown` (fração 0.0–1.0 da curva cumulativa por `exit_time`).
  - `profit_factor`: limitado a 5.0 quando não há perdas (`total_loss == 0`), eliminando `float('inf')` que quebrava o JSON do frontend.
  - Parâmetro `source: str = 'trades'` adicionado com fallback seguro para `'paper_trades'` (preparação para a Fase P1).
  - Testes: `tests/test_incubator_contract.py` (7 testes cobrindo contratos vazios, só ganhos, só perdas, mistos e consumidores `IncubatorRanking`/`LifecycleManager`).
- **Aplicação Real de Genes via `GENE_PATHS` (P0.3):**
  - `engine/strategy_factory.py`: criado `GENE_PATHS` com caminhos aninhados precisos para `MR`, `MOM`, `VWAP`, `ORB`, `MFI`, `RSCALP`, `RSI_EXTREMES` e `AI`.
  - Helper `_set_nested` para manipulação recursiva de dicionários e listas de regras.
  - Gravação de `instance._applied_genes` em todas as variantes para rastreabilidade e auditoria forense.
  - `engine/strategy_rapid_scalper.py`: integrado com `sl_atr_mult` e `tp_atr_mult` dinâmicos da config.
  - Testes: `tests/test_gene_application.py` (3 testes validando mutações reais de config e alteração de comportamento de SL/lookback em MOM e MR).
- **Validação Cruzada & Saneamento de Fantasmas (P0.4 & P0.5):**
  - Adicionado `RSCALP` ao `GENETIC_CLASS_MAP` e `RSI_EXTREMES` ao `GENE_DEFS`, com assert-suave de integridade no `GeneticEngine.__init__`.
  - `spawn_replacement`: validação obrigatória via `StrategyFactory` antes de persistir o indivíduo no pool.
  - Script de migração `scripts/incubator_reseed.py`: backup automático para `backups/`, desativação de 12 indivíduos obsoletos (fantasmas e clones não parametrizados) e re-seeding de 13 substitutos verificados com parâmetros ativos (pool ativo com 24 slots preenchidos para as 12 bases).
  - `deploy.sh`: proteção de `incubator_gene_pool.json`, `incubator_ranking.json` e execução da migração com sentinela `.incubator_reseeded_v2`.
- **Validação Institucional:**
  - Suite de testes completa: **1016 passed** (0 falhas).
  - Raio-X final: **Exit Code 0** (`verify_system_integrity.py`). Broker (2 posições AMZN/AVGO, 4 ordens ativas), BD (2 trades OPEN) e Estado perfeitamente sincronizados.

## Adendo (12/10/2026 20:45) — Incubadora NEXUS v2: Fase P1 Concluída (Commits e0e32eb, 87e3b90, 013ca77, 96fa8fc)
- **Simulador Virtual de Paper Trades (P1.1):**
  - `data/database.py`: adicionada a tabela SQLite `paper_trades` com rastreio de excursões (MAE/MFE), barras mantidas (`holding_bars`), slippage modelado, comissões e motivos de saída (`SL_HIT`, `TP_HIT`, `TIMEOUT`). Métodos adicionados: `insert_paper_trade`, `get_open_paper_trades`, `close_paper_trade`, `update_paper_trade_excursions`.
  - `engine/paper_trade_simulator.py`: motor `PaperTradeSimulator` implementado com modelação de slippage (2 bps), comissão ($2.00 round-trip), regras de saída conservadoras (prioridade para SL caso ambos sejam atingidos na mesma barra) e saída por timeout após 78 barras (~1 sessão diária de 5 min).
  - `core/strategy_evaluator.py`: integrado para abrir paper trades virtuais para sinais de variantes incubadas (`_is_incubator_variant=True`) e estratégias em papel, resolvendo as saídas a cada ciclo a partir do `_data_cache` (custo zero de chamadas API IBKR).
  - `engine/incubator_ranking.py`: `update_rankings` atualizado para extrair estatísticas de performance diretamente de `source='paper_trades'` para as variantes incubadas.
- **Ciclo Evolutivo por Época — `run_epoch` (P1.2):**
  - `engine/strategy_base.py::StrategyRegistry`: adicionado o método `unregister(strategy_id)`.
  - `engine/genetic_engine.py`: adicionado `run_epoch(factory, registry, analytics)` para detetar indivíduos estagnados (`detect_stagnant`), aplicar culling, desregistar do registry em memória, spawnar novos indivíduos mutados/cruzados, instanciar via `StrategyFactory` e registar no registry, avançando a geração.
  - Agendado no fecho diário de mercado em `core/scheduler.py::_daily_close_report`.
- **Governança de Swap & Proteção Canónica (P1.3):**
  - `engine/lifecycle.py::set_stage`: proteção absoluta das estratégias `CANONICAL_LIVE_PERMANENT` contra demoções automáticas disparadas por rotinas internas, exigindo override manual explícito ou `force=True`.
  - `engine/incubator_ranking.py::evaluate_swap`: estratégias canónicas permanentemente excluídas como alvos de demoção. Introduzidos gates quantitativos estritos para promoção de qualquer variante: mínimo de 12 trades virtuais executados, Profit Factor ≥ 1.20, Sharpe Ratio ≥ 0.40, Max Drawdown ≤ 6.0%, vantagem de pontuação de 15% sobre a pior estratégia LIVE não-canónica e estabilidade elegível há pelo menos 5 dias.
- **Verificador de Consistência no Arranque (P1.4):**
  - `engine/incubator_consistency.py::IncubatorConsistencyVerifier`: verificação automática no arranque do `Scheduler`, cruzando o registo de estratégias em memória, o pool genético e o lifecycle manager. Auto-corrige eventuais flags `live=True` indevidas em variantes e audita o estado de `ROT_G12_pmip` (mantida em `PAPER` probatório).
- **Validação Institucional & Deploy (P1.5):**
  - Criadas 4 novas suites de testes unitários: `test_paper_trade_simulator.py`, `test_genetic_epoch.py`, `test_swap_governance.py`, `test_incubator_consistency.py`.
  - Suite de testes completa no servidor Linux: **1030 passed** (0 falhas).
  - Serviço `quant-mentor.service` reiniciado e validado.
  - Raio-X final (`verify_system_integrity.py`): **Exit Code 0** limpo (Broker com 4 posições, 8 ordens, 4 trades OPEN e estado perfeitamente alinhado).


## Adendo (12/10/2026 20:55) — Incubadora NEXUS v2: Fase P2 Concluída (Commit ba54ed8)
- **Score Composto v2 & Rácio de Sortino (P2.1):**
  - `data/database.py::get_strategy_stats`: implementado o cálculo anualizado do rácio de Sortino baseado na volatilidade dos retornos negativos (`downside deviation`), com valor finito de `5.0` quando não há perdas e neutralidade em `0.0`. Chave `sortino_ratio` adicionada ao contrato partilhado `engine/incubator_contract.py`.
  - `engine/incubator_ranking.py`: rebalanceamento dos pesos institucionais do score composto: Sharpe (0.25), Sortino (0.20), Profit Factor (0.20), Win Rate (0.15), Drawdown (0.20), somando rigorosamente 1.00.
  - Implementado desconto estatístico de amostragem (`sample_discount = min(1.0, sqrt(N / 15.0))` para $N < 15$ trades), penalizando variantes com poucos trades de forma a impedir que a pura aleatoriedade atinja o topo do ranking.
- **Filtro Anti-Clone de Genes (P2.2):**
  - `engine/genetic_engine.py`: implementada a distância euclidiana normalizada entre genomas `calculate_gene_distance(base, params_a, params_b)`.
  - Definido `MIN_GENETIC_DISTANCE = 0.15` (distância mínima de 15% em relação a qualquer variante ativa da mesma base).
  - `spawn_replacement`: loop de até 5 tentativas com mutação agressiva e exploração aleatória para rejeitar clones imediatos e garantir diversidade contínua no pool.
- **Gate Walk-Forward / Multi-Regime para Promoção (P2.3):**
  - `engine/incubator_ranking.py::check_regime_robustness`: verificação obrigatória que consulta os trades virtuais na base de dados agrupados por regime de mercado antes de qualquer recomendação de swap.
  - Bloqueia candidatos que apresentem perdas catastróficas num dado regime (WR < 20% com prejuízo líquido para regimes com $\ge 3$ trades) ou que concentrem o seu retorno positivo num regime minoritário quando operam em múltiplos regimes significativos.
  - `core/strategy_evaluator.py`: passagem transparente de `self.ctx.database` para o avaliador de swap.
- **Notificação Enriquecida & Telemetria Torre de Controlo (P2.4):**
  - `core/scheduler.py::_daily_close_report`: alerta Telegram de época enriquecido com a lista de variantes eliminadas, criadas e o top 3 de performers virtuais com WR, PF, Sharpe e Sortino.
  - `api/routes_analytics.py`: adicionado o endpoint autenticado `/api/analytics/incubator/leaderboard` (e `/api/incubator/leaderboard`), expondo sumário da geração, top variantes virtuais, scores normalizados e histórico de swaps para a Torre de Controlo.
- **Validação Institucional & Deploy (P2.5):**
  - Criada nova suite de testes `tests/test_incubator_p2.py` cobrindo Sortino, Score v2, Anti-Clone, Gate Multi-Regime e API.
  - Suite de testes completa no servidor Linux: **1040 passed** (100% de sucesso, 0 falhas).
  - Serviço `quant-mentor.service` ativo e operacional.
  - Raio-X oficial de integridade (`scripts/verify_system_integrity.py`): **Exit Code 0** limpo sem qualquer divergência entre Broker (3 posições NEXUS, 7 LTI ignoradas, 6 ordens ativas), Base de Dados e Estado.
