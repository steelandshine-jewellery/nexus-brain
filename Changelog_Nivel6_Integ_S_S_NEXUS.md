# Nível 6 — Integração Arquitetural Steel & Shine x NEXUS

**Data:** 01 de Maio de 2026
**Status:** ✅ Concluído e em Produção (Lançamento Oficial: Fases 1, 2 e 3)
**Objetivo:** Telemetria Cruzada Avançada (E-Commerce ↔ Trading Institucional) mantendo Isolamento Térmico absoluto entre utilizadores (`steelandshine` vs `trader`).

---

## 🏗️ 1. O Pipeline Macro-Hedge (S&S → NEXUS)
**Objetivo:** O motor de Trading NEXUS passa a utilizar as métricas financeiras reais de economia primária (vendas da loja Steel & Shine) como um indicador avançado (leading indicator) de consumo.

* **S&S Hub:** Implementação do endpoint de telemetria `GET /api/internal/sales_velocity` (porta 8080).
* **NEXUS Safety:** O `SafetyManager` faz *poll* diário à saúde do negócio. Se a Steel & Shine reportar um rácio de vendas em contração (Consumer Contraction), o multiplicador de tamanho do NEXUS baixa automaticamente para `0.75x`, independentemente de outros regimes macro (VIX, GLD, TLT).
* *Status:* Activo no `core-server`. Comunicação estritamente restrita ao localhost, sem exposição de DBs.

## 🤖 2. Due Diligence Agent (Agente Investigador)
**Objetivo:** Eliminar "Flash Crashes Cegos". O sistema não liquida posições a mercado cegamente em caso de um grande "gap-down" sem antes analisar os fundamentais através de Inteligência Artificial Generativa.

* **Implementação (`engine/due_diligence.py`):** Agente autónomo integrado no `PositionGuardian`.
* **Mecânica de Trigger:** Quando o Guardian deteta que um ativo está >5% abaixo do preço de entrada (Gap-down) e encontra-se desprotegido (sem SL/TP):
  1. O Guardian entra num **"HOLD" de 5 minutos**.
  2. O `DueDiligenceAgent` interroga as últimas 5 notícias via `yfinance`.
  3. Envia o prompt estruturado à **Gemini 2.5 Flash** para avaliar se o gap é volatilidade normal ou falência/fraude estrutural.
  4. O resultado da Due Diligence é disparado em tempo real via Telegram.
  5. *Apenas após os 5 minutos de espera* (período de análise/intervenção humana) é que o Stop-Loss de emergência é cravado.

## 💰 3. Dynamic Pricing Baseado em Commodities (NEXUS → S&S)
**Objetivo:** Indexar o preçário dinâmico de alta joalharia aos custos subjacentes reais de insumos em tempo real (ETF proxy Ouro: GLD/4GLD).

* **Torre de Controlo (NEXUS):** Abertura do endpoint `GET /torre/api/macro/commodities` que serve os dados processados dos mercados bolsistas de commodities.
* **Pricing Agent (S&S):** Adição de *job cron* diário (08:30 pré-mercado). O agente lê o ouro do NEXUS. Se detetar uma variação > 3%, notifica instantaneamente o administrador via Telegram.
* **Segurança HITL (Human-in-the-Loop):** Para já, o modelo não altera preços em produção no Shopify de forma autoritária. Atua como co-piloto analítico.

---

## 🛡️ Gestão de Crise & Git Drift Guard
Ao longo desta implementação, as defesas do servidor acionaram por múltiplas vezes, cumprindo na perfeição a sua missão institucional:

1. **Bloqueio de Serviços por Código "Untracked":** O `Git Drift Guard v2.0` vetou repetidamente o arranque do `steelandshine-hub` quando ficheiros Python foram adulterados sem sincronia no master. Solucionado através de commits limpos.
2. **Correção SyntaxError Crítico:** O `pricing_agent` crashou devido a `\n` reais embebidos em `f-strings` no gerador de payloads JSON do Telegram. A reparação foi cirúrgica ao nível dos "raw bytes" em Python para suprimir newlines fantasmas.
3. **Deadlock do Supervisor NEXUS:** Bloqueios detetados pelo Ops Agent resolucionados via deploy Git limpo que acalmou o Heartbeat Timeout de 5 minutos.

Todos os commits foram fundidos (`origin HEAD:main`) nos três grandes repositórios de base:
- `quant-mentor` (NEXUS Engine)
- `torre-controlo` (Dashboard e APIs agregadoras)
- `steelandshine-daemon` (S&S Hub Operacional)

A máquina física remota (IP 192.168.1.131) encontra-se num estado absolutamente sincrónico, perfeitamente isolada ao nível de OS, e 100% monitorizada.
