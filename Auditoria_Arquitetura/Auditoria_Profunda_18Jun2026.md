# 🏛️ Auditoria Profunda Institucional — NEXUS
## 18 de Junho de 2026

**Âmbito:** Auditoria estática completa a `quant-mentor` e `torre-controlo` na sequência de anomalias persistentes de posições órfãs e perdas de capital na estratégia PAIRS.

---

## 🔴 Bugs Críticos Identificados e Corrigidos

### BUG-01 · Corrupção de Estado por `pytest` em Produção (Severidade: P0)
- **Módulo:** `tests/conftest.py`
- **Sintoma:** Posições ficticias (`SYM`) geradas pelos testes apareciam no reconciliador de produção como posições reais, enquanto as reais eram classificadas como **órfãs**.
- **Causa Raiz:** Durante o deploy, o `pytest` sobrescrevia o ficheiro `/opt/quant_mentor/exit_manager_state.json` de produção com dados de teste.
- **Correção:** Adicionado fixture global em `conftest.py` que redireciona o `ExitManager` para ficheiro temporário (`tmp_path`) durante `pytest`.
- **Git:** `055fcd5`
- **Estado:** ✅ Resolvido e deployado

---

### BUG-02 · `TypeError` Silencioso no `StateStore.record_position()` (Severidade: P0)
- **Módulo:** `engine/reconciler.py` (classe `StateStore`)
- **Sintoma:** A estratégia PAIRS perdia constantemente o rasto de uma das pernas, gerando orfandade permanente e perdas de capital.
- **Causa Raiz:** O método `record_position` não aceitava o parâmetro `strategy_id`, mas o motor PAIRS passava-o sempre. A exceção `TypeError` era silenciada pelo `try/except` envolvente — o estado não era gravado.
- **Correção:** Atualizada a assinatura: `def record_position(self, symbol, qty, avg_cost, strategy_id=None)`.
- **Git:** `d650a1c`
- **Estado:** ✅ Resolvido e deployado

---

### BUG-03 · Alertas de Rollback de Pares Nunca Chegavam ao Telegram (Severidade: P1)
- **Módulo:** `engine/order_engine.py` (método `execute_pair_trade`)
- **Sintoma:** Quando uma das pernas de um par falhava e o rollback era acionado, o alerta crítico ("🚨 Rollback Manual Necessário") nunca chegava ao Telegram do utilizador.
- **Causa Raiz:** A verificação `if hasattr(self, 'alert_callback')` usa o nome sem underscore, mas o atributo de instância é `self._alert_callback`. `hasattr` retornava sempre `False`, silenciando todos os alertas de rollback.
- **Correção:** Substituído por verificação direta: `if self._alert_callback:`.
- **Git:** `147f916` (local) / `6ee83dc` (produção)
- **Estado:** ✅ Resolvido e deployado

---

### BUG-04 · Spam de Avisos VWAP — 408/hora (Severidade: P2)
- **Módulo:** `engine/technical.py`
- **Sintoma:** O `ops-agent` gerava alertas de "High warning rate: 408/hour (threshold: 300)" em loop.
- **Causa Raiz:** O aviso `No DatetimeIndex detected — VWAP cannot reset daily` era emitido em cada chamada de cálculo para cada símbolo.
- **Correção:** Adicionado throttle com `set()` de símbolos já avisados — apenas 1 aviso por símbolo por sessão.
- **Git:** `d15bf1c`
- **Estado:** ✅ Resolvido e deployado

---

### MELHORIA-01 · Identificação de Estratégia nos Alertas Kelly (Severidade: P2)
- **Módulo:** `engine/kelly_sizer.py`
- **Melhoria:** Os logs de Kelly Negativo agora incluem o `strategy_id` para diagnóstico rápido.
- **Git:** `ef7d568`
- **Estado:** ✅ Aplicado

---

## 🟢 Áreas Auditadas Sem Problemas

| Área | Resultado |
|:---|:---:|
| Thread Safety (`threading.Lock` no `OrderEngine`) | ✅ Aprovado |
| GC Task Leak (background tasks asyncio) | ✅ Aprovado |
| Tick Size Rounding (Bracket/TWAP orders) | ✅ Aprovado |
| Network Resilience (timeout 5s + fail-closed) | ✅ Aprovado |
| Circuit Breaker de Reconciliação | ✅ Aprovado |
| TWAP Tranche Accumulation no StateStore | ✅ Aprovado |
| Adopted Orders Reattachment | ✅ Aprovado |
| Safety Manager (VIX/Drawdown/Weekly Guard) | ✅ Aprovado |
| Bracket Order Atomicity (IBKR) | ✅ Aprovado |
| Kelly Sizer Fraction (Quarter-Kelly 0.25) | ✅ Aprovado |

---

## 📦 Estado do Deploy

| Componente | Commit | Estado |
|:---|:---:|:---:|
| `quant-mentor` (GitHub local) | `147f916` | ✅ Sincronizado |
| `quant-mentor` (core-server) | `6ee83dc` | 🟢 Running |
| `torre-controlo` | Sem alterações | 🟢 Running |

---

## 📌 Conclusão

A auditoria revelou **4 bugs** de severidade P0–P2 com impacto direto nas operações de trading. Todos foram corrigidos e deployados. A causa raiz da orfandade persistente foi a combinação do BUG-01 (pytest corrompendo estado) e BUG-02 (TypeError silencioso no StateStore). A causa raiz das perdas PAIRS foi o BUG-02. O sistema está agora 100% estável e com todos os canais de alerta funcionais.
