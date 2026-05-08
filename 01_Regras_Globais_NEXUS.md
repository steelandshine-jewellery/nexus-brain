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

**ATENÇÃO ABSOLUTA:** É estritamente proibido criar/editar ficheiros localmente fora do fluxo (ex: `/home/arruda/scratch/`) e injetá-los em produção com `sudo cp`. Para garantir a integridade do repositório, utiliza **SEMPRE** o script automatizado na raiz do projeto:
`sudo -u trader ./nexus_deploy.sh "Mensagem da atualização"`

### 4. Isolamento Térmico de Ecossistemas
O Geekom abriga dois mundos que **não se tocam**:
* **NEXUS:** Trading, LTI, Ops (`/opt/quant_mentor/`, utilizador `trader`).
* **Steel & Shine:** Joalharia, Shopify (`/opt/steelandshine_hub/`, utilizador `steelandshine`).
Nunca partilhar permissões, chaves SSH ou cruzar dependências entre as contas.

### 5. Slippage e Segurança Fora de Horas
* **Nunca** submeter `MarketOrder` (ordens a mercado) em guiões de limpeza noturnos ou varreduras (Night Sweepers). Isto causa *slippage* severo na abertura do mercado.
* Operações de manutenção automatizadas usam **SEMPRE** `LimitOrder` focadas no *Mid-Price* e estão restritas a horários seguros (`outsideRth=False`).

---

### 6. Governança Progressiva & Drawdown Recovery (Nível 7)
O sistema não opera de forma binária (Halt/No-Halt). Ele possui um protocolo progressivo de gestão de capital:
* **Drawdown Recovery**: Consoante a saúde da conta, o NEXUS aplica um multiplicador de *sizing* global (100% → 75% → 50% → 25%). Ao atingir estado `DEFENSIVE` (>7% de DD), apenas estratégias `Mean-Reversion` (ex: VWAP) são autorizadas.
* **Approval Level (Matriz de Confiança)**: O tamanho de cada trade resulta de um score (0.0 a 1.0) baseado em 3 pilares vitais com peso igual (33%): (1) Confiança do modelo IA, (2) Multiplicador VIX Macro, e (3) Multiplicador do Drawdown Recovery.
* **Correlation Gating**: É terminantemente proibido o sistema comprar ativos cujos movimentos tenham mais de 85% de correlação com posições já abertas na conta no período de 20 dias (independência estatística rigorosa).

### 7. Sandbox de Inteligência Artificial (`StrategyGenerator`)
Qualquer código Alpha ou estratégia autogerada pelas IAs (Gemini/Vertex) atua num modo restrito e vigiado:
* **Fail-Closed Principle**: Nunca IAs operam "Fail-Open". Qualquer falha de timeout nas APIs de decisão assume imediatamente a interdição do trade de alto-risco.
* **Quarentena Estatística**: Uma estratégia gerada tem de preencher **30 operações em Paper/Simulação** (Significância Estatística). Apenas IAs com >52% de *Win Rate* e *Sharpe* > 0.8 ficam elegíveis.
* **Capital Real só por intervenção Humana**: A promoção de quarentena para a conta real (Interactive Brokers) tem de ser **obrigatoriamente validada de forma manual** pelo Gestor através do comando Telegram (`/approve_sandbox`).

---
*Documento gerido pelo Cérebro Central. Qualquer IA agente a operar no sistema deve ler e respeitar estas diretrizes antes de modificar a infraestrutura.*
