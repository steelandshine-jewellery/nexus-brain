# 🏛️ Relatório de Auditoria Cognitiva Brutal — Sistema NEXUS
## Análise Sistémica, Concorrência e Integridade Matemática · 13 de Junho de 2026

Efetuámos uma auditoria estrutural e de segurança profunda ao código do motor de trading do **Quant Mentor** (focando nos arquivos `order_engine.py`, `scheduler.py` e `reconciler.py`). O objetivo foi validar a thread safety, race conditions, precisão decimal, resiliência de rede e a integridade da máquina de estados local vs corretora.

---

## 1. Concorrência e Asyncio (Thread Safety & Race Conditions)

### 🔍 Deteção de Race Conditions
*   **Controlo de Duplicação e Cooldowns:**
    O `OrderEngine` utiliza um `self._pending_lock = threading.Lock()` síncrono para cercar a adição de símbolos à lista `self._pending_symbols` em `execute_trade`. Como o método corre na thread principal do loop assíncrono do `asyncio`, operações rápidas de CPU e memória sem `await` dentro do bloco `with` são perfeitamente seguras e não bloqueiam outras tarefas concorrentes. 
*   **Vazamento de Tarefas de Emergência (GC Task Leak):**
    Na rotina de slippage extremo (`_subscribe_trade_events` em `order_engine.py#L580`), o robô agenda o fecho de emergência em background usando `loop.create_task`. Para evitar que o Garbage Collector (GC) do Python destrua a corrotina pendente a meio do processamento (o que geraria o erro `Task was destroyed but pending`), a tarefa é registada num conjunto forte na instância:
    ```python
    engine_ref._background_tasks.add(_task)
    _task.add_done_callback(engine_ref._background_tasks.discard)
    ```
    > [!NOTE]
    > **Status de Auditoria:** **Aprovado.** Esta abordagem de retenção de tarefas assíncronas em background cumpre as melhores práticas de asyncio em Python.

---

## 2. Precisão Matemática (Floating-point & Tick-size Rounding)

### 🔍 Arredondamento do Preço no Mercado de Futuros e Ações
*   **Alinhamento de Tick Size:**
    No envio de ordens Bracket (SL/TP) e tranches TWAP, o `OrderEngine` lê as especificações de `AssetManager` e força o arredondamento para o `tick_size` real do ativo (ex: $0.01 ou ticks específicos de Micro-Futuros):
    ```python
    stop_loss = round(round(stop_loss / tick) * tick, 4)
    ```
    Isso impede de forma estrita que a Interactive Brokers rejeite ordens por conterem frações de preço inválidas no livro de ordens do ativo.

*   **PnL Realizado vs Custo Médio na Base de Dados (SQLite):**
    O cálculo de PnL em `order_engine.py#L735` é feito através de aritmética padrão de float:
    ```python
    _pnl = round((_fill_price - _entry_p) * abs(_qty) * _sign, 2)
    ```
    > [!WARNING]
    > **Nota de Risco:** Embora o arredondamento final para `2` casas decimais via `round()` mascare os desvios infinitesimais da representação binária de float, para fins de contabilidade estrita de saldos (NAV), qualquer desvio de frações de cêntimo pode acumular em alta frequência. 
    > *Recomendação:* Para o futuro, transições matemáticas críticas de comissão/PnL devem ponderar o uso do tipo de dados `Decimal` em vez de `float` puro.

---

## 3. Resiliência de Rede e Exceções

### 🔍 Resiliência a Quedas e Timeouts
*   **Controlo de Timeouts na Qualificação de Contratos:**
    A qualificação de contratos no broker é delimitada por um timeout estrito de 5 segundos:
    ```python
    await asyncio.wait_for(self.connector.ib.qualifyContractsAsync(contract), timeout=5)
    ```
    Se a rede cair a meio, o timeout é apanhado pelo `asyncio.TimeoutError` e falha de forma segura (`fail-closed`), travando a colocação de ordens no mercado.
*   **Circuit Breaker de Reconciliação (QM-V1):**
    No `reconciler.py#L416`, caso o broker reporte que uma posição local aberta está ausente (o que pode ser um bug ou interrupção na API do broker), o reconciliador valida se houve execuções de fecho hoje. Se não existirem execuções de saída, o sistema recusa-se a apagar o estado local do ativo.
    > [!IMPORTANT]
    > **Status de Auditoria:** **Aprovado.** Este circuit-breaker impede a remoção silenciosa de posições na base de dados SQLite durante perdas de pacotes na API da IBKR, blindando os brackets protetores contra cancelamentos inoportunos.

---

## 4. Integridade da Máquina de Estados (State Machine)

### 🔍 Sincronização Progressiva e adopted orders
*   **Acumulação de Volume TWAP no StateStore:**
    Durante a fragmentação de tranches no Micro-TWAP, o `StateStore` atualiza dinamicamente as posições ativas apenas no preenchimento confirmado de cada tranche (`confirm-fill` em `order_engine.py#L616`). O robô recalcula o custo médio ponderado progressivamente à medida que as tranches são executadas:
    ```python
    new_avg_cost = (existing_avg * abs(existing_qty) + fill_price * fill_shares) / abs(new_qty)
    ```
    Isto impede que tranches parciais gerem desalinhamentos de tamanho na base de dados, erradicando o problema do "Monday Orphan Sweep".
*   **Re-conexão de Manipuladores (`adopted_trades`):**
    No arranque, o Scheduler invoca `reattach_handlers_for_adopted_orders` para varrer ordens ativas na corretora que não existiam nesta sessão do processo do robô.
    ```python
    self._subscribe_trade_events(trade, symbol, entry_price=0)
    ```
    Isto garante que ordens "adotadas" do Gateway (após um reinício do robô) continuam a reportar preenchimentos e cancelamentos na base de dados local, eliminando o acúmulo de ordens órfãs pendentes.

---

## 📌 Conclusão da Auditoria
O sistema NEXUS apresenta uma maturidade de engenharia de software excelente na sua camada de robustez e gestão de estados assíncrona. Os patches aplicados recentemente (TWAP progressivo, disjuntor de rede e re-conexão de manipuladores de eventos) resolveram os gargalos sistémicos que provocavam a proliferação de posições órfãs.
O sistema está **100% consistente** (`Reconciliation: ✅ CONSISTENT`) e apto a correr a simulação de forma robusta e defensiva.
