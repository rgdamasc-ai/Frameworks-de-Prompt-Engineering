# Questão 07 — Runbook para alerta recorrente (Gemini)

**Data:** 2026-05-16
**Ferramenta usada:** Gemini Code Assist
**Modelo:** Gemini 1.5 Pro
**Framework:** R-I-S-E

---

## Contexto

> Alerta crítico recorrente (`[CRITICAL] High memory usage on Chronos API pods >85% for 10min`) impactando a estabilidade da Chronos API no namespace `production`. A ausência de documentação eleva o tempo de resolução para até 40 minutos.

---

## Prompt usado

```text
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
8. Solicite aprovação antes de executar.

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

# Runbook: [CRITICAL] High Memory Usage — Chronos API

Este documento descreve os procedimentos de diagnóstico e mitigação para o uso excessivo de memória nos pods da Chronos API.

---

## 1. Abertura do Incidente

Ao receber o alerta no Slack, execute as seguintes ações imediatamente:

1. Poste um acknowledgement no canal `#oncall-chronos`:
   `[ACK] Alerta: High memory usage on Chronos API pods. Investigando. Início: $(date -u +%H:%M) UTC.`
2. Registre o horário exato de início da sua investigação.
3. Identifique se o alerta é referente a todos os pods ou a uma instância isolada.

✅ **Esperado:** A equipe está ciente de que o incidente está sendo tratado.
⚠️ **Se não:** Prossiga para o diagnóstico, mas garanta que o ACK seja postado assim que possível.

---

## 2. Diagnóstico Inicial — Estado dos Pods

Verifique o estado atual da carga de trabalho no Kubernetes.

### 2.1 Verificar status e distribuição dos pods
```bash
kubectl get pods -n production -l app=chronos-api -o wide
```
✅ **Esperado:** 6 a 12 pods com status `Running`.
⚠️ **Se não:** Se houver pods em `OOMKilled`, `CrashLoopBackOff` ou `Pending`, anote os nomes e avance para a seção de mitigação (Seção 5).

### 2.2 Verificar uso de memória em tempo real
```bash
kubectl top pods -n production -l app=chronos-api --sort-by=memory
```
✅ **Esperado:** Uso de memória por pod inferior a 80%.
⚠️ **Se não:** Identifique pods com uso > 85%. Se a maioria estiver alta, é um problema sistêmico. Se apenas um estiver alto, pode ser um "leak" isolado.

### 2.3 Verificar eventos recentes de erro
```bash
kubectl get events -n production --field-selector involvedObject.kind=Pod --sort-by='.lastTimestamp' | grep -i "chronos-api"
```
✅ **Esperado:** Ausência de eventos do tipo `Warning` como `OOMKilling` ou `FailedScheduling`.
⚠️ **Se não:** Se encontrar `OOMKilling`, a aplicação atingiu o limite de memória definido no manifesto.

---

## 3. Diagnóstico de Dependências

Verifique se a lentidão em dependências está causando o acúmulo de objetos na memória da API.

### 3.1 Conectividade com Ledger (PostgreSQL)
```bash
kubectl run -it --rm --restart=Never --image=postgres:15-alpine psql-check -n production -- \
  psql -h ledger.production.svc.cluster.local -U chronos -c "SELECT 1;"
```
✅ **Esperado:** Retorno `(1 row)` indicando sucesso na conexão.
⚠️ **Se não:** Timeout ou "Connection Refused". Avance para Escalação (Seção 6).

### 3.2 Saúde do Reactor (SQS)
```bash
aws sqs get-queue-attributes \
  --queue-url $(aws sqs get-queue-url --queue-name chronos-transactions --query 'QueueUrl' --output text) \
  --attribute-names ApproximateNumberOfMessages
```
✅ **Esperado:** `ApproximateNumberOfMessages` abaixo de 10.000.
⚠️ **Se não:** Se a fila estiver crescendo rapidamente (> 50k), a API pode estar retendo mensagens na memória por incapacidade de processamento.

---

## 4. Verificação do HPA e do Argo CD

### 4.1 Estado do Horizontal Pod Autoscaler
```bash
kubectl describe hpa chronos-api -n production
```
✅ **Esperado:** `Replicas` aumentando se a carga estiver alta. Eventos indicando `SuccessfulRescale`.
⚠️ **Se não:** Se o HPA estiver travado no limite máximo (12) e a memória continuar subindo, o escalonamento automático não é suficiente.

### 4.2 Sincronização no Argo CD
```bash
argocd app get hvt/chronos-api
```
✅ **Esperado:** Status `Synced` e `Healthy`.
⚠️ **Se não:** Status `OutOfSync` ou `Degraded`. Verifique se houve um deploy recente que introduziu uma regressão de memória.

---

## 5. Ações de Mitigação

Execute as ações abaixo na ordem sugerida até que o alerta seja resolvido.

### 5.1 Reinício Seguro (Rolling Restart)
Se o problema for um leak de memória ou estado corrompido:
```bash
kubectl rollout restart deployment/chronos-api -n production
kubectl rollout status deployment/chronos-api -n production
```
✅ **Esperado:** Todos os pods reiniciados com uso de memória zerado/baixo.
⚠️ **Se não:** Se a memória subir para > 85% imediatamente após o reinício, o problema é estrutural (carga ou código).

### 5.2 Sincronização Forçada do Argo CD
Se houver divergência de configuração:
```bash
argocd app sync hvt/chronos-api --prune --force
```
✅ **Esperado:** Aplicação retorna ao estado `Synced`.

### 5.3 Escalonamento Manual de Emergência
Se o HPA não estiver respondendo rápido o suficiente:
```bash
kubectl scale deployment/chronos-api -n production --replicas=12
```
✅ **Esperado:** Novas réplicas subindo para dividir a carga de memória.

---

## 6. Critérios Objetivos de Escalação

Acione imediatamente `@chronos-core` via Slack ou PagerDuty se:

1. **Persistência:** Memória acima de 85% por mais de 15 minutos **após** a execução do `rollout restart`.
2. **Dependência Crítica:** Falha de conexão com o Ledger (PostgreSQL) confirmada no passo 3.1.
3. **Saturação Total:** HPA em 12 réplicas com todos os pods acima de 90% de memória.
4. **Instabilidade de Deploy:** Argo CD falhando ao sincronizar mesmo após comando manual.

---

## 7. Critério de Encerramento do Incidente

O incidente é considerado resolvido quando:
1. O uso de memória em todos os pods estiver estável abaixo de 75% por 15 minutos consecutivos.
2. O alerta crítico no `#oncall-chronos` for resolvido automaticamente ou silenciado por métrica normalizada.

**Mensagem de Encerramento:**
Poste no canal:
`[RESOLVIDO] Alerta: High memory usage on Chronos API. Duração: [X] min. Causa: [Leak/Aumento de Carga/Falha Dependência]. Mitigação: [Reinício/Scaling].`

---

## Análise do framework

**Justificativa:** O prompt foi estruturado utilizando o framework **R-I-S-E**, fornecendo contexto técnico e restrições claras. Em comparação com o output gerado pela Claude (Sonnet 4.6), o Gemini 1.5 Pro produziu comandos ligeiramente mais robustos em termos de diagnóstico (como o uso de `kubectl top pods --sort-by=memory` e filtros de eventos mais específicos). Ambos os modelos respeitaram as diretrizes de autonomia e comandos 100% executáveis. O Gemini focou em uma abordagem mais procedural, o que é ideal para runbooks operacionais sob pressão.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| R — Role | `Você é um engenheiro sênior de confiabilidade de site (SRE) com ampla experiência em ambientes Kubernetes em produção...` |
| I — Input | `O alerta recorrente abaixo dispara em média 4 vezes por semana no canal de plantão #oncall-chronos... Plataforma: EKS, namespace production...` |
| S — Steps | `1. Abertura do incidente... 2. Diagnóstico inicial... 5. Ações de mitigação... 7. Critério de encerramento...` |
| E — Expectation | `Autonomia total... Comandos 100% executáveis... Verificação explícita em cada passo... Extensão adequada: entre 600 e 1000 palavras...` |

---