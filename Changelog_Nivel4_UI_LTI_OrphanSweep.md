# Changelog — Nível 4 (UI Glassmorphism + LTI Orphan Sweep)

**Data:** 30 de Abril de 2026
**Tags:** #nexus #ecommerce #ui #lti #auto-liquidation #atualização

## 🎯 Resumo Operacional
Sessão focada em dois eixos paralelos: (1) modernização visual institucional de ambos os dashboards e (2) robustificação do motor de liquidação de posições órfãs do NEXUS, eliminando o risco de scripts soltos não supervisionados.

Toda a execução respeitou estritamente o **Git Drift Guard v2.0** e o isolamento térmico entre os ecossistemas NEXUS e Steel & Shine.

---

## 🛠️ O que foi implementado

### 1. Torre de Controlo NEXUS — Design Glassmorphism v3.1
- **Ficheiros:** `torre.html`, `torre_app.js`, `sw.js`
- **Descrição:** Implementação completa de "Extreme Glassmorphism" na Torre de Controlo:
  - Fundo espacial `#050810` com radial gradient ciano.
  - Painéis de vidro com `backdrop-filter: blur(12px)`.
  - Animações `@keyframes pulse` nos regimes de mercado.
  - Efeito `shimmerSweep` holográfico na secção de AI Intelligence.
  - Hover 3D nos cartões de estratégia.
- **Cache Busting:** Headers `no-cache, no-store` no `torre.py` + `sw.js` atualizado para `nexus-v7`.
- **Versão:** Header atualizado para **Command Center v3.1**.
- **Status:** ✅ Em produção.

### 2. Steel & Shine Command Center — Design Glassmorphism Premium
- **Ficheiro:** `/opt/steelandshine_hub/dashboard/templates/index.html`
- **Descrição:** Redesenho visual completo do painel do E-Commerce:
  - Mesmo sistema de design do NEXUS (fundo `#050810`, glassmorphism).
  - Animações `pulseActive` e `pulsePaused` nos badges dos 11 agentes.
  - `shimmerSweep` holográfico na `insight-box` da AI.
  - Hover responsivo em todos os cartões KPI.
  - Cache anti-regressão aplicado no `server.py` (`make_response` + headers).
- **Status:** ✅ Em produção.

### 3. Auto-Liquidator — Integração Nativa no Scheduler do NEXUS
- **Ficheiro modificado:** `/opt/quant_mentor/core/scheduler.py`
- **Descrição:** Eliminado o risco arquitetural do `auto_liquidator.py` standalone. A lógica foi integrada como job nativo `_monday_orphan_liquidation` no `core/scheduler.py`, com ativação automática às **09:31 ET de cada segunda-feira** (30s após abertura do mercado RTH).

#### Regras de Imunidade Implementadas (Invioláveis)
1. **Imunidade LTI Total:** `VWCE`, `SXR8`, `IMAE`, `EUNA`, `4GLD`, `SPYD` nunca serão tocados.
2. **Proteção de Trades Ativas:** Cruza `ib.positions()` com a tabela `trades WHERE status='OPEN'` da DB — posições com estratégia ativa no Nexus são preservadas.
3. **Escudo Anti-Deadlock (Guardian):** Se o Position Guardian já emitiu uma ordem com `orderRef='GUARDIAN'` para aquela posição, o Orphan Sweep recua e alerta via Telegram. Fim dos Deadlock Loops.
4. **Purga de Margem:** Ordens abertas antigas (sem referência GUARDIAN) são canceladas antes da submissão da Limit Order final, libertando margem.
5. **RTH Exclusivo:** `outsideRth=False` em todas as ordens. Zero exposição ao gap da abertura.

#### Vantagens face ao script standalone anterior
| Aspeto | Antes | Depois |
|---|---|---|
| Supervisão | Nenhuma (script manual) | `systemctl quant-mentor` |
| Sobrevive a reboot | ❌ | ✅ |
| Alertas Telegram | ❌ | ✅ integrado |
| Sessão IB | Nova (clientId=98) | Reutiliza a do Scheduler |
| Visível nos logs | ❌ | ✅ `[ORPHAN-SWEEP]` |

---

## 🔒 Status Git Guard & Daemons

| Ecossistema | Commit | Status |
|---|---|---|
| `quant-mentor` (NEXUS) | `e3d07d3` — Monday Orphan Sweep nativo | ✅ Running |
| `quant-mentor` (NEXUS) | `95b1cd9` — Auto Liquidator LTI-immune | ✅ Histórico |
| `steelandshine_hub` (S&S) | `852a0ee` — UI Glassmorphism Premium | ✅ Running |

---

## 📌 Notas Técnicas
- O script `/opt/quant_mentor/scripts/auto_liquidator.py` permanece no repositório como **backup/referência**, mas não deve ser invocado manualmente. A lógica canónica é agora o método `_monday_orphan_liquidation` do Scheduler.
- O flag `_monday_liquidation_sent` faz reset automático à meia-noite de cada segunda-feira.

---

## 🔜 Próximos Passos
- **LTI Drift Rebalancer:** Ativação em modo de execução (atualmente em modo relatório apenas).
- **Dynamic Pricing Agent:** Transição do modo Human-in-the-Loop para autonomia parcial (aprovação automática em intervalos de confiança elevados).
