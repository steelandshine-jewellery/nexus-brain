# Changelog Nível 12 — Incidente BAC (short órfão), adopt_target e Torre (10 Outubro 2026)

## Contexto
Diagnóstico externo (DeepSeek-R1) apontou "estado crítico". Auditado contra os logs reais: a maior parte era
interpretação errada da arquitetura, mas expôs um bug real de execução. Corrigida também a análise inicial
do assistente (o "cooldown 60s" era backoff interno do Guardian, não spam à IBKR; um `sleep(0.5)` não resolvia nada).

## 1. Incidente BAC — posição short órfã (-32) na conta PAPER
Sequência (09/10/2026, CEST):
- 21:46:28 — `emergency_close` (Hard EOD Exit): `SELL 32 BAC` IOC.
- 21:54:36 — o stop filho da bracket (ID 165946) dispara: `SLD 32 BAC @ 54.26` → posição passa a **-32**.
- 21:55:15 — Guardian coloca stop `BUY 32 @ 55.89` para proteger o short (ID 166139, clientId 2).

**Causa-raiz:** `emergency_close` não cancelava as ordens pendentes do símbolo antes de fechar. O stop sobrevivente
disparou depois e inverteu a posição.

**Correção** (`engine/order_cancellation.py`, commit `9d404ae`):
- `_cancel_symbol_orders_and_wait()` cancela e confirma o cancelamento antes do fecho.
- Fail-closed: sem confirmação de cancelamento NÃO envia o fecho (alerta crítico); ciclo seguinte repete.
- Posição relida após o cancelamento; se já está a 0 não envia ordem.
- Testes: `tests/test_emergency_close_cancels_orders.py` (5; 4 falham contra o código antigo).

## 2. Posições fantasma na Torre ("Descompasso de Registo / UNLINKED")
- Torre lê `live_positions.json`, escrito apenas no ciclo de mercado → fora de horas ficou com ~10,5 h.
- 5 das 6 linhas (AMD, INTC, GS, V, AMZN) não existiam na IBKR; BAC aparecia +32 quando é **-32**.
- **Perigo:** `adopt_target.py` confiava no snapshot e, sem símbolo, inventava um trade (BUY 1 @ 100);
  com trades antigos, ressuscitava o último trade fechado como OPEN (`ROT_G12_pmip`); escrevia o
  `state_store.json` sem atomicidade.

**Correções:**
- `scripts/adopt_target.py` (commit `b43ae95`): valida a posição no broker (read-only, clientId 31), recusa sem posição
  real ou com trade OPEN existente, usa qty/sentido/avgCost reais, cria sempre trade novo, state_store atómico (664).
  Testes: `tests/test_adopt_target.py` (6).
- Torre (commit `4783da4`): `snapshot_age_seconds()` + `POSITIONS_STALE_SECS=900`; API devolve `stale`/`age_seconds`;
  UI mostra banner e troca os botões por "SNAPSHOT ANTIGO". Testes: `tests/test_positions_staleness.py` (4).
- `scripts/liquidate_target.py`: recusa liquidar se existirem ordens pendentes do símbolo (outro clientId não é
  cancelável por este script) — remete para o motor. Testes: `tests/test_liquidate_target_guard.py` (4).

## 3. Estado deixado em aberto (a tratar)
- **BAC short -32** continua aberto na conta paper, protegido pelo stop `BUY 55.89`. Fechar na segunda-feira após 15:45 CEST
  **via motor** (`emergency_close`), NUNCA pelo botão LIQUIDAR.
- **`STREAK HALT`** ativo (4 perdas seguidas). Reativar (`/reset_halt`) na segunda-feira **depois** de o BAC estar flat.
- **`clientId 2 already in use?`** às 23:45 CEST — por investigar (sessão duplicada?).
- **`LiveReadiness 6/8 NOT READY`** — 2 checks por identificar (esperado em paper, não verificado).
- **Segurança:** o remote git do servidor tem um token GitHub em texto claro no URL → revogar e usar deploy key.
- Torre fora de horas não mostra o estado real do broker (só o snapshot); melhoria possível: consulta direta ao broker.

## Validação
Suite QM: 933 passed (922 + 5 + 6) antes do `liquidate_target`; Torre: 4 testes novos.

*Auditado e documentado — Sábado, 10 de Outubro de 2026.*
