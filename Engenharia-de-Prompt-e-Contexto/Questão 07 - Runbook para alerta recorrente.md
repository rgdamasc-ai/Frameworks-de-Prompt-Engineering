# Questão 07 — Runbook para alerta recorrente

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** Sonnet 4.6
**Framework:** R-I-S-E

---

## Contexto

> Alerta crítico recorrente (`[CRITICAL] High memory usage on Chronos API pods >85% for 10min`) dispara ~4 vezes por semana no canal `#oncall-chronos`. Sem runbook documentado, o tempo médio de resolução é 30–40 minutos e depende do conhecimento individual do plantonista.

---

## Prompt usado

```
# Role
Você é um engenheiro sênior de confiabilidade de site (SRE) com ampla experiência em ambientes Kubernetes em produção, pipelines GitOps com Argo CD e operações de plantão em sistemas distribuídos. Você escreve documentação operacional clara, objetiva e à prova de ambiguidade, projetada para ser executada por plantonistas sob pressão, sem conhecimento prévio do sistema.

# Input
O alerta recorrente abaixo dispara em média 4 vezes por semana no canal de plantão `#oncall-chronos`:

  `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`

Não existe runbook documentado. O tempo de resolução atual varia de 30 a 40 minutos e depende de quem está no plantão conhecer o sistema. O ambiente é o seguinte:

- **Plataforma**: EKS, namespace `production`
- **Workload**: Chronos API — 6 réplicas, HPA configurado (min 4, max 12, CPU target 70%)
- **Deploy**: Argo CD, repositório `hvt/chronos-api`
- **Dependências diretas**: Ledger (PostgreSQL) e Reactor (filas SQS)
- **Observabilidade**: métricas em `/metrics`, logs centralizados no Beacon, dashboards no Grafana
- **Ferramentas disponíveis no plantão**: `kubectl`, `aws cli`, `argocd cli`
- **Canal de plantão**: `#oncall-chronos` no Slack
- **Escalação sênior**: `@chronos-core` — SLA de 15 min em horário comercial, 30 min fora

# Steps
Para escrever o runbook, siga este processo de construção passo a passo:

1. **Abertura do incidente**: defina as primeiras ações do plantonista ao receber o alerta — postagem de acknowledgement no canal `#oncall-chronos`, registro de horário de início e identificação do alerta.
2. **Diagnóstico inicial — estado dos pods**: liste os comandos `kubectl` exatos para verificar o status dos pods, uso de memória e eventos recentes no namespace `production`. Para cada comando, indique o que o plantonista deve observar na saída e o que é considerado normal versus anormal.
3. **Diagnóstico de dependências**: liste os comandos para verificar a conectividade e a saúde do Ledger (PostgreSQL) e do Reactor (SQS) a partir do contexto do EKS e do `aws cli`. Especifique o resultado esperado de cada verificação.
4. **Verificação do HPA e do Argo CD**: inclua os comandos para inspecionar o estado atual do HPA e o status do sync no Argo CD. Indique os sinais que sugerem que o escalonamento automático está agindo corretamente ou está travado.
5. **Ações de mitigação**: descreva as ações ordenadas que o plantonista pode executar de forma autônoma (ex.: reiniciar pods com problema, forçar sync no Argo CD, ajustar réplicas manualmente), com o comando exato para cada ação e a verificação de que a ação funcionou.
6. **Critérios objetivos de escalação**: defina condições mensuráveis e sem ambiguidade que obrigam o plantonista a acionar `@chronos-core` — use limiares numéricos, contagens de eventos ou estados observáveis, não julgamentos subjetivos.
7. **Critério de encerramento do incidente**: defina a condição exata (métrica, observação ou ausência de alerta por N minutos) que confirma a resolução, e o texto-padrão de postagem de encerramento no canal.
8. Com a saída (output) deste prompt, use o "Modelo.md" que esta no projeto "Frameworks-de-Prompt-Engineering", na pasta "Engenharia-de-Prompt-e-Contexto" para prencher os itens e adicione no arquivo "Questão 07 - Runbook para alerta recorrente.md"
9. Solicite aprovação antes de executar.

Escreva o runbook em português, em linguagem imperativa direta ("execute", "verifique", "poste"). Use seções numeradas com títulos em negrito. Inclua os comandos em blocos de código. Não inclua explicações teóricas ou contexto de arquitetura além do necessário para executar cada passo.

# Expectation
O entregável é um runbook operacional completo, pronto para uso imediato, que atenda a todos estes critérios:

- **Autonomia total**: um plantonista sem conhecimento prévio do Chronos API consegue executar o runbook de ponta a ponta sem precisar perguntar a ninguém.
- **Comandos 100% executáveis**: todos os comandos devem ser copiáveis e executáveis sem adaptação — nenhum placeholder genérico como ``; use a sintaxe de seleção dinâmica (`kubectl get pods -n production -l app=chronos-api`) sempre que o valor não for estático.
- **Verificação explícita em cada passo**: cada ação termina com uma linha "✅ Esperado:" descrevendo o que confirma que o passo foi bem-sucedido, e uma linha "⚠️ Se não:" descrevendo o que fazer caso o resultado seja diferente.
- **Critérios de escalação sem subjetividade**: as condições para acionar `@chronos-core` são baseadas em valores observáveis (ex.: "memória ainda acima de 85% após 15 minutos de mitigação"), não em julgamentos como "se parecer grave".
- **Critério de encerramento claro**: inclui o texto exato para postar no `#oncall-chronos` ao fechar o incidente.
- **Extensão adequada**: entre 600 e 1000 palavras — denso o suficiente para ser completo, enxuto o suficiente para ser lido sob pressão.
- **Formato Markdown**: título H1, seções H2, subseções H3 quando necessário, blocos de código para todos os comandos.
```

---

## Output completo da IA

```markdown
# Runbook: [CRITICAL] High Memory Usage — Chronos API Pods

**Alerta:** `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`
**Namespace:** `production` | **Canal:** `#oncall-chronos` | **Escalação:** `@chronos-core`

---

## 1. Abertura do Incidente

Poste imediatamente no `#oncall-chronos`:

[ACK] Alerta: High memory usage on Chronos API pods. Início: $(date -u '+%Y-%m-%d %H:%M UTC'). Investigando: @<seu-usuário>

Registre o horário de início:

```bash
echo "Início do incidente: $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
```

✅ **Esperado:** mensagem postada no canal e horário anotado localmente.
⚠️ **Se não:** poste mesmo sem confirmação do sistema e siga para o passo 2.

---

## 2. Diagnóstico Inicial — Estado dos Pods

### 2.1 Listar pods e status

```bash
kubectl get pods -n production -l app=chronos-api -o wide
```

✅ **Esperado:** 4–12 pods com status `Running`.
⚠️ **Se não:** pods com `OOMKilled`, `CrashLoopBackOff` ou `Pending` — anote-os e avance para a Seção 5.

### 2.2 Verificar uso de memória

```bash
kubectl top pods -n production -l app=chronos-api --sort-by=memory
```

✅ **Esperado:** uso de memória abaixo de 85% do limite em todos os pods.
⚠️ **Se não:** identifique os 3 pods com maior consumo — candidatos a reinicialização na Seção 5.1.

### 2.3 Verificar eventos recentes

```bash
kubectl get events -n production --sort-by='.lastTimestamp' | grep -i "chronos-api\|OOMKill\|memory"
```

✅ **Esperado:** sem eventos `OOMKilled` ou `BackOff` na última hora.
⚠️ **Se não:** conte os eventos `OOMKilled` — se ≥ 3 em 30 minutos, acione `@chronos-core` (Seção 6).

### 2.4 Inspecionar logs dos pods

```bash
kubectl logs -n production -l app=chronos-api --tail=100 --max-log-requests=12 | grep -i "error\|warn\|memory\|heap\|oom"
```

✅ **Esperado:** sem mensagens críticas de memória, heap ou conexões.
⚠️ **Se não:** copie as mensagens de erro para incluir no acionamento de `@chronos-core`.

---

## 3. Diagnóstico de Dependências

### 3.1 Ledger (PostgreSQL)

```bash
kubectl run -it --rm debug-ledger --image=postgres:15-alpine --restart=Never -n production -- \
  psql -h ledger.production.svc.cluster.local -U chronos -c "SELECT 1;" 2>&1 | head -5
```

✅ **Esperado:** retorno `1 row` em menos de 2 segundos.
⚠️ **Se não:** timeout ou recusa de conexão — acione `@chronos-core` imediatamente (Seção 6).

### 3.2 Reactor (SQS)

```bash
aws sqs get-queue-attributes \
  --queue-url $(aws sqs get-queue-url --queue-name chronos-api-tasks --query 'QueueUrl' --output text) \
  --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible \
  --output table
```

✅ **Esperado:** `ApproximateNumberOfMessages` < 10.000.
⚠️ **Se não:** valor ≥ 50.000 é critério de escalação automática — registre e avance para Seção 6.

---

## 4. Verificação do HPA e do Argo CD

### 4.1 Estado do HPA

```bash
kubectl get hpa -n production chronos-api -o wide
kubectl describe hpa -n production chronos-api | tail -20
```

✅ **Esperado:** `REPLICAS` entre 4–12, eventos `SuccessfulRescale` recentes, CPU target em torno de 70%.
⚠️ **Se não:** réplicas em 12 e memória ainda alta — HPA não resolve; avance para mitigação manual.

### 4.2 Status do Argo CD

```bash
argocd app get hvt/chronos-api
argocd app history hvt/chronos-api | head -5
```

✅ **Esperado:** status `Synced` e `Healthy`. Deploy nas últimas 2 horas pode ser causa da pressão.
⚠️ **Se não:** status `OutOfSync` ou `Degraded` — execute sync manual na Seção 5.3.

---

## 5. Ações de Mitigação

Execute na ordem. Verifique o resultado de cada ação antes de avançar.

### 5.1 Reiniciar pods com alto consumo

```bash
kubectl delete pod -n production \
  $(kubectl top pods -n production -l app=chronos-api --sort-by=memory --no-headers | head -3 | awk '{print $1}')
```

Aguarde 60 segundos e verifique:

```bash
kubectl get pods -n production -l app=chronos-api
kubectl top pods -n production -l app=chronos-api --sort-by=memory
```

✅ **Esperado:** pods reiniciados com `Running` e memória abaixo de 70%.
⚠️ **Se não:** memória ainda alta — avance para 5.2.

### 5.2 Escalar réplicas manualmente

```bash
kubectl scale deployment -n production chronos-api --replicas=10
```

Aguarde 90 segundos:

```bash
kubectl rollout status deployment/chronos-api -n production
kubectl top pods -n production -l app=chronos-api --sort-by=memory
```

✅ **Esperado:** 10 pods `Running` com uso de memória por pod abaixo de 80%.
⚠️ **Se não:** memória acima de 85% após 15 minutos — acione `@chronos-core` (Seção 6).

### 5.3 Forçar sync no Argo CD

Execute apenas se o status do Argo CD for `OutOfSync` ou `Degraded`:

```bash
argocd app sync hvt/chronos-api --prune --force
argocd app wait hvt/chronos-api --timeout 120
```

✅ **Esperado:** status `Synced` e `Healthy` dentro de 120 segundos.
⚠️ **Se não:** registre o erro e acione `@chronos-core`.

---

## 6. Critérios Objetivos de Escalação

Acione `@chronos-core` imediatamente se **qualquer** condição abaixo for verdadeira:

| Condição | Gatilho |
|----------|---------|
| Memória > 85% após mitigações (5.1 + 5.2) | > 15 min persistindo |
| Pods com status `OOMKilled` | ≥ 3 em 30 minutos |
| Pods com status `CrashLoopBackOff` | ≥ 2 simultâneos |
| Fila SQS (`ApproximateNumberOfMessages`) | > 50.000 mensagens |
| Falha de conexão com o Ledger (PostgreSQL) | qualquer timeout ou recusa |
| Sync do Argo CD falhou após tentativa manual | status `Degraded` persistente |
| Todas as 12 réplicas ativas e memória ainda > 85% | — |

Poste no `#oncall-chronos`:

[ESCALAÇÃO] Acionando @chronos-core.
Condição: <condição exata da tabela acima>
Início do incidente: <HH:MM UTC>
Ações executadas: <Seções X, Y, Z>
Logs: <link do Beacon ou output copiado>

---

## 7. Encerramento do Incidente

O incidente está encerrado quando **todas** as condições abaixo forem verdadeiras por **15 minutos consecutivos**:

- Memória em todos os pods: **< 75%**
- Todos os pods com status `Running` (0 com `OOMKilled` ou `CrashLoopBackOff`)
- HPA sem eventos de rescale nos últimos 10 minutos
- Nenhum novo alerta `[CRITICAL] High memory usage` disparado no `#oncall-chronos`

Poste no `#oncall-chronos`:

[ENCERRADO] Incidente: High memory usage on Chronos API pods.
Início: <HH:MM UTC> | Encerramento: <HH:MM UTC> | Duração: <X min>
Causa: <identificada ou "em investigação">
Ação tomada: <resumo 1–2 linhas>
Memória atual: <valor>% | Pods saudáveis: <N>/12
```

---

## Análise do framework

**Justificativa:** O prompt estrutura a instrução nas quatro camadas do framework **R-I-S-E**. O **Role** estabelece o perfil do agente (SRE sênior) e o tom esperado ("clara, objetiva, à prova de ambiguidade"). O **Input** descreve o problema concreto — o alerta, frequência, ambiente, dependências e ferramentas disponíveis. Os **Steps** definem o processo de construção em 8 etapas ordenadas, cada uma com escopo preciso. A **Expectation** lista critérios de aceitação mensuráveis (autonomia, comandos executáveis, verificação explícita, limite de palavras e formato Markdown), eliminando qualquer margem de interpretação sobre o entregável esperado.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| R — Role | `"Você é um engenheiro sênior de confiabilidade de site (SRE) com ampla experiência em ambientes Kubernetes em produção... Você escreve documentação operacional clara, objetiva e à prova de ambiguidade, projetada para ser executada por plantonistas sob pressão"` |
| I — Input | `"O alerta recorrente abaixo dispara em média 4 vezes por semana... Não existe runbook documentado... Plataforma: EKS, namespace production... Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS)... Ferramentas disponíveis no plantão: kubectl, aws cli, argocd cli"` |
| S — Steps | `"1. Abertura do incidente... 2. Diagnóstico inicial — estado dos pods... 3. Diagnóstico de dependências... 4. Verificação do HPA e do Argo CD... 5. Ações de mitigação... 6. Critérios objetivos de escalação... 7. Critério de encerramento do incidente"` |
| E — Expectation | `"Autonomia total: um plantonista sem conhecimento prévio... Comandos 100% executáveis... Verificação explícita em cada passo... Critérios de escalação sem subjetividade... Critério de encerramento claro... Extensão adequada: entre 600 e 1000 palavras... Formato Markdown"` |
