# ?? Estado Atual de Produ??o (Atualizado a 24/09/2026)

## 1. M?dulos Core e Fases Implementadas (Setembro 2026)
Nas ?ltimas semanas o sistema levou atualiza??es profundas, passando a contar com:
* **Pacing Guard (Anti-Timeouts):** Limite r?gido a 10 ativos por segundo via 
eqHistoricalDataAsync para evitar bloqueios da IBKR (Erro 1100 / 111).
* **LTI Engine:** Independ?ncia total do motor de Day Trading. Snapshot di?rio corrigido (corre??o do date import).
* **Circuit Breaker de Futuros:** Volume m?nimo de Micro Futuros reduzido para 1 para permitir M6E e METALS operarem nas aberturas Europeias e Asi?ticas.

## 2. Matriz de Estrat?gias Ativas (NEXUS)
Atualmente o servidor est? a correr 18 motores de estrat?gia distintos:
- strategy_ai
- strategy_base
- strategy_crt_scalper
- strategy_factory
- strategy_fx_breakout
- strategy_gap
- strategy_generator
- strategy_metals
- strategy_mfi
- strategy_momentum
- strategy_orb
- strategy_pairs
- strategy_rapid_scalper
- strategy_rotation
- strategy_rsi_extremes
- strategy_scalper
- strategy_squeeze
- strategy_vwap

## 3. Seguran?a e Guardian
* **Institutional Fail-Closed:** Se a IB Gateway (Porta 4002) for abaixo, as estrat?gias congelam e n?o duplicam ordens.
* **Position Guardian:** Continua como sentinela ass?ncrono. Esta semana fechou lacunas de posi??es "?rf?s" n?o guardadas em caso de quebra de rede.

---
*Gerado automaticamente pela IA (Antigravity).*

## [30/09/2026] OrderEngine (Cancel-and-Replace OCA)
- **Problema**: A corretora bloqueava modificaÁıes de Trailing Stops em posiÁıes protegidas por Bracket Orders com o Erro 10326.
- **ResoluÁ„o**: Implementado o padr„o Cancel-and-Replace em 	rade_executor.py. A IA agora cancela a ordem antiga e emite uma nova associada a um novo ocaGroup, mantendo a ligaÁ„o ao Take Profit sem violar as regras da IBKR.

## [30/09/2026] VixGuard & Sector Segregation (LTI Leakage Fix)
- **Problema**: O Sector Guard do NEXUS Trading estava a somar cegamente os ativos do cofre LTI (VUAA, 4GLD, BTCE) ao limite do setor [OTHER] (3 posiÁıes m·ximas). Isto fazia com que micro-futuros como o Ouro (MGC) fossem bloqueados por limite de setor cheio, mesmo quando sinais de alta qualidade eram disparados.
- **ResoluÁ„o**: 1. O VixGuard (ix_guard.py) ignora agora por completo qualquer sÌmbolo LTI na contagem setorial da carteira (Regra 3 cumprida na Ìntegra). 2. Micro-futuros foram mapeados devidamente (MGC, SIL para METALS e M6E para FOREX) em sectors.py.

## [30/09/2026] Signal Auditor (Alpha Capture Metric)
- **Problema**: O sistema n„o distinguia entre uma estratÈgia falhar ("Strategy Performance") ou a infraestrutura cortar um trade bom ("System Performance / Pipeline Friction").
- **ResoluÁ„o**: Criado o core/signal_auditor.py. Corre diariamente no fecho para calcular a "Pipeline Efficiency", gravando o ratio entre os sinais que a inteligÍncia gerou e os trades que a governanÁa deixou passar. Base para a prÛxima evoluÁ„o do Incubator.

## [01/10/2026] Position Guardian & Emergency Airbag Fix
- **Problema**: O CancellationManager falhava com AttributeError (falta de atributo store) ao tentar acionar o fecho de p‚nico para posiÁıes Ûrf„s (ex: AMZN, MSFT, GOOGL ontem), desativando a proteÁ„o vital do sistema.
- **ResoluÁ„o**: Corrigido *typo* humano (self.store -> self.engine.store) no order_cancellation.py. O C„o de Guarda (Position Guardian) recuperou a capacidade imediata de liquidar a mercado qualquer posiÁ„o que perca a proteÁ„o da corretora.

## [01/10/2026] OCA Group Linkage Fix (Orphaned SLs)
- **Problema**: ApÛs a primeira modificaÁ„o do Trailing Stop por Cancel-and-Replace, a ordem de Take Profit perdia o atributo parentId. O cÛdigo de atualizaÁıes subsequentes ignorava o Take Profit (porque procurava parentId > 0), o que desmembrava o grupo OCA. Se o TP fosse atingido (ex: GOOGL), o Stop Loss ficava Ûrf„o e aberto no mercado.
- **ResoluÁ„o**: Em 	rade_executor.py, a procura pela perna do TP (LMT) deixou de exigir parentId e passou a basear-se na correspondÍncia matem·tica da *action* (BUY/SELL) da posiÁ„o. O grupo OCA mantÈm-se agora firmemente unificado independentemente de quantas vezes o Trailing Stop se mover.

## [01/10/2026 - 19:55] Transmit Flag Bug in OCA Replacements (Missing SLs)
- **Problema**: Ao modificar o Trailing Stop de uma posiÁ„o que tinha Take Profit, o cÛdigo definia 
ew_sl_order.transmit = False (na suposiÁ„o errada de que o TP a seguir transmitiria ambas). Contudo, em grupos OCA independentes (sem parentId de entrada), a IBKR n„o transmite ordens irm„s por heranÁa! Como consequÍncia, a ordem de Stop Loss ficava retida localmente como n„o-transmitida (desaparecendo da TWS/mercado), e apenas a ordem de Take Profit era enviada.
- **ResoluÁ„o**: Definido 
ew_sl_order.transmit = True incondicionalmente em 	rade_executor.py (idÍntico ao padr„o validado em fechos parciais). Ambas as ordens do grupo OCA s„o agora submetidas e transmitidas de imediato ‡ corretora, garantindo que o Stop Loss NUNCA desaparece ao ser modificado.
- **[2026-10-02] Anti-Fat-Finger e Reconciler Consciente:** Implementada verifica√ß√£o dura na liga√ß√£o (DU para Paper, U para Live) no \ib_connector.py\ para impedir ordens agressivas na conta LTI. O \PositionGuardian\ passou a atirar ordens *Market Exit* imediatamente caso descubra posi√ß√µes √≥rf√£s onde o pre√ßo de mercado j√° furou o SL ou TP original calculado, selando falhas de *gap down* durante *downtime* do servidor.
- **[2026-10-03] Erradica√ß√£o de Cliques de Rato no 2FA:** A configura√ß√£o do TOTP do utilizador \josearruda5\ (Conta LIVE) foi movida com sucesso para o \/opt/ibc/config.ini\ nativo (\TwoFactorAuth.SystemProvider=Google\). O daemon \uto_2fa.py\ foi completamente reescrito para abandonar os cliques vulner√°veis em coordenadas (X=500, Y=372) e injetar os *keystrokes* diretamente no *buffer* do X11, imunizando o cofre LTI contra mudan√ßas de interface gr√°fica na IBKR.

- **[2026-10-03] Expans√£o do Hard EOD Exit e Sess√£o Europeia:** As estrat√©gias de a√ß√µes dos EUA (MR, MOM, PAIRS, SQZ, VWAP) foram convertidas de Swing (pernoite permitido at√© 720 barras) para Intraday Puro, aplicando o *Hard EOD Exit* √†s 21:46 a toda a carteira acionista para evitar gaps noturnos destrutivos. Paralelamente, o scheduler.py foi modificado para acordar √†s 08:00 CET, avaliando exclusivamente as estrat√©gias de Futuros (METALS e M6E) durante o forte fluxo de liquidez da abertura de Londres, transformando efetivamente o NEXUS num motor bidi√°rio (Sess√£o Europeia para Futuros, Sess√£o Americana para A√ß√µes).
