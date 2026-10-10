# 📊 Blueprint: Separação "Strategy Performance" vs "System Performance"

**Criado em:** 30 de Setembro de 2026  
**Estado:** BLUEPRINT (Aguarda implementação após acumulação de dados)  
**Prioridade:** Alta — Implementar quando atingirmos ~500 sinais registados  
**Ficheiro de implementação futuro:** `core/signal_auditor.py`

---

## 🧠 A Filosofia

Uma das armadilhas mais comuns na quant finance amadora é confundir a qualidade da **estratégia** com a eficiência do **sistema que a executa**. São duas coisas completamente distintas:

| Camada | O Que Mede | Responsável |
| :--- | :--- | :--- |
| **Strategy Performance** | A ideia matemática — se o sinal fosse executado puro, sem fricção, sem filtros, sem comissões | A estratégia em si (MR, GAP, VWAP, etc.) |
| **System Performance** | O que o sistema *realmente* conseguiu capturar da oportunidade — depois do pipeline de risco, do Kelly, dos filtros, do slippage | A infraestrutura NEXUS |
| **Pipeline Efficiency (%)** | A diferença entre ambos — o que a nossa tubagem "comeu" | A governança central, o Sector Guard, o Kelly Sizer |

---

## 💡 O Caso Prático que gerou esta ideia (30/Set/2026)

Hoje, a estratégia `GAP` detetou um sinal perfeito no Micro-Ouro (`MGC`):
- **Entrada:** $4,183.90 SELL
- **Take Profit:** $4,148.70 (+$35 por contrato)
- **Stop Loss:** $4,201.80

O **Incubator** iria eventualmente ver este sinal bloqueado e penalizar a `GAP`. Mas a culpa não era da estratégia — era de um bug de segregação no `vix_guard.py` que colocava os ETFs do LTI no mesmo setor que o Ouro (`[OTHER]`), fazendo crer que o setor estava cheio.

**Sem esta separação de métricas, mataríamos uma estratégia brilhante por razões erradas.**

---

## 📐 Arquitetura do Signal Auditor (Futuro `signal_auditor.py`)

O módulo corre silenciosamente **ao fim do dia** (via `scheduler.py`), sem interferir com o trading.

```
┌─────────────────────────────────────────────────────────┐
│                   SIGNAL AUDITOR                        │
│                                                         │
│  Tabela [signals]  ──►  Backtesting Rápido contra       │
│  (O que a IA quis)       preços históricos do dia       │
│                               │                         │
│                               ▼                         │
│                       Expectancy Bruta                  │
│                       (o que a estratégia valia)        │
│                               │                         │
│                               │                         │
│  Tabela [trades]   ──►  Resultado Real                  │
│  (O que o sistema fez)        │                         │
│                               ▼                         │
│                       Expectancy Realizada              │
│                       (o que o sistema capturou)        │
│                               │                         │
│                               │                         │
│  DIFERENÇA = Pipeline Efficiency (%)                    │
│                               │                         │
│                               ▼                         │
│  Gravado em: [signal_audit_log] na BD                   │
│  Visível em: Torre de Controlo → Tab "Signal Audit"     │
│  Alerta Telegram se Efficiency < 40%                    │
└─────────────────────────────────────────────────────────┘
```

---

## 📊 Métricas a Calcular por Estratégia

Para cada estratégia (ex: `MR`, `GAP`, `VWAP`, `ROT_G12_pmip`…), o auditor calculará:

### A. Expectancy Bruta (Strategy Performance)
> *"Se o sistema executasse 100% dos sinais sem filtros, sem corte de Kelly, sem Sector Guard — quanto ganharia por signal?"*

```
Expectancy_bruta = (Win_Rate_bruta × Avg_Win_bruto) - (Loss_Rate_bruta × Avg_Loss_bruto)
```
- Calculado com backtest rápido: preço de entrada (do sinal) vs preço de TP ou SL (do OHLC do dia)

### B. Expectancy Realizada (System Performance)
> *"Quanto o sistema realmente ganhou, por sinal gerado?"*

```
Expectancy_realizada = PnL_total_fechado / Total_sinais_gerados
```

### C. Pipeline Efficiency
```
Pipeline_Efficiency = Expectancy_realizada / Expectancy_bruta × 100
```

Interpretação:
| Eficiência | Diagnóstico |
| :--- | :--- |
| > 80% | 🟢 Pipeline limpo. Estratégia e sistema alinhados. |
| 40–80% | 🟡 Fricção moderada. Investigar filtros. |
| < 40% | 🔴 ALERTA: Pipeline asfixia a estratégia. Não punir a estratégia — investigar a infraestrutura. |
| < 0% | ☠️ Sistema está a piorar uma estratégia que seria lucrativa. Revisão urgente. |

---

## 🛡️ Regras Sagradas de Implementação

> [!IMPORTANT]
> O Signal Auditor é **read-only**. Nunca interfere, cancela nem modifica trades ativos.

> [!WARNING]
> O Incubator só deve usar as métricas de "Expectancy Realizada" para **penalizar ou promover estratégias DEPOIS de validar que a Pipeline Efficiency > 60%**. Se a Pipeline Efficiency for baixa, o veredicto é suspenso e o alerta é emitido para o humano decidir.

> [!NOTE]
> O backtesting rápido do auditor usa os dados de OHLC já presentes na cache de `ib_connector.py`. Não há chamadas extra à corretora fora de horas de mercado.

---

## 🗓️ Roadmap de Implementação

| Fase | Condição de Entrada | Ação |
| :--- | :--- | :--- |
| **Fase 0 (Agora)** | Imediata | Instalar este Blueprint. Tabela `signals` acumula dados passivamente. |
| **Fase 1** | ~500 sinais registados (≈ 3-4 semanas de Paper) | Implementar `signal_auditor.py` em modo silencioso (só grava, não alerta). |
| **Fase 2** | ~1500 sinais / 2 meses de dados | Ligar alertas Telegram quando Pipeline Efficiency < 40%. |
| **Fase 3** | ~3000 sinais / 4 meses | Integrar no Incubator: bloquear penalização de estratégias quando Efficiency é baixa. |

---

## 🔗 Ficheiros Envolvidos (Futura Implementação)

- **Novo:** `core/signal_auditor.py` — Motor de cálculo
- **Alterado:** `engine/strategy_incubator.py` — Lógica de punição condicionada à eficiência
- **Alterado:** `data/database.py` — Nova tabela `signal_audit_log`
- **Alterado:** `core/scheduler.py` — Agendamento do auditor (EOD, ~21h CET)
- **Alterado:** `torre_controlo/` — Nova tab "Signal Audit" no Dashboard
