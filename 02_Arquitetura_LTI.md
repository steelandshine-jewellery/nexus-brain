# 🏦 Arquitetura LTI (Long-Term Investment)

O ecossistema NEXUS opera numa única conta partilhada da corretora (Interactive Brokers). Para prevenir a liquidação acidental de ativos detidos a longo prazo, o sistema foi desenhado com uma **Segregação Rigorosa** entre o portfólio de investimento a longo prazo (LTI) e a conta margem para estratégias táticas ativas.

### O Arquivo Sagrado (`config.yaml`)
* Todos os ETFs que fazem parte do núcleo passivo (DCA) da estratégia LTI são definidos no ficheiro absoluto `/opt/lti/config.yaml`.
* **Constituintes Atuais (Exemplos):** `VWCE`, `SXR8`, `IMAE`, `EUNA`, `4GLD`, `SPYD`.

### Regras de Operação de Sistemas Secundários
Qualquer sistema, bot, guiões de reconciliação, Night Sweepers ou o Agente Operacional (Watchdog) que leiam o portfólio completo via `ib.positions()` têm a **obrigação arquitetural** de filtrar ativamente qualquer ativo listado no LTI.

1. **Nunca Liquidar LTI:** Ativos marcados no `config.yaml` jamais devem ser confundidos com "pernas órfãs" (orphan legs).
2. **Monitorização:** A Torre de Controlo reflete o P&L do LTI separadamente na tab "Risk & LTI", garantindo que a performance do motor genético do NEXUS não seja inflacionada/deflacionada pelas flutuações do portfolio de longo prazo.

---

### Orphan Sweep — Liquidação Autónoma de Posições Órfãs
A partir de 30 de Abril de 2026, o motor de liquidação de posições órfãs foi **integrado nativamente no `core/scheduler.py`** como o método `_monday_orphan_liquidation`.

- **Disparo:** Automático às **09:31 ET de cada segunda-feira** (30s após a abertura do mercado RTH).
- **Imunidade LTI:** A lista `LTI_ETFS = {'VWCE', 'SXR8', 'IMAE', 'EUNA', '4GLD', 'SPYD'}` está codificada no próprio método — jamais serão submetidas ordens contra estes ETFs.
- **Proteção Adicional:** Cruza posições com a tabela `trades WHERE status='OPEN'` para preservar estratégias táticas ativas e respeita o `orderRef='GUARDIAN'` do Position Guardian para evitar Deadlock Loops.
- **O script standalone** `/opt/quant_mentor/scripts/auto_liquidator.py` permanece como backup histórico, mas a lógica canónica vive no Scheduler.

---
*Este documento garante que as jóias da coroa nunca são vendidas pelo motor tático de curto-prazo.*
