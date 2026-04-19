# 🌍 Regras de Ouro & Comportamento Global (NEXUS)

Este documento atua como a constituição primordial para o desenvolvimento e operação da infraestrutura.

### 1. Perfil de Operação
* **Localização**: Países Baixos (Timezone: CET/CEST).
* Todas as interações devem adotar postura de engenharia crítica: questionar más decisões arquiteturais, prever falhas de escalabilidade e priorizar segurança e isolamento.

### 2. Ecossistema NEXUS (Trading Platform)
* O Nexus corre num servidor headless Ubuntu (Geekom Mini PC) remoto no IP `192.168.1.131` sob o utilizador `trader`.
* **Banco de Dados Central:** `quant_platform.db`. Nunca apagar tabelas ou realizar *DROPs* de dados críticos (PnL, Risco, Orders) sem validação explícita de backup prévio.
* **Limites Intocáveis:** O motor de risco (`SafetyManager`) e os Limits (Drawdown, Regra 16 VIX, Daily Loss) são a lei suprema. Em caso de HALT do sistema, **jamais** forçar um *override* ou by-pass para testar sem autorização absoluta do operador humano.

### 3. Git Drift Guard v2.0 (Pipeline de Deploy)
A plataforma possui imunidade contra edições diretas "a quente". O daemon `quant-mentor` e `torre-controlo` abortarão o arranque manual se detetarem ficheiros locais `Untracked` ou `Modified`.
**Regra de Ouro do Deploy:**
1. Escrever / Refatorar código.
2. Adicionar ao controlo de versão (`git add`).
3. Comitar (`git commit -m "..."`).
4. Enviar para a nuvem (`git push origin HEAD:main`).
5. Apenas no estado *clean*, executar `sudo systemctl restart`.

### 4. Isolamento Térmico de Ecossistemas
O Geekom abriga dois mundos que **não se tocam**:
* **NEXUS:** Trading, LTI, Ops (`/opt/quant_mentor/`, utilizador `trader`).
* **Steel & Shine:** Joalharia, Shopify (`/opt/steelandshine_hub/`, utilizador `steelandshine`).
Nunca partilhar permissões, chaves SSH ou cruzar dependências entre as contas.

### 5. Slippage e Segurança Fora de Horas
* **Nunca** submeter `MarketOrder` (ordens a mercado) em guiões de limpeza noturnos ou varreduras (Night Sweepers). Isto causa *slippage* severo na abertura do mercado.
* Operações de manutenção automatizadas usam **SEMPRE** `LimitOrder` focadas no *Mid-Price* e estão restritas a horários seguros (`outsideRth=False`).

---
*Documento gerido pelo Cérebro Central. Qualquer IA agente a operar no sistema deve ler e respeitar estas diretrizes antes de modificar a infraestrutura.*
