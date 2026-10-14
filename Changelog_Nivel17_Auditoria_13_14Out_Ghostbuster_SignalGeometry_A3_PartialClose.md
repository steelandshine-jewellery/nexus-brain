# Changelog Nível 17 — Auditoria Forense 13–14/10/2026: PAIRS Cadência 79m, Ghostbuster Fill-Guard, Geometria GAP/Paper, Escudo PAIRS (A3) e Guarda de Partial Close (`6d31143`)

**Data:** 13–14/10/2026 · **Commits:** `3d24a98`, `f7cd5a0`, `0d27999`, `933a1cd`, `27eaa38`, `f6abe6e`, `5eaf9b1`, `0704cc2`, `3999ab1`, `6d31143` · **Repo:** `quant-mentor`

---

## 1. Resumo Executivo dos Incidentes e Achados Forenses

1. **Ciclo de 79 Minutos em PAIRS (`ids 750–758`, 13/10/2026):**
   - Entre as `06:02` e as `12:37 UTC` de 13/10, o par `NVDA/AMD` foi submetido a cada ~79 minutos com preços estáticos (`NVDA BUY 5 @ 226.42`, `AMD SELL 2 @ 594.13`) e fechado por `RECONCILER_SYNC` com `pnl=NULL`.
   - No trade `758/759` (`12:37 UTC`), a perna `NVDA` (`+5 @ $224.58` custo médio com comissão; fill a `$224.38`) executou e a perna `AMD` foi cancelada por timeout, tendo o `order_events.py` gravado a etiqueta `REJECTED_BY_BROKER` e o `order_pairs.py` partilhado o mesmo `ocaGroup` nos stops de segurança das duas pernas (o cancelamento da `AMD` cancelou por OCA o stop original da `NVDA`).
   - **Correções (`3d24a98`, `f7cd5a0`, `0d27999`):** Isolamento de `ocaGroup` por perna em `order_pairs.py`, bloqueio de reentrada em par com perna aberta e sincronização de reconciliação.

2. **Origem Provada da Ordem `#4 SELL 5 STP NVDA @ $215.10 (clientId=55)`:**
   - Rastreio forense nos logs da conversa (`transcript.jsonl`, passos `14411–14432`): colocada pelo assistente a **13/10/2026 às 15:55:23 CEST** via `scratch/place_nvda_stop.py` (`clientId=55`), imediatamente após o `"ok"` do utilizador às `15:54:33 CEST`, para repor o stop de segurança da `NVDA` que tinha sido cancelado pelo OCA cruzado da `AMD`.
   - Confirmado que não houve qualquer acesso externo na Tailscale. Documentado em `docs/RUNBOOK_STOP_DE_OUTRO_CLIENTE.md` (`3999ab1`).

3. **Ghostbuster vs Fill em Voo (`UNH` Trade `778`, `#175716`, 14/10/2026):**
   - Às `15:59:07 CEST` de 14/10, o Ghostbuster cancelou o bracket de entrada da `UNH` (`#175713`, `#175714`, `#175715`) 169 ms antes do fill de `SELL 4 UNH @ $391.07`.
   - Às `15:59:16 CEST`, o `order_brackets.py` detetou a ausência de filhos e colocou o stop de emergência retroativo `#175716 BUY 4 STP @ $399.11` (`lmt=399.14`, `tif=GTC`, `client=2`, sem TP e sem `orderRef`).
   - **Correção (`933a1cd`):** Guarda de fill em voo no Ghostbuster (`engine/order_cancellation.py` / `engine/reconciler.py`) e sincronização de `ProbationEngine` no arranque.

4. **AI Learning (`TimeSeriesSplit` Dinâmico + Desduplicação de Clones) e Telemetria (`27eaa38`):**
   - `engine/ai_learning.py`, `engine/strategy_ai.py`, `core/strategy_evaluator.py`: desduplicação de amostras gémeas emitidas por clones genéticos no mesmo ciclo, dimensionamento dinâmico de `TimeSeriesSplit(n_splits, gap)` quando `n < 80` e atualização de `_last_retrain_time` mesmo em skip/erro para impedir retrains repetidos a cada ciclo.
   - `data/ib_connector.py`: flag `_intentional_disconnect` para suprimir falsos `WARNING - IBKR connection lost unexpectedly!` em `SIGTERM`.
   - `tests/conftest.py`: `_enforce_repo_sys_path()` para impedir que `sys.path.insert(0, '/opt/quant_mentor')` em scripts contamine os worktrees isolados `/tmp/nexus_predeploy.*`.

5. **Flag `--orders` no Raio-X Oficial (`f6abe6e`):**
   - `scripts/verify_system_integrity.py`: adicionada flag somente-leitura `--orders` (e `--symbol=XXX`) que lista todas as ordens ativas por símbolo com `orderId`, `action`, `qty`, `orderType`, `status`, `auxPrice`, `lmtPrice`, `tif`, `orderRef`, `ocaGroup`, `parentId`, `clientId`.

6. **Geometria Canónica de Sinais `GAP` e Bloqueio no `PaperTradeSimulator` (`5eaf9b1`):**
   - Descobertos 21 trades fechados em `paper_trades` na família `GAP` (`15` em `GAP`, `3` em `GAP_G37_9ms2`, `3` em `GAP_G38_v8ip`) com `stop_loss >= entry_price` em sinais `BUY` (porque `morning_low - mult*ATR` não garantia `stop < entry` e `validate_rr` usava `abs()`), gerando `+$422.66` artificiais em `SL_HIT` no Paper e readmitindo a `GAP` da quarentena às `15:16:38 UTC` de 14/10.
   - Criado `engine/signal_geometry.py::geometry_error(action, entry_price, stop_loss, take_profit)` como fonte única de verdade, integrada em `GapFadeStrategy.validate_rr`, `PaperTradeSimulator.on_signal` e `OrderValidator`.

7. **Escudo PAIRS Condicional no `PositionGuardian` — `A3` (`0704cc2` & `3999ab1`):**
   - `engine/position_guardian.py`: o escudo `PAIRS` só faz `continue` silencioso se existir stop do próprio motor (`protection_map[sym]['has_sl']` com `sl_qty >= abs(pos_data['qty'])`).
   - Se não houver stop do motor, consulta `_foreign_protective_orders(sym, pos_data['qty'])` via `reqAllOpenOrdersAsync()`:
     - Se existir stop de outro `clientId` (como `#4` de `client=55` na `NVDA`), emite alerta `STOP_DE_OUTRO_CLIENTE` (máx. 1x/30m) e **faz `continue` sem duplicar o stop**.
     - Só se não existir qualquer stop ativo no broker (nem do motor nem de outro `clientId`) é que suspende o escudo e coloca stop de emergência.
   - `scripts/verify_system_integrity.py`: emite `WARNING [STOP_DE_OUTRO_CLIENTE]` quando uma posição só está protegida por ordem de outro `clientId`.

8. **Guarda de `Partial Close` Não Executado em `core/trade_executor.py` (`6d31143`) e Resolução da `GOOGL`:**
   - **Incidente (`20:11:41 CEST`):** No trade `793` (`SELL 10 GOOGL @ $349.44`), o `ExitService` pediu `PARTIAL_TP` de 5 ações (`#177435 BUY 5 LMT @ $349.26 IOC`). Ao fim de 2s, a `#177435` ainda estava `Submitted` com `Filled: 0.0`. O código antigo (`L209-210`) tinha `if filled_qty <= 0: filled_qty = qty`, forçando `filled_qty = 5`, cancelando a `#177435` e os brackets de 10 ações (`#177336`/`#177337`), colocando brackets `DAY` para apenas 5 ações (`#177436`/`#177437`), gravando `+5` (sinal invertido) no `state_store.json` e inserindo o registo fantasma `id=798` na BD.
   - **Correção em 4 Pontos (`core/trade_executor.py`, commit `6d31143`):**
     1. Se `filled_qty <= 0`, aborta imediatamente o partial close, cancela apenas a ordem IOC não executada e chama `exit_manager.reconcile_partial_close_failure(symbol)`.
     2. `_is_resting_bracket_child(o)` exclui o `orderId` da ordem de fecho parcial e qualquer ordem `tif == "IOC"` da leitura de `tp_price`/`sl_price` e do cancelamento de brackets.
     3. Define explicitamente `tif = "GTC"` nas ordens substitutas (`PARTIAL_REPLACE` SL e TP).
     4. Preserva o sinal negativo de `pos.position` (`signed_remaining_qty`) no `state_store` e na BD.
   - **Recuperação e Fecho Real em Lucro da `GOOGL` (`20:58` – `21:02 CEST`):**
     - Às `20:58:14 CEST` (com backup `/opt/quant_mentor/quant_platform.db.bak_20261014_googl798`), foi reposto o bracket `GTC` de 10 ações sob `client=2` (`#177683`/`#177684`), restaurado o `793` para `quantity=10` e anulado o fantasma `798` (`status='CANCELLED'`, `exit_reason='UNFILLED_IOC_PHANTOM_REVERTED'`, `pnl=NULL`).
     - Às `21:01:31 CEST`, a `GOOGL` executou um `PARTIAL_TP` real de 5 ações a `$346.74` (`id=800`, `CLOSED`, **`pnl=+$12.46`**), reduzindo o `793` para 5 ações e colocando `#177746`/`#177747` (`GTC`).
     - Às `21:02:50 CEST`, as 5 ações restantes do `793` atingiram o Take Profit a `$346.57` (`id=793`, `CLOSED`, `WIN (TP hit)`, **`pnl=+$12.82`**), totalizando **`+$25.28` líquidos** e zerando a posição sem ordens órfãs.
     - Às `21:09:41 CEST`, também a `AVGO` (`id=795`) executou um `PARTIAL_TP` real de 5 ações (`id=802`, `pnl=+$5.96`), mantendo `-5` no Broker, `-5` na BD e `-5` no `state_store` com ordens `GTC`.

---

## 2. Validação e Estado Final (`14/10/2026 21:12 CEST`)

- **Portão Oficial (`predeploy_gate.log`):**
  - `2026-10-14 20:50:56 OK head=6d31143 :: 1148 passed, 1 skipped, 13 warnings in 133.27s (0:02:13)`
- **Raio-X (`verify_system_integrity.py --orders`):**
  - `9` posições no Broker (`AAPL -10`, `AVGO -5`, `BAC +91`, `LLY -1`, `MSFT -19`, `NVDA +5`, `UNH -4`, `WMT +45`, `XOM +27`) = `9` trades `OPEN` na BD = `9` posições no `state_store.json`.
  - Zero divergências críticas (`0 CRITICAL`, `0 QTD_DIVERGENTE`, `0 STATE_STORE_DIVERGENTE`).
