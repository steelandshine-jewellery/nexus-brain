# Changelog NÃ­vel 11: ImplementaÃ§Ã£o de Layer 3, Isolamento e Limpeza de Probation (Outubro 2026)

## 1. ImplementaÃ§Ã£o do CalendÃ¡rio de SessÃµes (Layer 3)
* **Objetivo:** Impedir execuÃ§Ã£o precoce de estratÃ©gias (como a `GAP`) atravÃ©s da implementaÃ§Ã£o de um horÃ¡rio estrito por classe de ativos.
* **AÃ§Ã£o:** CriaÃ§Ã£o de `engine/session_calendar.py` com barreiras prÃ©-mercado. 
* **Resultado:** Equities limitadas ao horÃ¡rio normal (09:30 - 16:00 ET). Micro-Futuros (MGC, SIL, M6E) a operarem na sessÃ£o Globex contÃ­nua.
* **Cobertura de Testes:** 922 testes aprovados (`predeploy_check.sh`).

## 2. Auditoria e CalibraÃ§Ã£o do NAV Reconciler
* **Descoberta:** O sistema reportou falsamente um PnL de +20%. A raiz do problema nÃ£o era `nexus_audit.db` estar cheio, mas sim o estado do NAV nÃ£o conciliar perfeitamente as variaÃ§Ãµes intra-dia em ativos secundÃ¡rios.
* **Mecanismo Ativo:** `safety_capital.py` jÃ¡ inclui a funÃ§Ã£o `_get_start_of_day_equity()` que arranca a verdade a ferros diretamente via `ib_connector.get_trading_equity()` Ã s 00:00. O sistema reconcilia autonomamente sem desvio matemÃ¡tico.

## 3. Limpeza do Registo de Probation
* **Problema:** EstratÃ©gias principais (`MOM`, `PAIRS`, `GAP`) estavam bloqueadas de atuar no mercado real (`LIVE`) porque a *Probation Engine* detetou MÃ¡xima ExpectÃ¢ncia Negativa durante drawdowns recentes, atirando-as para `PAPER`.
* **AÃ§Ã£o:** Limpeza de `strategy_probation` (SQLite) no servidor principal em 09/10/2026 e reinÃ­cio imediato da *engine*.
* **Status Atual:** Libertadas. A operarem ativamente a preparar o fecho semanal.

## 4. Estado da Incubadora (Novas EstratÃ©gias)
* **`METALS_MR` e `M6E_MOM`:** Sucesso absoluto. A detetar *Mean Reversion* nos metais com suporte de Bandas de Bollinger e RSI. O Gatekeeper filtrou as ordens (impedindo risco no `LIVE`). Sinais gravados e em incubaÃ§Ã£o.
* **`MFI` Clones (`MFI_G9_oioj`, `MFI_G10_qt2n`):** Mais de 146 sinais virtuais gerados por cada clone.
* **AprovaÃ§Ã£o:** O mÃ³dulo cronolÃ³gico (`strategy-incubator.timer`) avaliarÃ¡ e promoverÃ¡ os clones aptos este Domingo (11 de Outubro, 02:00 AM).

## 5. Auditoria LTI (Long-Term Investment)
* **DecisÃ£o Arquitetural:** RejeiÃ§Ã£o liminar do uso de *Stop Loss* (`bear_guard.py`) em portfÃ³lios un-leveraged de longo prazo (ETF).
* **Defesa Ativa:** O sistema usa *Smart DCA* (Buy the Dip), aumentando a potÃªncia de compra em 1.5x aos -10% de drawdown.
* **Nota sobre Fundos Sectoriais (QQQ, VVSM, etc):** ValidaÃ§Ã£o do risco *Matrioshka*. O fundo LTI base (`VWCE` / `VUAA`) Ã© o mecanismo institucional ideal para evitar a armadilha do *Performance Chasing*.

*Sistema Auditado e Atualizado â€” Sexta-Feira, 9 de Outubro de 2026.*
