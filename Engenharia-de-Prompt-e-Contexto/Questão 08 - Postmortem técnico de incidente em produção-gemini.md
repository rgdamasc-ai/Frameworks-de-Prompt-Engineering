# Questão 08 — Postmortem técnico de incidente em produção (Gemini)

**Data:** 2026-05-16
**Ferramenta usada:** Gemini Code Assist
**Modelo:** Gemini 1.5 Pro
**Framework:** R-I-S-E

---

## Contexto

> Diagnóstico de incidente crítico de latência e erros na Chronos API causado por esgotamento de pool de conexões RDS após deploy de nova biblioteca interna e alteração de timeouts.

---

## Prompt usado

```text
# Role
Você é um engenheiro sênior de SRE com experiência em diagnóstico de incidentes em produção, análise de métricas de latência e erros, e tomada de decisão sob pressão entre rollback e mitigação emergencial. Você domina PostgreSQL/RDS, connection pooling, circuit breakers e arquiteturas de filas de mensageria.

# Input
Os artefatos do incidente em andamento são os seguintes:

1. Changelog do deploy (ontem, 18:42 UTC):
— Deploy chronos-api: v2.47.0 → v2.48.0
— Novo endpoint: POST /v2/transactions/batch
— Refatoração do cliente Ledger: pool de conexões migrado para biblioteca interna nova
— Bump psycopg 3.1.18 → 3.2.0
— Timeout do Ledger reduzido de 5 s para 2 s

2. Métricas Beacon (últimos 30 min):
— 13:30 UTC → p99: 420 ms | req: 1.200/s | err: 0,2%
— 13:45 UTC → p99: 510 ms | req: 1.450/s | err: 0,3%
— 14:00 UTC → p99: 780 ms | req: 1.780/s | err: 0,8%
— 14:10 UTC → p99: 2.400 ms | req: 2.100/s | err: 4,5%
— 14:15 UTC → p99: 5.200 ms | req: 2.400/s | err: 8,2%
— 14:20 UTC → p99: 8.100 ms | req: 2.650/s | err: 11,7%

3. Logs do pod chronos-api-79c4d8b9-xk2jp (14:19–14:19 UTC):
— connection pool exhausted (max=20, active=20, waiting=147)
— Query timeout após 2.000 ms (SELECT em transactions)
— POST /v2/transactions/batch → context deadline exceeded
— connection reset by peer
— Circuit breaker OPEN (threshold 50%, atual 87%)
— Falha de publicação no Reactor: upstream error

4. Estado da fila Reactor (chronos-transactions):
— 50.127 msgs acumuladas, crescendo ~800/min
— Consumer lag: 18 min e aumentando

5. Estado do cluster (agora):
— Chronos: 12/12 pods (HPA no máximo)
— CPU médio: 62% | Memória: 71%
— Conexões ativas ao Ledger RDS: 240/250

# Steps
Execute a análise na ordem abaixo, sem pular etapas:
Passo 1 — Correlação temporal... Passo 2 — Causa raiz... Passo 3 — Cascata de falhas... Passo 4 — Avaliação das opções... Passo 5 — Recomendação executiva...

# Expectation
Produza um documento de postmortem técnico com as seguintes seções...
```

---

## Output completo da IA

### Postmortem Técnico: Incidente Chronos API (v2.48.0)

**① Sumário executivo**
A Chronos API apresenta degradação severa (p99 > 8s) e erro de 11.7% devido ao esgotamento do pool de conexões RDS causado por um leak na nova biblioteca interna de pooling em cenários de timeout agressivo (2s). Recomendação imediata: **Rollback para v2.47.0**.

**② Linha do tempo do incidente**
| Timestamp | Evento | Severidade |
| :--- | :--- | :--- |
| Ontem 18:42 | Deploy v2.48.0 (Nova lib de pool + Timeout 2s) | Baixa (Início) |
| Hoje 13:30 | Início da subida de tráfego orgânico | Baixa |
| Hoje 14:00 | P99 dobra (780ms); HPA escala pods | Média |
| Hoje 14:10 | Esgotamento de pool detectado nos pods; Erros > 4% | Alta |
| Hoje 14:15 | Circuit Breaker OPEN; Fila Reactor inicia acúmulo crítico | Crítica |
| Hoje 14:20 | Saturação de conexões RDS (240/250); Latência em 8.1s | Crítica |

**③ Análise de causa raiz**
*   **Causa Primária:** A nova biblioteca interna de pool de conexões. O gap temporal sugere que o problema se manifesta sob carga, onde o novo timeout reduzido (2s) causa expiração de contextos sem a devida liberação da conexão de volta ao pool ("connection leak").
*   **Vetor de Risco:** 12 pods x 20 conexões = 240 conexões. O fato de o RDS estar com 240 conexões ativas prova que o pool está retendo todas as conexões possíveis, impedindo novas requisições.
*   **Avaliação Changelog:** 
    *   Lib interna de Pool: **Causa Provável** (comportamento de leak).
    *   Timeout 2s: **Causa Provável** (gatilho do leak).
    *   Psycopg 3.2.0: **Improvável** (mudança menor de versão).

**④ Cascata de falhas**
1. Aumento de carga → 2. Queries atingem o novo timeout de 2s → 3. Lib de pool não recicla conexões expiradas → 4. Pool de cada pod esgota (20/20) → 5. HPA escala para 12 pods tentando compensar latência → 6. RDS atinge limite de conexões (240) → 7. Circuit Breaker abre devido à taxa de erro → 8. Backlog na fila Reactor por falha de upstream.

**⑤ Avaliação das opções**
| Opção | Velocidade | Resolve Causa Raiz | Risco Residual | Complexidade |
| :--- | :--- | :--- | :--- | :--- |
| **A: Rollback v2.47.0** | ~5 min | Sim | Baixo (versão estável) | Baixa |
| **B: Scaling RDS/Pool** | ~15 min | Não (Sintomático) | Alto (Leak consumirá novas conns) | Média |

**⑥ Recomendação**
**Decisão: Rollback imediato para v2.47.0.**
O scaling emergencial apenas daria mais "fôlego" para o leak de conexões continuar consumindo recursos do RDS. O rollback é a única forma de restaurar a gestão estável de conexões e o timeout de 5s, estancando a falha em cascata. Após o rollback, monitorar a drenagem da fila Reactor e a queda de conexões ativas no RDS para < 150.

---

## Análise do framework

**Justificativa:** O framework **R-I-S-E** foi o mais adequado para este cenário de incidente pois permitiu separar os dados brutos (Input) da estratégia de resolução (Steps), garantindo que a IA não pulasse direto para a recomendação sem antes validar a cascata de falhas. A definição de **Expectation** garantiu que o output estivesse pronto para consumo imediato em uma "war room". O Gemini tendeu a ser mais analítico na correlação aritmética entre o HPA (12 pods) e o limite do RDS (240/250), identificando o leak de forma determinística, enquanto o Claude focou mais na descrição semântica dos logs em relação ao Claude.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| **R** — Role | `Você é um engenheiro sênior de SRE com experiência em diagnóstico de incidentes...` |
| **I** — Input | `Os artefatos do incidente em andamento são os seguintes: 1. Changelog... 2. Métricas...` |
| **S** — Steps | `Passo 1 — Correlação temporal... Passo 2 — Causa raiz... Passo 5 — Recomendação...` |
| **E** — Expectation | `Produza um documento de postmortem técnico com as seguintes seções... Ton: técnico e direto.` |

---

### Comparação com outros frameworks

1. **R-I-S-E vs R-T-F (Role, Task, Format):** O R-I-S-E é superior aqui porque o R-T-F não separa o "Input" (os logs e métricas) da "Task". Em incidentes de engenharia, a qualidade da análise depende da interpretação correta de dados variáveis fornecidos no input, o que o R-I-S-E isola perfeitamente.
2. **R-I-S-E vs C-A-R-E (Context, Action, Result, Example):** O C-A-R-E foca em resultados passados e exemplos. Para um incidente *em andamento*, precisamos de "Steps" (instruções de raciocínio sequencial) para evitar alucinações de causa raiz, algo que o R-I-S-E fornece e o C-A-R-E não prioriza.
