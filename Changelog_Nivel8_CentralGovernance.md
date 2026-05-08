# Changelog — Nível 8: CentralGovernance Engine

> **Data:** 2 de Maio de 2026
> **Commit:** `6c93c34`
> **Status:** ✅ Em Produção

---

## Objetivo

Centralizar toda a inteligência de decisão de risco do NEXUS num único motor auditável, eliminando a lógica dispersa pelo `OrderValidator`.

---

## Problema Anterior (Nível 7)

O `OrderValidator` acumulava responsabilidades de naturezas distintas:
- **Mecânicas** (conexão IBKR, spread, Stop Loss, duplicados) — OK.
- **Quantitativas** (VIX sizing, DD Recovery, Correlação, AI) — espalhadas em 90+ linhas de código no meio do fluxo de validação.

O score de Approval Level era calculado como média ingénua de 3 fatores (1/3 AI + 1/3 VIX + 1/3 DD), o que dava à IA 33% de poder de sizing — um risco arquitetural.

---

## Arquitetura Nível 8

### Novo Ficheiro: `engine/central_governance.py`

**Motor Central de Decisão** — o único lugar onde o NEXUS decide sizing e vetos.

#### Classes:
- **`GovernanceDecision`** — Dataclass auditável com todos os fatores registados:
  - `approved`, `final_qty`, `reason`
  - `ai_vetoed`, `vix_multiplier`, `dd_multiplier`, `dd_state`
  - `corr_multiplier`, `atr_multiplier`, `composite_multiplier`

- **`CentralGovernance`** — Motor de decisão:
  - **FASE 1: AI Shield (Veto Binário Assimétrico)**
    - A IA apenas pode VETAR (bloquear). Nunca tem poder de aumentar sizing.
    - Se a API da IA falhar → Fail-Closed (bloqueia trade).
  - **FASE 2: Modo DEFENSIVE (DD > 7%)**
    - Apenas estratégias `MR/VWAP/MEAN/REVERSION` são autorizadas.
    - Decisão binária — não há sizing parcial aqui.
  - **FASE 3: Multiplicadores Compostos**
    - `composite = VIX × DD × Correlação × ATR`
    - Todos comprimem. Nenhum pode ampliar o sizing.
    - Kelly Sizer aplicado por último sobre o composto.
  - **FASE 4: Qty Final**
    - `final_qty = max(1, int(base_qty × composite × kelly))`

### Refactoring: `engine/order_validator.py`

- **Removido**: Checks 12 (Correlação dispersa) e 13 (AI + VIX + DD + Approval Level).
- **Adicionado**: Check 12 unificado — chamada única `await self.governance.evaluate_trade(...)`.
- **Injeção via `__init__`**: `self.governance = CentralGovernance(safety, corr, news, kelly)`.
- **Fallback**: Se `CentralGovernance` não estiver disponível (erro de import), aplica apenas VIX×DD como mínimo de segurança.

---

## Separação de Responsabilidades (Definitiva)

| Módulo | Responsabilidade |
|---|---|
| `OrderValidator` | Validação Mecânica: IBKR, spread, SL, duplicados, limites de conta |
| `CentralGovernance` | Inteligência Quantitativa: sizing, vetos, macro, risco |
| `SafetyManager` | Limites de Risco de Sistema: HALT, Drawdown, VIX |
| `CorrelationEngine` | Dados de Correlação estatística (sensor passivo) |
| `NewsFilter (IA)` | Sensor de Eventos Extremos (veto assimétrico apenas) |

---

## Regra Arquitetural Chave

> A IA **não participa** no cálculo do sizing.
> Ela é o **Black Swan Shield**: se detetar um evento extremo, emite um VETO TOTAL.
> Em condições normais, cala-se e deixa os modelos quantitativos trabalharem.
