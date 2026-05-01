# Changelog — Nível 3 (Fases 1, 2 e 4)

**Data:** 28 de Abril de 2026
**Tags:** #nexus #ecommerce #lti #atualização

## 🎯 Resumo Operacional
Hoje expandimos a infraestrutura de um estado "passivo/estável" para um estado de **Inteligência Predicional e Automação de Nível 3**.

As operações foram executadas respeitando estritamente o princípio de **Isolamento Térmico** e a **Git Guard v2.0**. Tudo está no controlo de versão (branch `main`/`master`) e a correr nos respetivos daemons (`systemd`).

---

## 🛠️ O que foi implementado

### 1. Kelly Position Sizer (NEXUS)
- **Local:** `/opt/quant_mentor/engine/kelly_sizer.py`
- **Descrição:** Implementação de um alocador de capital *Fractional Kelly* (25%). O tamanho das posições já não é fixo, ajustando-se dinamicamente com base no *Win Rate* e *Payoff Ratio* reais do motor.
- **Segurança:** Capado a limites de concentração. Se der erro matemático, recua para multiplicador de `1.0` (Fail-Neutral).

### 2. Handle Guard — Agente #12 (Steel & Shine)
- **Local:** `/opt/steelandshine_hub/agents/handle_guard.py`
- **Descrição:** Agente QA read-only que corre a cada 6 horas. Deteta assimetrias entre os *handles* do catálogo Shopify (criados em PT) e as traduções esperadas em NL/EN nos ficheiros `.liquid` e `.json` do tema.
- **Status:** **ATIVO e em produção**.

### 3. Dynamic Pricing — Agente #13 (Steel & Shine)
- **Local:** `/opt/steelandshine_hub/agents/pricing_agent.py`
- **Descrição:** Sistema de precificação algorítmica. Analisa o *Momentum Score*:
  - Score > 75 (por 3 dias) → Propõe +5%
  - Score < 25 (por 7 dias) → Propõe -10% (Desconto)
- **Segurança:** Fase 1 não altera preços automaticamente na Shopify API. Emite propostas e envia para **Telegram** com botões de *Aprovar/Rejeitar*.

### 4. LTI Drift Rebalancer (NEXUS LTI)
- **Local:** `/opt/lti/engine/rebalancer.py`
- **Descrição:** Motor analítico acoplado ao `lti-engine`. Compara a alocação de mercado real contra os "Pesos Alvo" estipulados no `config.yaml` (VWCE=34%, VUAA=29%, EUNA=15%, etc).
- **Notificação:** Envia um relatório detalhado via Telegram indicando o *drift* (desvio) de cada ETF e sugerindo ações de compra/venda para manter o equilíbrio.
- **Status:** **ATIVO (Modo Relatório apenas)**.

---

## 🔒 Status Git Guard & Daemons
Todos os ecossistemas foram limpos de ficheiros soltos/untracked.

- `quant-mentor` — ✅ Commit `55a9ec2` (Kelly Sizer)
- `steelandshine_hub` — ✅ Commit `eb384e8` (Agentes 12 e 13)
- `lti-portfolio` — ✅ Commit `fcab703` (Drift Report Engine)

Todos os serviços `systemctl` foram reiniciados sem loops de colisão.

---

## 🔜 Próximos Passos Agendados
- **Fim-de-Semana:** Implementar a **Fase 3 (Correlation Engine)** no `order_validator` do Nexus. Agendado para um período *Closed Market* para testes backtest/dry-run seguros, impedindo que o motor fique "cego" durante as sessões de RTH (Regular Trading Hours).
