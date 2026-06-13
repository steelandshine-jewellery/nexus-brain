# 🎲 Auditoria de Risco & Implementação da Fase 4 (NEXUS)

Este documento atua como o registo oficial da auditoria cirúrgica e da expansão do ecossistema de risco da plataforma de trading **NEXUS**, implementando a **Fase 4** (Risco, Regimes e Análise Visual).

---

## 🛠️ Registo de Versionamento & Deploy

Em conformidade estrita com o **Git Drift Guard v2.0**, todas as alterações foram devidamente commitadas e os serviços reiniciados a partir de um estado limpo (*clean state*):

### 1. Versionamento (Git)
* **`quant_mentor` (Backend/Risco):** Commit `f66f5d8`
  * Implementação do motor estatístico de Monte Carlo (`monte_carlo.py`), do detetor Gaussian HMM (`regime_hmm.py`) e exposição das APIs no `routes.py`.
* **`torre_controlo` (Frontend/Operações):** Commits `93b431d`, `7627bd0` e `5754c6d`
  * Integração das abas visuais de Risco (Risk Lab), Gráficos (TradingView) e Performance (Analytics) na Torre, com Flask-SocketIO e Yahoo Finance API.

### 2. Deploy (Systemd Services)
Os daemons foram recarregados e reiniciados no `core-server` (Ubuntu remoto):
* **`quant-mentor.service`** 🟢 **Active (running)**
  * Porta: `5000` (API de suporte e simulação).
* **`torre-controlo.service`** 🟢 **Active (running)**
  * Porta: `8000` (Interface de Controlo com HTTP Basic Auth ativa: `arruda`/`nexus2026`).

---

## 🎲 Fase 4: Novas Implementações de Risco e Análise

### A. Motor de Risco Monte Carlo (`monte_carlo.py`)
Desenvolvido um motor avançado de simulação para stress-test do capital da conta (`/opt/quant_mentor/core/monte_carlo.py`):
1. **Geometric Brownian Motion (GBM):** Simulação de trajetórias de retorno contínuas baseadas no drift diário e volatilidade do histórico de trades.
2. **Historical Bootstrap Resampling:** Re-amostragem não-paramétrica dos trades realizados, mantendo a distribuição e curtose reais do histórico de trading (especialmente valioso para estratégias com caudas grossas).
3. **Parametric VaR & CVaR:** Métricas matemáticas de Value at Risk (VaR) e Conditional Value at Risk (CVaR) calculadas a 95% e 99% de confiança.
* **Agendamento Noturno:** Integrado no `scheduler.py` para correr automaticamente às **16:30 ET (22:30h em Amesterdão)**.
* **Notificações Telegram:** O motor dispara um alerta sonoro crítico se a simulação projetar um drawdown potencial (CVaR) superior aos limites definidos.

### B. Detetor de Regimes por Cadeias de Markov (`regime_hmm.py`)
Integração de um modelo estatístico **Gaussian Hidden Markov Model (HMM)** de 4 estados para categorizar dinamicamente o comportamento de volatilidade e retornos do SPY:
* 🟢 **BULL_TREND** (Multiplicador de Alocação: `1.0`)
* 🔴 **BEAR_TREND** (Multiplicador de Alocação: `0.5`)
* 🟡 **HIGH_VOL_RANGE** (Multiplicador de Alocação: `0.7`)
* 🔵 **LOW_VOL_SIDEWAYS** (Multiplicador de Alocação: `0.3`)

> [!NOTE]
> O modelo treinado é persistido de forma serializada em `/opt/shared/hmm_model.pkl`. O motor de ordens (`order_engine.py`) poderá aceder a este ficheiro no futuro para modular o tamanho das posições (*position sizing*) em tempo real.

### C. Abas Adicionadas na Torre de Controlo & Correções
* **Risk Lab (🎲):** Painel interativo para visualizar os drawdowns projetados por Monte Carlo, com controlo manual para correr novas simulações a quente.
* **Charts (📈):** Integração dos gráficos oficiais da **TradingView Lightweight Charts (v4.2)** via CDN, consumindo velas dinâmicas OHLCV da Yahoo Finance.
* **Analytics (📊):** Dashboard analítico avançado contendo Sharpe Ratio (retorno diário ajustado), Profit Factor, Win Rate, Max Drawdown estatístico, Tabela de Performance por estratégia e a Curva de Equity global da conta.
* **Correção Crítica de UX — DOM Placeholder Pattern (🛠️):** Resolvido o congelamento da interface ao tentar maximizar painéis. O ecrã ficava em branco/preto na aba Charts porque o Lightweight Charts induzia Stacking Contexts acelerados por hardware no navegador. A lógica de expansão global foi atualizada para mover temporariamente o card maximizado para o final do `body` do DOM, contornando todas as restrições e garantindo um fullscreen absoluto fluido e o fecho correto em qualquer circunstância.

---

## 🧹 Histórico de Auditorias de Engenharia (Fases 1 a 3)

Para assegurar a imunidade a falhas operacionais em live trading, foram resolvidas as seguintes vulnerabilidades críticas no core do sistema:

1. **look-ahead Bias Eliminado (IA):**
   * Corrigido o treino do modelo preditivo que usava `train_test_split` aleatório (o que permitia à IA treinar com dados do futuro). Substituído por `TimeSeriesSplit` cronológico estrito.
2. **Resiliência da Conexão IBKR:**
   * Centralizadas e sincronizadas as escritas no `mentor.py` usando mecanismos *Thread-Safe* para prevenir que múltiplas threads do scheduler e da API criem *deadlocks* na ligação única ao Gateway do Interactive Brokers.
3. **Imunidade de Armazenamento:**
   * Resolvido um vazamento silencioso de temporários (`.tmp`) gerados pelo `scheduler.py` que estavam em risco de esgotar os *inodes* do sistema de ficheiros do servidor Ubuntu.
4. **Proteção "Zero Trust" Web:**
   * A Torre de Controlo foi blindada com **HTTP Basic Auth** para impedir a exposição pública na rede Tailscale. As escritas na base de dados central SQL foram devidamente parametrizadas contra *SQL Injection*.
5. **Segurança de Posições (IBKR):**
   * O motor de execução obriga agora à especificação de um `stop_loss` superior a `0` para submissão de ordens Bracket, impedindo que ordens órfãs de risco fiquem no mercado sem proteção em caso de queda de ligação.

---
*Documento integrado diretamente no Vault do Obsidian local. Nexus Engine Fase 4 homologado e em execução.*
