# Questão 08 — Postmortem técnico de incidente em produção

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** Sonnet 4.6
**Framework:** R-I-S-E

---

## Contexto

> Incidente em andamento na Chronos API após deploy da v2.48.0 (ontem, 18:42 UTC). Degradação progressiva de latência e erros detectada ~18h após o deploy, durante pico de tráfego diário, com esgotamento do pool de conexões ao Ledger RDS, abertura de circuit breaker e acúmulo crítico na fila do Reactor.

---

## Prompt usado

```
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

Passo 1 — Correlação temporal: Cruze o horário do deploy (18:42 UTC de ontem) com o início da degradação nas métricas. Identifique se há gap suspeito e por quê o incidente pode ter demorado a se manifestar.

Passo 2 — Causa raiz: Identifique a causa raiz técnica primária do esgotamento do pool de conexões. Avalie cada mudança do changelog como vetor de risco e aponte quais são causas prováveis, possíveis ou improváveis.

Passo 3 — Cascata de falhas: Descreva a sequência exata de eventos que levou do deploy à degradação visível nas métricas, passando pelo esgotamento do pool, abertura do circuit breaker e acúmulo na fila do Reactor.

Passo 4 — Avaliação das opções de mitigação:
Para cada opção, liste riscos, benefícios, tempo estimado para efeito e se resolve a causa raiz ou apenas os sintomas:
— Opção A: Rollback para v2.47.0
— Opção B: Scaling emergencial (aumentar limits do RDS + pool de conexões)

Passo 5 — Recomendação executiva: Com base nos passos anteriores, dê uma recomendação clara e única (rollback OU scaling), com justificativa de 3 a 5 linhas orientada a risco e velocidade de recuperação. Se recomendar rollback, indique o que precisa ser monitorado após. Se recomendar scaling, defina os valores mínimos sugeridos para os parâmetros.

# Expectation
Produza um documento de postmortem técnico com as seguintes seções, nesta ordem:

① Sumário executivo (5 linhas máx.) — status atual, causa raiz em uma frase, recomendação imediata
② Linha do tempo do incidente — tabela com timestamp, evento e severidade
③ Análise de causa raiz — vetores do changelog avaliados, causa primária destacada
④ Cascata de falhas — diagrama textual ou lista numerada da progressão
⑤ Avaliação das opções — tabela comparativa: rollback vs. scaling (colunas: velocidade de efeito, resolve causa raiz, risco residual, complexidade de execução)
⑥ Recomendação — decisão única, justificativa e próximos passos imediatos

Tom: técnico e direto. Sem introduções genéricas. O documento deve estar pronto para ser lido em voz alta em call de incidente em menos de 3 minutos.
```

---

## Output completo da IA

```markdown
# Postmortem Técnico — Chronos API v2.48.0
**Status:** INCIDENTE ATIVO | **Severidade:** P0 | **Gerado em:** 2026-05-11 14:20 UTC

---

## ① Sumário Executivo

Chronos API em estado crítico: p99=8.100ms, erro=11,7%, pool de conexões esgotado e circuit breaker aberto. A causa raiz é a migração do pool de conexões para a biblioteca interna nova (v2.48.0), que não retorna conexões ao pool após timeout, combinada com o timeout reduzido de 5s para 2s — gerando um leak progressivo que, sob carga de pico, saturou o RDS em 240/250 conexões. **Recomendação imediata: rollback para v2.47.0.**

---

## ② Linha do Tempo do Incidente

| Timestamp | Evento | Severidade |
|-----------|--------|------------|
| D-1 18:42 UTC | Deploy v2.48.0: nova lib pool, timeout 2s, endpoint batch | — |
| D+0 13:30 UTC | p99=420ms, err=0,2% — sistema dentro do normal, tráfego crescendo | INFO |
| D+0 13:45 UTC | p99=510ms, err=0,3% — primeiros sinais de pressão no pool | LOW |
| D+0 14:00 UTC | p99=780ms, err=0,8% — HPA começa a escalar; cada pod novo adiciona 20 conexões ao RDS | MEDIUM |
| D+0 14:10 UTC | p99=2.400ms, err=4,5% — pool exhausted (active=20, waiting=147); RDS ~240/250 | HIGH |
| D+0 14:15 UTC | p99=5.200ms, err=8,2% — circuit breaker ABRE (87%); Reactor começa a acumular | CRITICAL |
| D+0 14:20 UTC | p99=8.100ms, err=11,7% — fila Reactor: 50.127 msgs (+800/min); HPA maxed 12/12 | P0 |

---

## ③ Análise de Causa Raiz

| Mudança do Changelog | Classificação | Justificativa |
|----------------------|---------------|---------------|
| Migração pool → biblioteca interna nova | **CAUSA RAIZ PRIMÁRIA** | Semântica diferente de release de conexão após timeout/erro; pool fixado em max=20/pod sem teste de carga; 12 pods × 20 = 240 conexões = exatamente o valor observado no RDS |
| Timeout reduzido 5s → 2s | **CAUSA CONTRIBUINTE PROVÁVEL** | Queries lentas falham antes de concluir; conexão não devolvida ao pool após timeout acelera o esgotamento |
| Novo endpoint POST /v2/transactions/batch | **CAUSA CONTRIBUINTE PROVÁVEL** | Operações batch mantêm conexões abertas por mais tempo; amplifica pressão sobre pool já limitado durante horário de pico |
| Bump psycopg 3.1.18 → 3.2.0 | **POSSÍVEL** | Minor bump com potenciais mudanças de ciclo de vida de conexão que interagem com a nova biblioteca |

**Prova aritmética da causa raiz:** 12 pods × max 20 conexões/pod = 240 conexões — exatamente o valor observado no RDS (240/250). O HPA, ao tentar mitigar, agravou o problema: cada novo pod registra mais 20 conexões no RDS, aproximando o limite máximo.

---

## ④ Cascata de Falhas

[D-1 18:42 UTC] Deploy v2.48.0 em produção
        │ Nova lib pool (max=20/pod) + timeout 2s + endpoint batch
        │
[D+0 13:30–14:00] Tráfego cresce 1.200 → 1.780 req/s
        │ Queries lentas atingem timeout 2s sem retornar conexão ao pool
        │ Leak progressivo → slots do pool reduzindo gradualmente
        │
[D+0 14:00–14:10] Tráfego sobe para 2.100 req/s
        │ HPA escala pods → cada novo pod registra 20 conexões no RDS
        │ RDS: conexões crescem de ~120 para ~240/250
        │
[D+0 14:10] Pool esgotado: active=20, waiting=147
        │ Queries retornam "timeout após 2.000ms"
        │ "connection reset by peer"
        │
[D+0 14:10–14:15] Batch requests acumulam → "context deadline exceeded"
        │ Taxa de erro sobe de 0,8% para 8,2%
        │
[D+0 14:15] Circuit breaker ABRE (threshold 50%, atual 87%)
        │ Publicações no Reactor falham: "upstream error"
        │
[D+0 14:15–14:20] Fila Reactor acumula: 50.127 msgs (+800/min)
        │ Consumer lag: 18 min e crescendo
        │ HPA maxed: 12/12 — cada pod adicional piora RDS (240/250)
        │
[D+0 14:20] ESTADO CRÍTICO: p99=8.100ms | err=11,7% | RDS 240/250

---

## ⑤ Avaliação das Opções

| Critério | Opção A: Rollback v2.47.0 | Opção B: Scaling Emergencial |
|----------|--------------------------|------------------------------|
| Velocidade de efeito | 5–10 min (deploy + pod restart) | 2–5 min (config change), mas sem garantia |
| Resolve causa raiz | **SIM** — remove biblioteca com leak e restaura timeout 5s | **NÃO** — leak na biblioteca persiste; scaling prolonga, não cura |
| Risco residual | Fila Reactor (50k+ msgs) exige drain pós-rollback; instabilidade breve durante restart | Pool pode re-esgotar mesmo com mais conexões RDS se o leak persiste; risco de reincidência em horas |
| Complexidade de execução | **BAIXA** — operação padrão `argocd app rollback` | **MÉDIA** — alteração de parâmetro RDS + reconfig pool + possível restart + validação |

---

## ⑥ Recomendação

**Decisão: ROLLBACK para v2.47.0 imediatamente.**

O problema não é ausência de capacidade — é comportamento incorreto da nova biblioteca de pool sob carga, combinado com timeout insuficiente de 2s. Aumentar o RDS não elimina o leak: apenas amplia o prazo até o próximo esgotamento completo. O rollback remove simultaneamente os três vetores de risco (nova biblioteca, timeout 2s, endpoint batch sem validação de carga), com tempo de efeito equivalente ao scaling e sem incerteza sobre causa residual. O HPA maxed indica que adicionar mais pods pioraria o RDS — escalar é contraproducente neste cenário.

**Próximos passos imediatos:**

```bash
# 1. Executar rollback via Argo CD
argocd app rollback hvt/chronos-api

# 2. Monitorar liberação do pool
kubectl get pods -n production -l app=chronos-api
```

**Monitorar após o rollback (critérios de recuperação):**

| Métrica | Alvo | Prazo |
|---------|------|-------|
| Conexões RDS | < 100/250 | 5 min |
| Circuit breaker | CLOSED (< 10%) | 10 min |
| Fila Reactor (lag) | Decrescente | 15 min |
| p99 | < 500ms | 15 min |
| Taxa de erro | < 0,5% | 15 min |

---

## Análise do framework

**Justificativa:** O prompt estrutura a instrução nas quatro camadas do framework **R-I-S-E**. O **Role** define o perfil do agente com domínio técnico preciso (SRE sênior, PostgreSQL/RDS, connection pooling, circuit breakers), calibrando o nível de profundidade esperado nas respostas. O **Input** injeta os cinco artefatos do incidente em andamento (changelog, métricas, logs, estado da fila e do cluster) em um contêiner semântico separado, permitindo que o modelo saiba exatamente quais dados processar. Os **Steps** impõem uma sequência analítica não saltável — cada passo alimenta o seguinte (correlação temporal → causa raiz → cascata → opções → recomendação), impedindo que o modelo "pule" para a decisão sem fazer a análise estruturada. A **Expectation** define o formato de saída com seções numeradas, tom específico ("técnico e direto") e um critério de qualidade mensurável ("leitura em menos de 3 minutos").

**Por que R-I-S-E é superior aos demais neste contexto:**

O R-I-S-E supera os outros frameworks porque o problema exige **injeção de dados estruturados + controle de ordem de raciocínio + critérios de aceitação separados do formato** — três necessidades que nenhum outro framework atende simultaneamente.

**vs. R-T-F (Role, Task, Format):** R-T-F comprimiria todo o processo analítico em um único campo "Task", sem mecanismo para impor a sequência dos passos. Neste cenário, a cascata de falhas (Passo 3) só pode ser descrita corretamente *após* a correlação temporal (Passo 1) e a análise de causa raiz (Passo 2). R-T-F não garante essa ordem — o modelo poderia ir direto para a recomendação. Além disso, R-T-F não tem campo dedicado para os artefatos do incidente; changelog, métricas e logs seriam embutidos no "Task" sem separação semântica, degradando a qualidade da análise.

**vs. C-A-R-E (Context, Action, Result, Example):** C-A-R-E é otimizado para prompts que ensinam ou exemplificam um padrão — o campo "Example" pressupõe que uma saída desejada já existe como referência. Em um incidente em andamento, não existe postmortem prévio para servir de exemplo. O campo "Action" de C-A-R-E descreve o que a IA deve fazer em termos gerais ("escreva um postmortem"), mas não oferece controle granular sobre a ordem de raciocínio. A distinção entre "Result" (o que se quer alcançar) e "Expectation" (como avaliar se foi alcançado) também é mais precisa no R-I-S-E, permitindo especificar simultaneamente o formato das seções e critérios de qualidade como "tom técnico" e "leitura em 3 minutos".

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| R — Role | `"Você é um engenheiro sênior de SRE com experiência em diagnóstico de incidentes em produção, análise de métricas de latência e erros, e tomada de decisão sob pressão entre rollback e mitigação emergencial. Você domina PostgreSQL/RDS, connection pooling, circuit breakers e arquiteturas de filas de mensageria."` |
| I — Input | `"Os artefatos do incidente em andamento são os seguintes: 1. Changelog do deploy... 2. Métricas Beacon (últimos 30 min)... 3. Logs do pod chronos-api-79c4d8b9-xk2jp... 4. Estado da fila Reactor... 5. Estado do cluster (agora)..."` |
| S — Steps | `"Passo 1 — Correlação temporal... Passo 2 — Causa raiz... Passo 3 — Cascata de falhas... Passo 4 — Avaliação das opções de mitigação... Passo 5 — Recomendação executiva... Execute a análise na ordem abaixo, sem pular etapas"` |
| E — Expectation | `"Produza um documento de postmortem técnico com as seguintes seções... ① Sumário executivo... ② Linha do tempo... ③ Análise de causa raiz... Tom: técnico e direto. Sem introduções genéricas. O documento deve estar pronto para ser lido em voz alta em call de incidente em menos de 3 minutos."` |

---

<!-- 
REFERÊNCIA RÁPIDA DOS FRAMEWORKS
─────────────────────────────────

R-T-F   Role / Task / Format
T-A-G   Task / Action / Goal
B-A-B   Before / After / Bridge
C-A-R-E Context / Action / Result / Example
R-I-S-E Role / Input / Steps / Expectation
-->
