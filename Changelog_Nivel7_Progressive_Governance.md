# Changelog — Nível 7: NEXUS Intelligence & Progressive Governance

> **Data**: 2026-05-02
> **Commit Range**: `0981838` → `b56bd4d`
> **Sessão**: Brainstorming Arquitetural → Implementação Completa

---

## Contexto

Após estabilização do motor de trading (Nível 6) e a integração do ecossistema S&S,
o Nível 7 foca-se em **governança autónoma inteligente**: o sistema deixa de tomar
decisões binárias (executa / não executa) e passa a operar com uma **matriz de confiança
dinâmica** que dita o tamanho, a estratégia e o comportamento de cada trade.

---

## Filosofia Central (Jim Simons Mode)

> "A diversificação é o único almoço grátis em finanças — mas só funciona
> se as apostas forem estatisticamente independentes."

A Lei dos Grandes Números (que garante que um Win Rate de 60% funciona a longo prazo)
**só é válida se os eventos forem independentes**. Comprar AAPL, MSFT e NVDA com
correlação > 0.85 não são 3 apostas independentes — são **1 aposta gigante** no
setor tecnológico que pode rebentar num único dia.

---

## Mudanças Implementadas

### 1. `engine/order_validator.py` — Progressive Governance

**Problema anterior:** `ValidationResult` era binário (True/False). Não havia gradação.

**Solução:** Introdução do campo `approval_level` (0.0 → 1.0), calculado como
média ponderada de **3 fatores independentes**:

| Fator | Peso | Fonte |
|---|---|---|
| `AI_Confidence` | 1/3 | Benzinga AI News Filter (`get_confidence()`) |
| `VIX_Macro_Multiplier` | 1/3 | SafetyManager (Rule of 16) |
| `DD_Recovery_Multiplier` | 1/3 | SafetyManager (Drawdown Protocol) |

**Comportamento:**
- `approval_level < 0.40` → Trade **bloqueado** completamente
- `0.40 ≤ approval_level < 0.95` → Trade executado com **qty reduzida proporcionalmente**
- `approval_level ≥ 0.95` → Trade com tamanho **máximo permitido**
- Estado `DEFENSIVE` → apenas estratégias MR/VWAP são permitidas (outras bloqueadas)

---

### 2. `engine/correlation_monitor.py` — Gating Ativo Síncrono

**Problema anterior:** O `CorrelationMonitor` era um monitor passivo que enviava
alertas Telegram. Não tinha capacidade de intervir na execução.

**Solução:** Novo método `get_correlation_with_portfolio(symbol: str) → dict`

**Thresholds:**

| Correlação | Ação | Multiplicador |
|---|---|---|
| ≤ 0.70 | `ALLOW` | 1.0 (full size) |
| 0.70 – 0.85 | `SCALE_DOWN` | 0.5 (metade) |
| > 0.85 | `BLOCK` | 0.0 (bloqueado) |

**Características:**
- **Síncrono e em memória** — sem chamadas à API da IBKR no pré-trade
- Usa a matriz calculada pelo scan periódico dos últimos 20 dias
- Fail-open se não houver dados (não bloqueia sem evidência)
- Razão humana legível em cada decisão para auditoria

---

### 3. `engine/safety.py` — Drawdown Recovery Protocol

**Problema anterior:** O `SafetyManager` tinha um HALT binário (tudo ou nada).
Ao atingir `MAX_DRAWDOWN_LIMIT`, o sistema parava abruptamente.

**Solução:** Protocolo de quatro estados com alertas progressivos:

| Estado | Drawdown | Sizing | Comportamento |
|---|---|---|---|
| `NORMAL` | < 3% | 100% | Trading normal |
| `CAUTION` | 3% – 5% | 75% | 🟡 Alerta + sizing reduzido |
| `REDUCED` | 5% – 7% | 50% | 🟠 Alerta urgente + sizing cortado |
| `DEFENSIVE` | 7% – HALT | 25% | 🔴 Apenas MR/VWAP; motor continua |
| `HALT` | ≥ 10% (config) | 0% | Sistema OFF — `/reset_halt` |

**Novos métodos:**
- `get_drawdown_recovery_multiplier() → float`
- `get_drawdown_recovery_state() → str`

---

### 4. `engine/strategy_generator.py` — ML Strategy Lab [NOVO]

Motor de geração autónoma de estratégias via IA.

**Arquitetura:**
- Integração dual: **Vertex AI** (produção, europe-west1, GDPR) + `google-generativeai` (dev)
- Prompt estruturado com contexto real: trades recentes, VIX, regime macro, estratégias existentes
- Parser JSON robusto com validação de campos obrigatórios
- Classe `SandboxedStrategy` para rastreio completo do ciclo de vida

**Regras de Quarentena:**
- **Mínimo 30 trades fechados** antes de qualquer avaliação (significância estatística)
- Win Rate ≥ 52% + Sharpe ≥ 0.8 para elegibilidade de graduação
- Cooldown de 24h entre gerações automáticas
- **Nunca aloca capital real diretamente** — aprovação manual obrigatória

---

### 5. `core/telegram_service.py` — Novos Comandos Nível 7

| Comando | Função |
|---|---|
| `/status` | Agora mostra estado do DD Recovery (🟢/🟡/🟠/🔴) + sizing atual |
| `/sandbox` | Lista todas as estratégias ML em quarentena com métricas |
| `/approve_sandbox AI_GEN_XXX` | Promove uma estratégia ML para capital real (aprovação manual) |

---

## Estado da Governance Matrix após Nível 7

```
Pré-Trade Pipeline (Order Validator):
┌─────────────────────────────────────────────────────┐
│ 1. Stop Loss obrigatório                            │
│ 2. Reconciliação + Conexão                          │
│ 3. Safety HALT check                               │
│ 4. Position exists?                                 │
│ 5. Max concurrent positions                         │
│ 5.5. Sector Guard                                   │
│ 6. Pending orders                                   │
│ 7. Risk per trade (ATR + VIX size)                 │
│ 8. Stop Loss direction                              │
│ 9. Bid-ask spread                                   │
│ 10. Portfolio exposure                              │
│ 11. Concentration single position                   │
│ 12. Correlation Guard (BLOCK >0.85, SCALE >0.70)   │ ← NOVO
│ 13. Benzinga AI News + ApprovalLevel 3-fator       │ ← NOVO
│     └─ AI Confidence + VIX Mult + DD Recovery Mult │ ← NOVO
└─────────────────────────────────────────────────────┘
```

---

## Commits desta sessão

| Hash | Descrição |
|---|---|
| `e181295` | feat(nivel7): Correlation Engine gating, Progressive Governance, StrategyGenerator ML sandbox |
| `b56bd4d` | feat(nivel7): Drawdown Recovery Protocol, ApprovalLevel 3-fator, /sandbox + /approve_sandbox Telegram |

---

## Próximos Passos (Nível 8 — Backlog)

1. **P&L Attribution por Fator** — quantificar a contribuição de cada dimensão de risco
2. **Weekend Analytics Enhancement** — incluir métricas Nível 7 nos relatórios semanais
3. **Torre de Controlo Update** — adicionar painel visual do DD Recovery Protocol e da Sandbox
4. **Primeira geração ML** — quando o Vertex AI estiver configurado, observar a primeira estratégia gerada
