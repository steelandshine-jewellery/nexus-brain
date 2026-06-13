# Plano de Implementação — Otimização da Rotação de Logs do Quant Mentor

Este plano detalha a correção para evitar reinícios diários do trading engine à meia-noite causados pela configuração de rotação de logs (`logrotate`) no servidor.

## User Review Required

> [!IMPORTANT]
> A alteração modifica a configuração do `logrotate` do sistema Linux para o Quant Mentor.
> Em vez de enviar o sinal `SIGUSR1` (que termina abruptamente o processo Python do robô) e forçar o systemd a reiniciar o serviço (gerando o alerta `⚠️ MENTOR UNRESPONSIVE`), o sistema passará a usar a diretiva `copytruncate`.
> Isto trunca o ficheiro de log ativo mantendo o processo do robô ativo de forma contínua e sem interrupções.
>
> **Serviço a recarregar:** Não afeta o código do robô diretamente, apenas a configuração do utilitário `logrotate`.

---

## Proposta de Alterações

### Configurações de Sistema

#### [MODIFY] [/etc/logrotate.d/quant-mentor](file:///etc/logrotate.d/quant-mentor)
* **Objetivo:** Adicionar `copytruncate` e remover o script de `postrotate` que envia o sinal `SIGUSR1`.
* **Alteração:**
  Substituir o bloco de configuração:
  ```text
  /opt/quant_mentor/logs/*.log {
      daily
      missingok
      rotate 30
      compress
      delaycompress
      notifempty
      create 0640 trader trader
      sharedscripts
      postrotate
          # Signal the process to reopen logs if needed
          systemctl kill --signal=SIGUSR1 quant-mentor.service 2>/dev/null || true
      endscript
  }
  ```
  Por:
  ```text
  /opt/quant_mentor/logs/*.log {
      daily
      missingok
      rotate 30
      compress
      delaycompress
      copytruncate
      notifempty
      create 0640 trader trader
  }
  ```

---

## Plano de Verificação

### Verificação Manual
1. Testar a sintaxe do logrotate no servidor remoto executando:
   `sudo logrotate -d /etc/logrotate.d/quant-mentor` (modo debug, sem alterar ficheiros).
2. Forçar a rotação manual de forma a garantir que o processo do `quant-mentor` permanece ativo e sem interrupção de logs:
   `sudo logrotate -f /etc/logrotate.d/quant-mentor`.
3. Validar se o `quant-mentor.service` continua a correr de forma ininterrupta e sem gerar logs de restart.
