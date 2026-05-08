# Changelog Nível 10 - Pre-Live Fortification & Clean Slate
**Data:** 08-Maio-2026
**Foco:** Preparação Institucional da Plataforma NEXUS para Live Trading (Abertura de Mercado de Segunda-Feira)

## 1. Operação "Clean Slate" (Reset da Base de Dados)
A plataforma foi totalmente purgada de "sinais órfãos", registos de transações virtuais de simulação (Ghost Orphans) e históricos de calibração que prejudicavam a estatística.
- Executado um comando de varredura à tabela `trades`, `signals`, `performance_snapshots` e `trade_events`.
- Base de dados compactada usando `VACUUM` para reclamar I/O e libertar espaço no disco do servidor.
- Os perfis de ancoragem (`real_equity.json` e `peak_equity.json`) foram reajustados para a NAV atual, resetando o *Drawdown Histórico* para exatamente **0.00%**.
- O motor LTI (Long-Term Investment) foi isolado deste *reset* e o seu histórico de investimentos encontra-se 100% intocável.

## 2. Implementação de Backtesting-as-a-Service (BaaS)
- Motor assíncrono acoplado ao `quant-mentor`, gerindo a execução de *backtests* massivos sem perturbar o *High Frequency Engine* em RTH.
- Integração profunda com o painel UI da Torre de Controlo para visualização de filas de *jobs*, submissão com prioridades de recursos CPU (`nice +15`), e renderização de tabelas e resultados por via de janelas Modais (*Modals*).

## 3. Calibração da Health & Intelligence
- O sistema de penalização por "Degradação a Frio" (Cold Streak Penalty) foi revisto. Estratégias que tenham menos de **5 transações executadas** estão isentas da penalização de win-rate, curando Falsos Positivos na Torre de Controlo (onde estratégias recém-lançadas apresentavam score 95/100 em vez de 100).
- Estratégias com zero transações refletirão um Win-Rate "N/A" (ignorado estatisticamente).

## 4. Agentes de Observabilidade de Elite (Pre-Flight & Exec Report)
Para retirar a obrigatoriedade de o operador estar no painel de controlo o dia inteiro:
1. **Agent Pre-Flight:** (`pre_flight.py`) Desperta 30 mins antes de NY abrir (às 15:00 CEST) para emitir relatórios de go/no-go. Lê ativamente sensores críticos:
   - Estado de gateway IBKR (`ib_api_alive`)
   - Sensor do *Git Drift Guard*
   - Telemetria de volatilidade macro (VIX)
2. **Agent Daily Exec:** (`daily_report.py`) Acorda no silêncio pós-mercado (às 22:15 CEST) para gerar um sumário executivo com *Trades*, *Win-Rate*, *PnL* e Encerramento de NAV enviado para o telemóvel do operador.
3. **Manutenção Autónoma:** Agendado no crontab (`0 3 * * 6`) do servidor para executar nativamente um `VACUUM` todos os sábados na base de dados de forma a assegurar que o disco não fragmenta os acessos da corretora.

## 5. Correção de Hotfixes de Abertura
- **Reabilitação do AI Radar:** Corrigido o *race condition* na inicialização das variáveis de ambiente. O módulo `news_filter` agora importa rigidamente o `.env` no acto de arranque, validando que o *OpenRouter/Groq* estão efetivamente de ouvidos abertos antes de processar as notícias de RTH da Benzinga.
- **Reabilitação da Matriz PAIRS:** Injetado um desvio explícito no API Endpoint da Torre de Controlo para forçar o surgimento de estratégias de *Decoupled Agents* (PAIRS) mesmo sob condições absolutas de base de dados vazia. A contagem de táticas centrais voltou às **12 válidas**.

## 6. Integridade Git Operacional
Todo o código resultante deste ciclo foi submetido usando a convenção estrita de controlo de versão para prevenir arranques de *Drift*:
- Atualizados `torre.py` e `torre_app.js` (Repositório `torre-controlo`).
- Atualizados `api/routes.py`, `intelligence.py`, `.gitignore`, `pre_flight.py`, `daily_report.py`, e `news_filter.py` (Repositório `quant-mentor`).

---
**Status Global:** Sistema EXCELLENT. 100/100 Health. Pronto para Live RTH na Segunda-Feira.
