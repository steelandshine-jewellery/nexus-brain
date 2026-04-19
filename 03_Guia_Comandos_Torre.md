# 📡 Guia de Comandos Táticos (Torre de Controlo)

A Torre de Controlo não é um mero painel de leitura; é o comando central interativo para garantir a estabilidade do sistema e intervir em anomalias de mercado.

### 1. Botão de HALT (Kill Switch Global) 🛑
**Onde:** Canto superior direito da Torre, ao lado do indicador de "OK".
**O que faz:**
* Não tenta parar violentamente os serviços via SystemD (o que poderia causar perdas de estado ou transações truncadas na DB).
* Em vez disso, cria instantaneamente um ficheiro-baliza (`/opt/quant_mentor/trading_halt.json`).
* O *Safety Manager* (motor de segurança do Quant Mentor) que monitoriza o sistema intercepta este ficheiro no ciclo seguinte (questão de milissegundos) e **corta totalmente o acesso a novas submissões de ordens à IBKR**. Apenas monitorização continua ativa.

### 2. Live Positions Radar (Hospital de Órfãos) 🏥
**Onde:** Painel central "Overview", imediatamente abaixo dos gráficos Macro.
**O que faz:**
Exibe em tempo real (atualizado a cada 5 minutos pelo `scheduler.py` em ciclo de mercado) a leitura absoluta da base de dados da IBKR, processada via a nossa heurística tripartida:
1. **LTI [Verde]**: Ativos seguros do portfólio de investimento de longo prazo.
2. **NEXUS [Ciano]**: Posições ativas legítimas a ser geridas pelas estratégias táticas.
3. **ORPHANS [Vermelho Piscante]**: Posições abertas que escaparam às bases de dados de rastreio (ex: por erros de API passados).

**Ação de Liquidação (TWAP Limit):**
O botão de liquidação ao lado de um *Orphan* dispara o script independente `/opt/quant_mentor/scripts/liquidate_target.py` em *background*. Ele executa a liquidação cirúrgica (fechando o lado curto ou comprido) focado em **Limit Orders pelo Mid-Price**. 
*Nota*: Nunca usamos Market Orders na Torre, garantindo resiliência contra volatilidade absurda.
