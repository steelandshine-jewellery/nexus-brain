# Changelog Nível 13 — Fortificação de Integridade, Watchdog Autónomo & Remediação de Segurança (10 Outubro 2026)

## 🏛️ Contexto & Postura Operacional
Em cumprimento estrito da Diretiva Constitucional §10 (*"mesmo que pareça bem, investiga sempre; prefiro errar por excesso que por falta"*), realizou-se uma auditoria profunda de causa-raiz que resolveu todas as vulnerabilidades pendentes sem recorrer a "pensos rápidos".

---

## 1. Causa-Raiz das 23:45 e Conflito de Portas Resolvido
- **Diagnóstico Forense**: 
  - O IBC está configurado com `AutoRestartTime=23:45` em `/opt/ibc/config.ini`. O IB Gateway Paper encerra e reinicia automaticamente todos os dias úteis às 23:45:00 CEST.
  - O crontab do utilizador `trader` tinha o `night_sweeper.py` agendado exatamente para o mesmo minuto (`45 23 * * 1-5`).
  - Durante o reinício do Gateway (~30s), o motor `quant-mentor` tentava reconectar-se ao mesmo tempo que o `night_sweeper.py` tentava abrir ligação, gerando `ConnectionRefusedError` e conflitos de sessão (`clientId 2 already in use`).
- **Resolução Arquitetural**:
  - Crontab atualizado: `night_sweeper.py` movido para as `23:55` (`55 23 * * 1-5`), garantindo uma janela limpa de 10 minutos para o IB Gateway reiniciar, autenticar e estabilizar antes de qualquer reconciliação noturna.

---

## 2. Correção Estrutural do `night_sweeper.py`
- **Bugs Identificados**:
  - `DB_PATH` apontava para `/opt/quant_mentor/data/quant_platform.db` (um ficheiro vazio residual de 4KB sem tabelas), lançando silenciosamente `no such table: trades` e assumindo 0 trades protegidos.
  - `IB_PORT` estava hardcoded para fallback `4001` (porta do Gateway LIVE LTI) em vez da porta de trading `4002`.
- **Resolução Implementada**:
  - `DB_PATH` aponta agora estritamente para a base de dados central `/opt/quant_mentor/quant_platform.db` via `config.PROJECT_DIR`.
  - `IB_PORT` obtido dinamicamente de `config.IB_PORT` (ou fallback seguro `4002` Paper).
  - **Princípio Fail-Closed**: Se a base de dados central ou a tabela `trades` não puder ser lida, o script aborta imediatamente com erro crítico (`RuntimeError`/`FileNotFoundError`), impedindo que trades legítimos sejam acidentalmente liquidados como se fossem órfãos.
  - Ficheiro vestigial `/opt/quant_mentor/data/quant_platform.db` arquivado permanentemente.
  - Cobertura com 5 testes unitários dedicados em `tests/test_night_sweeper.py` (todos aprovados).

---

## 3. Vigilância Autónoma e Alertas Proativos (`PositionGuardian` & `safety_watchdog.py`)
- **Limitação Anterior**: O `PositionGuardian` ignorava silenciosamente posições no broker ausentes do `state_store.json` (`continue`). O short BAC permaneceu órfão sem que o utilizador fosse alertado no telemóvel.
- **Implementação no `PositionGuardian` (`engine/position_guardian.py`)**:
  - **Deteção de Órfãs em Tempo Real**: Cada posição ativa no broker fora do LTI é validada contra o `state_store.json` e a base de dados central `quant_platform.db`.
  - **Diagnóstico Diferenciado**:
    - *Dessincronização de Estado*: Se existir trade `OPEN` na BD mas faltar no `state_store`, emite alerta crítico: `🚨 [GUARDIAN CRÍTICO] Dessincronização de Estado Detetada`.
    - *Órfã Total*: Se não constar no `state_store` nem na BD, emite alerta crítico imediato: `🚨 [GUARDIAN CRÍTICO] Posição Órfã Desgovernada no Broker`.
  - **Proteção Anti-Flood**: Cooldown de 15 minutos por símbolo (a menos que a quantidade mude).
  - **Registo de Risco**: Grava evento institucional na tabela `risk_events` da BD.
  - **Deteção de Snapshot Desatualizado**: No arranque de cada scan durante horário de mercado, verifica a idade de `live_positions.json`. Se superior a 15 minutos, emite alerta Telegram e regista `stale_live_positions_snapshot`.
  - Passagem de `db=self.database` no `core/scheduler.py`.
- **Fortificação do `safety_watchdog.py` (`/opt/shared`)**:
  - Corrigido o erro no crontab (`ModuleNotFoundError: No module named 'dotenv'` devido ao uso do python de sistema).
  - Crontab atualizado para usar explicitamente `/opt/quant_mentor/venv/bin/python3`.
  - Adicionada leitura com fallback manual de `.env`, verificação de horário de mercado dos EUA e alerta para snapshot desatualizado (>15m) como vigilante externo de segundo escalão.
  - Repositório `shared-modules` unificado e sincronizado com `main`.

---

## 4. Remediação de Segurança: Plaintext GitHub PATs
- **Problema**: 5 repositórios locais e no servidor continham o token de acesso pessoal do GitHub diretamente no URL do remote (`https://<TOKEN>@github.com/...`), expondo credenciais em logs e comandos `git remote -v`.
- **Solução Institucional**:
  - Configurado o assistente global `git config --global credential.helper store`.
  - Credenciais armazenadas em `~/.git-credentials` com permissões estritas `600` (`-rw-------`).
  - URLs de todos os repositórios (`quant_mentor`, `torre_controlo`, `lti`, `strategy_incubator`, `shared`) alterados para o formato canónico limpo: `https://github.com/steelandshine-jewellery/<repo>.git`.
  - Testes de autenticação e validação do `git_guard.sh` executados com 100% de sucesso.

---

## 5. Validação Forense e Testes
- **Suite Completa de Testes**: **968 testes executados e aprovados** com 0 falhas (`968 passed, 11 warnings in 111.37s`).
- **Verificação de Integridade (§10)**:
  - Executado `/opt/quant_mentor/scripts/verify_system_integrity.py` em produção.
  - Identificou com precisão o estado real: 1 posição no broker (`BAC -32`), 1 ordem de stop ativa (`55.89`), 7 posições LTI blindadas, snapshot de mercado fechado desatualizado.
  - Confirmou conformidade fail-closed sem apaziguamento falso.
- **Serviços Ativos e Saudáveis**:
  - `quant-mentor.service`: Reiniciado com sucesso (`Git guard v2.1: clean and synced with GitHub`).
  - `torre-controlo.service`: Ativo e operacional.

---

## 6. Backup Institucional para Pendrive (`/mnt/usb`)
- Script `run_pen_backup.py` expandido para incluir os novos scripts forenses (`order_cancellation.py`, `verify_system_integrity.py`, `adopt_target.py`, `liquidate_target.py`, `night_sweeper.py`) e ficheiros de estado (`live_positions.json`, `watchdog_state.json`).
- Executado com sucesso: **542.9 MB gravados em `/mnt/usb/Nexus_Backups_2026/20261010_110306`** com manifesto oficial `MANIFESTO.json` e snapshot das 4 bases de dados SQLite.

---

## 7. Roteiro para Segunda-Feira (15:45 CEST)
1. **Posição BAC (-32)**: Encontra-se estável, blindada pelo stop GTC `BUY 32 @ 55.89` na IBKR.
2. Na abertura de mercado (15:45 CEST), fechar a posição via motor de trading com `emergency_close` corrigido (que cancela ordens pendentes antes do fecho).
3. Após BAC flat, reativar o motor através de `/reset_halt`.

*Auditado, testado e registado — Sábado, 10 de Outubro de 2026.*
