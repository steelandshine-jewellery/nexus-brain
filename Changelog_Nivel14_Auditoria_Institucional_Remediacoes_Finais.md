# Changelog Nível 14 — Auditoria Institucional 360°, Neutralização de Zombies e Purificação de Logs (10 Outubro 2026)

## 📌 Contexto & Motivação
Em cumprimento rigoroso da diretiva *"mesmo que pareça bem, investiga sempre; prefiro errar por excesso que por falta"* e da Constituição NEXUS (§7, §8, §10), realizou-se uma auditoria institucional de 360° aos 4 pontos de verdade, infraestrutura e processos de sistema. Foram identificadas e remediadas na raiz quatro fragilidades latentes.

---

## 🛠️ Intervenções Realizadas

### 1. Neutralização de Processo Zombie & Alinhamento Canónico de Systemd
- **Diagnóstico:** O processo legado `ib-gateway.service` (PID 1810231) encontrava-se ativo há 28 horas consumindo ~291 MB de RAM e 339 MB de swap. Quatro ficheiros em `/etc/systemd/system/` ainda mantinham referências a esta unidade descontinuada.
- **Ação:**
  - `ib-auto2fa.service`: atualizado para `Requires=ib-gateway-live.service` e `After=ib-gateway-live.service` (conforme Constituição §2, 2FA atua exclusivamente sobre o Display `:98` / conta `josearruda5`).
  - `quant-mentor.service`: atualizado para `Wants=ib-gateway-paper.service` e `After=ib-gateway-paper.service` (Display `:99` / conta `fgpywr755`).
  - `lti-engine.service` & `lti-dashboard.service`: atualizados para `After=ib-gateway-live.service`.
  - O serviço legado `ib-gateway.service` foi parado, desativado e mascarado permanentemente (`systemctl mask`).
- **Resultado:** 300+ MB de swap e 290 MB de RAM libertados; zero conflitos no diretório `/home/trader/Jts`.

### 2. Imposição de Permissões de Estado Canónicas (`0664`) & Single Source of Truth
- **Diagnóstico:** O uso de `tempfile.mkstemp()` por omissão atribui máscara POSIX `0600`. Como a substituição atómica ocorria sem ajuste de permissões, ficheiros de estado como `health.json`, `state_store.json` e `trading_halt.json` ficavam `-rw-------`, em violação da Constituição §6.
- **Ação:**
  - Centralizada a lógica de escrita atómica na função institucional `write_json_atomic` em `engine/safety_io.py`.
  - Refatorados 8 módulos consumidores (`reporting_service.py`, `holiday_mode.py`, `news_filter.py`, `pairs_monitor.py`, `failclosed_metrics.py`, `lti_engine_remote.py`, `regime_hmm.py`, `scheduler.py`).
  - Aplicada a permissão canónica `chmod 664` a todos os ficheiros JSON de estado em `/opt/quant_mentor` e `/opt/lti/state`.
- **Resultado:** Todos os ficheiros persistem de forma atómica, protegidos por file locks, com permissões estritas `0664` e grupo `trader:trader`.

### 3. Purificação de Logs de Produção & Isolamento do Pytest
- **Diagnóstico:** A biblioteca de logging (`core/logger.py`) usava o caminho relativo `logs/`. Durante execuções da suite `pytest`, os testes escreviam logs de teste (ex: `FakeTicker`, injeções propositadas de JSONs corrompidos, acessos mock não autorizados) diretamente em `/opt/quant_mentor/logs/trading_YYYYMMDD.log`. Isso induzia IAs externas (ex: DeepSeek-R1) a diagnosticar falsos colapsos em produção.
- **Ação:**
  - `config.py`: Definido `LOG_DIR = os.path.join(PROJECT_DIR, 'logs')`.
  - `core/logger.py`: Derivação dinâmica de `log_dir` via `config.LOG_DIR`.
  - `tests/conftest.py`: Durante os testes, `config.LOG_DIR` é redirecionado para a directoria temporária `_TEST_STATE_DIR/logs`, que é eliminada no fim da sessão.
- **Resultado:** Suite de 970 testes correu em 111s sem escrever uma única linha no ficheiro de produção `trading_20261010.log`.

### 4. Git Drift Guard — Rotação de Logs em `torre_controlo`
- **Ação:** Adicionada a regra `*.log*` ao `.gitignore` de `torre_controlo` para acomodar logs rotativos (`deploy.log.1`).
- **Resultado:** Zero drift em todos os 5 repositórios (`quant-mentor`, `torre-controlo`, `lti-portfolio`, `strategy-incubator`, `shared-modules`).

---

## 📊 Veredicto da Suite de Testes
- **970 testes aprovados, 0 falhas** (`970 passed, 11 warnings in 111.66s`).
- Todos os serviços ativos, íntegros e operacionais.
