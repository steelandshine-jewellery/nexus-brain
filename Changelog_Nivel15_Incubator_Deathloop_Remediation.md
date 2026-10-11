# Changelog Nível 15: Remediação do Loop de Morte por Hiper-Filtragem na Incubadora de Estratégias

**Data:** 11 de Outubro de 2026  
**Componentes Afetados:** `strategy-incubator` (`engine/ai_researcher.py`, `engine/scorer.py`, `incubator.py`, `.gitignore`)  
**Commits:** `7f98334`, `2ee2091`  
**Autor/Auditor:** NEXUS Core Team & Antigravity  

---

## 1. Contexto e Diagnóstico Forense
Na execução semanal de 11/10/2026 às 02:00 CEST do `strategy-incubator.service`, 75% dos backtests (18 em 24) terminaram com ZERO trades (em contraste com a média histórica de 10,9%).
Estratégias líquidas com centenas ou milhares de trades perdiam todo o sinal assim que entravam no loop de evolução da IA (`VOL_BRK` 43->31->23->0; `VWAP_DIVERGENCE` 293->0; `VWAP_RSI_REV` 1396->0; `VOL_REV_RSI_VWAP` 1556->0; `TREND_MFI_BB` 0 trades desde a v1).

### Causa-Raiz (Circuito de Realimentação Positiva Destrutivo)
1. **Scorer:** Amostras com `< min_trades (20)` devolviam `score=0.0, sharpe=0.0, win_rate=0.0%`.
2. **AI Researcher:** `evolve_strategy` lia `sharpe=0.0` como `Sharpe too low` e ignorava o défice de trades.
3. **Prompt:** O `EVOLVE_PROMPT` orientava o LLM a apertar filtros se o `Win Rate` fosse baixo. O LLM lia `Win Rate: 0.0%` e adicionava mais filtros cumulativos.
4. **Orquestrador sem Elitismo:** O filho "morto" (0 trades) passava a ser a base da ronda seguinte incondicionalmente, contaminando toda a linhagem.

---

## 2. Remediação Arquitetural Implementada
1. **`engine/scorer.py`**:
   - Amostras insuficientes devolvem breakdown transparente (`error: 'insufficient_trades'`, `total_trades`, `min_trades`, `trade_deficit`) e deixam de fabricar Sharpe ou Win Rate falso de 0.0.
2. **`engine/ai_researcher.py`**:
   - Adicionada função `diagnose(metrics)` com prioridade ao tamanho de amostra (`sample-size-first`).
   - Se `total_trades < min_trades`, aciona o `RELAX_PROMPT` (proíbe novos filtros, exige aumento de frequência de sinal e alívio de limiares).
   - Reordenado `EVOLVE_PROMPT` para validar liquidez antes de Win Rate.
3. **`incubator.py`**:
   - **Elitismo com Rollback:** O progenitor mais líquido é preservado; se um filho morrer (0 trades), reverte para o código elite e notifica o LLM com `prior_evolution_died=True`.
   - **Early-Stop:** 2 filhos mortos consecutivos interrompem a linhagem para poupar chamadas de API e tempo de computação.
4. **Governança Git:**
   - Adicionado `backups/` ao `.gitignore`.
   - Repositório sincronizado em `origin/main` e no workspace local.
   - Teste unitário `/tmp/test_deathloop_fix.py` com 24/24 passed.
   - `nexus_health.py`: **Git Incubator 100% CLEAN** (39 Passed | 0 Errors).
