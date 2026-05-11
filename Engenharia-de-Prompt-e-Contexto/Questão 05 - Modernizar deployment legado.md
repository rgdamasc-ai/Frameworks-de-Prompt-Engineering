# Questão 05 — Modernizar deployment legado

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** Sonnet 4.6
**Framework:** B-A-B (Before / After / Bridge)

---

## Contexto

> Um manifest Kubernetes legado do serviço `chronos-api` expõe credenciais em texto puro, roda com réplica única, usa imagem `:latest` e não possui nenhum contexto de segurança, recursos ou probes definidos. O objetivo é modernizá-lo seguindo boas práticas de segurança, alta disponibilidade e observabilidade.

---

## Prompt usado

```
# Before
Você é um engenheiro sênior de plataforma especializado em Kubernetes e práticas
de produção. Analise o deployment legado abaixo, identificando todos os problemas
de segurança, disponibilidade e aderência às boas práticas:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"
```

# After
A versão modernizada DEVE obrigatoriamente conter:

**Alta Disponibilidade**
- replicas: mínimo 3
- strategy do tipo RollingUpdate com maxSurge e maxUnavailable explícitos
- podAntiAffinity para distribuição entre nodes

**Imagem**
- Tag semântica versionada (ex: 1.4.2) — proibido o uso de `latest`
- imagePullPolicy: IfNotPresent

**Segurança**
- Credenciais (DB_PASSWORD, JWT_SECRET) removidas do manifest e referenciadas
  via secretKeyRef apontando para um Secret Kubernetes separado
- securityContext no nível do pod: runAsNonRoot: true, runAsUser >= 1000
- securityContext no nível do container: readOnlyRootFilesystem: true,
  allowPrivilegeEscalation: false, capabilities drop ALL

**Recursos**
- resources.requests e resources.limits definidos para CPU e memória
- Valores razoáveis para uma API (ex: request 100m/128Mi, limit 500m/512Mi)

**Observabilidade e Resiliência**
- livenessProbe via httpGet no path /healthz, com initialDelaySeconds,
  periodSeconds e failureThreshold
- readinessProbe via httpGet no path /ready, com parâmetros distintos
  do liveness

**Boas Práticas Gerais**
- labels enriquecidas: app, version, component, managed-by
- annotations com descrição e contato do time responsável
- terminationGracePeriodSeconds: 30
- YAML válido, indentado e comentado onde necessário

# Bridge
Com base no diagnóstico do manifest legado e nos requisitos acima, produza:

1. **Resumo dos problemas encontrados** no manifest original — liste cada item
   violado e classifique como: 🔴 Crítico (segurança/disponibilidade) ou
   🟡 Melhoria (boas práticas).
2. **O manifest modernizado completo** — um único bloco YAML pronto para
   aplicar via `kubectl apply`, sem placeholders vagos. Onde um valor real
   depende do contexto (ex: tag da imagem, nome do Secret), use um valor
   exemplo realista e sinalize com um comentário inline `# ajustar`.
3. **Um Secret Kubernetes separado** (somente a estrutura, com os campos
   DB_PASSWORD e JWT_SECRET em base64 placeholder) para acompanhar o deploy.
4. **Checklist de validação pós-apply** com os comandos `kubectl` para
   confirmar que replicas, probes e securityContext estão funcionando
   corretamente em produção.

Responda em português. Organize a saída nas quatro seções acima, nessa ordem.
```

---

## Output completo da IA

```
### 1. Resumo dos problemas encontrados

| # | Problema | Classificação |
|---|----------|---------------|
| 1 | `DB_PASSWORD` e `JWT_SECRET` em texto puro no manifest | 🔴 Crítico |
| 2 | `replicas: 1` — ponto único de falha, sem alta disponibilidade | 🔴 Crítico |
| 3 | `image: chronos-api:latest` — tag não determinística | 🔴 Crítico |
| 4 | Ausência de `securityContext` — container roda como root | 🔴 Crítico |
| 5 | Ausência de `resources.requests/limits` — risco de OOMKill ou consumo total do node | 🔴 Crítico |
| 6 | Ausência de `strategy` RollingUpdate — risco de downtime em deploys | 🟡 Melhoria |
| 7 | Ausência de `podAntiAffinity` — todos os pods podem cair no mesmo node | 🟡 Melhoria |
| 8 | Ausência de `livenessProbe` e `readinessProbe` — sem health checking | 🟡 Melhoria |
| 9 | `imagePullPolicy` não definido | 🟡 Melhoria |
| 10 | Labels incompletas (faltam `version`, `component`, `managed-by`) | 🟡 Melhoria |
| 11 | Ausência de `annotations` com descrição e contato do time | 🟡 Melhoria |
| 12 | `terminationGracePeriodSeconds` não definido | 🟡 Melhoria |

---

### 2. Manifest modernizado completo

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    version: "1.4.2"          # ajustar para a tag real da imagem
    component: api
    managed-by: platform-team
  annotations:
    description: "API principal do sistema Chronos"
    contact: "platform-team@empresa.com"   # ajustar para o e-mail do time
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chronos-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: chronos-api
        version: "1.4.2"      # ajustar
        component: api
        managed-by: platform-team
    spec:
      terminationGracePeriodSeconds: 30
      # Contexto de segurança no nível do pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      # Distribuição de pods entre nodes distintos
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: chronos-api
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: chronos-api:1.4.2   # ajustar para a tag semântica real
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        env:
        # Credenciais referenciadas via Secret — nunca em texto puro
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets   # ajustar se o Secret tiver outro nome
              key: DB_PASSWORD
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: JWT_SECRET
        # Contexto de segurança no nível do container
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        # Probe de vivacidade: reinicia o container se travar
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        # Probe de prontidão: remove do balanceador se não estiver pronto
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
```

---

### 3. Secret Kubernetes separado

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: chronos-api-secrets
  namespace: production
  labels:
    app: chronos-api
    managed-by: platform-team
type: Opaque
data:
  # Valores abaixo são placeholders em base64 — substituir pelos reais
  DB_PASSWORD: BASE64_DO_PASSWORD_REAL==   # ajustar: echo -n 'senha' | base64
  JWT_SECRET: BASE64_DO_JWT_SECRET_REAL==  # ajustar: echo -n 'secret' | base64
```

---

### 4. Checklist de validação pós-apply

```bash
# 1. Confirmar que as 3 réplicas estão Running
kubectl get deployment chronos-api -n production

# 2. Verificar distribuição dos pods em nodes distintos
kubectl get pods -n production -l app=chronos-api -o wide

# 3. Verificar eventos de liveness e readiness probe
kubectl describe pods -n production -l app=chronos-api | grep -A8 "Liveness\|Readiness"

# 4. Confirmar securityContext aplicado no pod
kubectl get pod -n production -l app=chronos-api \
  -o jsonpath='{.items[0].spec.securityContext}' | jq .

# 5. Confirmar securityContext aplicado no container
kubectl get pod -n production -l app=chronos-api \
  -o jsonpath='{.items[0].spec.containers[0].securityContext}' | jq .

# 6. Confirmar resource limits e requests
kubectl get pod -n production -l app=chronos-api \
  -o jsonpath='{.items[0].spec.containers[0].resources}' | jq .

# 7. Garantir que nenhum pod usa imagem :latest
kubectl get pods -n production -l app=chronos-api \
  -o jsonpath='{.items[*].spec.containers[*].image}'

# 8. Verificar que o Secret existe e tem as chaves corretas
kubectl get secret chronos-api-secrets -n production -o jsonpath='{.data}' | jq 'keys'
```

---

## Análise do framework

**Justificativa:** O framework B-A-B (Before / After / Bridge) estrutura o prompt em três camadas progressivas: primeiro estabelece o estado atual problemático com evidências concretas (o manifest legado), depois define com precisão o estado desejado através de requisitos categorizados, e por fim instrui a IA sobre como construir a ponte entre os dois estados — gerando não apenas o manifest final, mas todo o raciocínio intermediário (diagnóstico, secret separado, checklist). Essa estrutura é especialmente eficaz para tarefas de refatoração/modernização, pois elimina ambiguidade sobre o delta esperado.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| B — Before (estado atual) | `"Você é um engenheiro sênior... Analise o deployment legado abaixo"` + o bloco YAML com `replicas: 1`, `latest`, credenciais expostas |
| A — After (estado desejado) | `"A versão modernizada DEVE obrigatoriamente conter"` seguido das seções Alta Disponibilidade, Imagem, Segurança, Recursos, Observabilidade e Boas Práticas |
| B — Bridge (como chegar lá) | `"Com base no diagnóstico... produza: 1. Resumo dos problemas... 2. O manifest modernizado completo... 3. Um Secret... 4. Checklist de validação"` |
