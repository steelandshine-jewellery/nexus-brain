# Auditoria Global NEXUS — Arquitetura Institucional v8.0

> **Data de Auditoria:** Maio de 2026
> **Estado:** PRODUÇÃO (Nível 8 — CentralGovernance Engine)
> **Ecossistema:** Servidor Ubuntu Headless (Mini PC Geekom) — Isolamento Térmico do E-Commerce (S&S)

---

## 1. Visão Global e Filosofia

O **NEXUS** deixou de ser um simples bot de trading para se tornar um **Ecossistema Institucional de Governança Quantitativa**.
O sistema não foca apenas em executar ordens, mas sim na preservação extrema de capital (Risk-First) e na adaptação inteligente aos regimes macroeconómicos.

**Princípios Cardeais:**
1. **Sobrevivência Máxima:** "Proteger o Capital" sobrepõe-se sempre a "Fazer Lucro".
2. **Independência Estatística:** Regras ativas contra correlação de portfólio.
3. **Morte e Renascimento Autónomos:** Capacidade de estancar sangramentos (HALT) e auto-recuperar o estado sem corrupção.
4. **Paper-to-Live Quarentena:** Nenhuma IA toca em dinheiro real sem provas estatísticas empíricas (30 trades).

---

## 2. Infraestrutura e Operações (Ops & DevSecOps)

O NEXUS funciona através de múltiplos níveis de supervisão, garantindo resiliência `24/7/365`.

### 2.1. Git Drift Guard v2.0
- **Função:** Impede que código modificado ou não rastreado localmente entre em produção sem revisão e *commit*.
- **Mecanismo:** Antes do arranque, o script `git_guard.sh` analisa a "Working Tree". Se detetar ficheiros soltos (.bak) ou não commitados, **bloqueia o arranque do serviço**.
- **Impacto:** Força um desenvolvimento limpo, auditorias claras e recuperação imediata caso seja necessário um rollback.

### 2.2. Supervisor (`run_forever.py`)
- O cérebro local que inicia e mantém o processo principal de trading (`main.py` -> `scheduler.py`).
- Gere falhas silenciosas e "Crash Loops".

### 2.3. Watchdog 2.0 (`ops_agent.py`)
- **O Guarda 24/7 independente:** Um processo separado que monitoriza o sistema de fora para dentro.
- Monitoriza o ficheiro `health.json` (o *Heartbeat* do NEXUS).
- Gere "Off-hours" pacientemente sem disparar alarmes falsos quando o gateway está desligado.
- Monitoriza métricas de disco, CPU e integridade da Base de Dados (`quant_platform.db`).

---

## 3. Segurança e Preservação de Capital (Risk Engine)

O pilar central do NEXUS (o `SafetyManager` e associados) opera com **"Fail-Closed"**: na dúvida, bloqueia.

### 3.1. Drawdown Recovery Protocol (Nível 7)
Redução dinâmica e progressiva de tamanho de posição consoante a perda da conta:
- `NORMAL` (<3% DD): 100% Sizing
- `CAUTION` (3% - 5%): 75% Sizing + Alertas
- `REDUCED` (5% - 7%): 50% Sizing
- `DEFENSIVE` (7% - HALT): 25% Sizing + **Apenas estratégias MR/VWAP permitidas**
- `HALT` (>10%): Corte cirúrgico de posições + Bloqueio permanente até reinício manual.

### 3.2. VIX & Macro Overlay
- O regime de volatilidade (VIX) e correlação obrigacionista/ouro (TLT/GLD) injeta um `vix_size_multiplier` no portfólio. Acima de determinados níveis (Regra dos 16), todo o capital alocado no NEXUS é drasticamente reduzido.

### 3.3. Isolation Shield: LTI (Long-Term Investment)
- A conta Interactive Brokers é partilhada. O NEXUS possui um "muro invisível" no `scheduler.py` que ignora ativos cruciais da carteira longa (VWCE, SXR8, 4GLD, etc.), garantindo que os limpa-neves noturnos (Sweepers) não liquidam o portfólio macroeconómico.

---

## 4. Motor de Execução (Trading Engine)

### 4.1. CentralGovernance Engine (Nível 8)
- O **motor central de decisão** único do NEXUS. Todos os módulos de risco são "Sensores" que reportam aqui.
- Devolve sempre um `GovernanceDecision` completo e auditável com todos os fatores registados.
- **FASE 1 — AI Shield (Veto Binário Assimétrico):** A IA apenas pode VETAR. Nunca tem poder de aumentar sizing. Se a API falhar → Fail-Closed (bloqueia trade).
- **FASE 2 — Modo DEFENSIVE:** Se DD > 7%, apenas estratégias `MR/VWAP` são autorizadas. Decisão binária e implacável.
- **FASE 3 — Multiplicadores Compostos:** `composite = VIX × DD × Correlação × ATR`. Todos comprimem. Nenhum pode ampliar.
- **FASE 4 — Qty Final:** `final_qty = max(1, int(base_qty × composite × kelly))`.

### 4.2. Order Validator (Validação Mecânica)
- Responsabilidade **estrita**: verificar IBKR online, spread aceitável, Stop Loss obrigatório, sem posições duplicadas.
- Calcula o `base_qty` pelo risco (1% do capital).
- Delega toda a inteligência de sizing ao `CentralGovernance` numa única chamada auditável.

### 4.3. Approval Level (Matriz de Confiança) — Nível 7 (substituído no 8)
- O score antigo de Approval Level (1/3 AI + 1/3 VIX + 1/3 DD) foi **abandonado**.
- A IA deixou de ter peso no sizing. O composto é agora matematicamente puro: `VIX × DD × Correlação × ATR`.

### 4.4. Correlation Engine & Position Guardian
- O **Guardian** coloca imediatamente ordens Stop Loss de emergência (Limit Orders fora do RTH mitigadas).
- O **Ghost Sweeper** faz conciliação entre a API da corretora e a Base de Dados local. Ordens marcadas com `[GUARDIAN]` são imunes ao Sweeper.

---

## 5. Inteligência Artificial e Sandbox (`StrategyGenerator`)

A evolução natural do Nível 7 trouxe um Laboratório de Pesquisa ao NEXUS.

- **Sandbox e Quarentena:** Integração com Google Gemini (Vertex AI para Prod e API local para Dev). O NEXUS é capaz de pesquisar anomalias macro e propor código Alpha (novas estratégias).
- **Prova de Trabalho:** Nenhuma estratégia criada por IA toca em capital real. Fica retida em modo `PAPER` até fechar no mínimo **30 transações**.
- Se apresentar Win Rate ≥ 52% e Sharpe Ratio ≥ 0.80, a estratégia é marcada como **Elegível para Graduação**.
- O Gestor de Fundos (Tu) aprova a graduação manualmente via Telegram.

---

## 6. Comunicação e Torre de Controlo (Front-End)

### 6.1. Telegram Service (Comando Bidirecional)
O NEXUS não é só um relator passivo. Possui comandos nativos que atuam na hora:
- `/status` (Verifica a saúde do Drawdown Recovery e IBKR)
- `/sandbox` e `/approve_sandbox` (Monitorização e promoção das ML Strategies)
- `/panic` (Liquidate Everything + Halt)
- `/ferias` (Diminuição radical de exposição quando estás fora)
- `/wake` (Wake-on-LAN Magic Packet).

### 6.2. Torre de Controlo
- Dashboard Web em Flask implementado com tipografia institucional e UI em **Glassmorphism**. Monitoriza não só P&L e ativos, mas os limites de drawdown, AI Sentiment, e regimes macro em tempo real. Acesso protegido através de VPN (Tailscale Exit Node).

---

## 7. Diretrizes para Adicionar Código Futuro

1. **Testar Isoladamente:** Todo o novo script Python tem de ser corrido localmente para validar a sintaxe antes do push.
2. **Git Commit & Push OBRIGATÓRIOS:** Se a árvore de ficheiros não estiver alinhada com a master da núvem, o `git_guard.sh` do NEXUS no servidor matará o arranque.
3. **Fail-Closed Principle:** Se uma nova API (ex: nova IA) falhar no Timeout, o Nexus tem de assumir o pior (bloquear trades de alta confiança), e nunca assumir o melhor (Fail-Open) com risco de capital não calculado.
4. **LTI Isolation:** Nunca iterar sobre `ib.positions()` sem filtrar explicitamente o que é estratégia de "Alpha" versus o que é o Cofre intocável.

---

*Fim de Auditoria — Sistema certificado e em produção tática Nível 8.*
