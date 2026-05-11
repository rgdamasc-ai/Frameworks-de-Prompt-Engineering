# Questão 03 — Relatório de redução de custos cloud

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** Claude Sonnet 4.6
**Framework:** T-A-G (Task / Action / Goal)

---

## Contexto

> Análise do breakdown de custos mensais de uma conta AWS com 12 serviços, totalizando $41.800/mês. O objetivo é identificar e priorizar oportunidades de redução de pelo menos 15% sem degradar SLAs de produção, como insumo para decisão executiva.

---

## Prompt usado

```
# Task
Analise o breakdown de custos AWS abaixo e gere um relatório executivo de oportunidades de
redução de custos, priorizadas por impacto financeiro, com o objetivo de atingir pelo menos
15% de redução no custo total mensal sem degradar SLAs de produção.

# Action
Com base nos dados do CSV fornecido, execute as seguintes etapas:

1. Calcule o custo total mensal da conta e o valor em USD equivalente a 15%.
2. Para cada serviço ou grupo de serviços, identifique oportunidades concretas
   de economia (ex: rightsizing, mudança de tier, ajuste de retenção,
   conversão de on-demand para reserved/savings plans, etc.).
3. Para cada oportunidade identificada, estruture as informações em formato
   de tabela com as colunas:
   - Serviço / Grupo
   - Oportunidade (descrição curta da ação)
   - Economia estimada (USD/mês)
   - % da conta total
   - Esforço de implementação (Baixo / Médio / Alto)
   - Riscos ou pré-requisitos
4. Ordene as oportunidades por economia estimada (maior para menor).
5. Ao final da tabela, inclua:
   - Soma total de economia identificada em USD e em % da conta
   - Um semáforo visual indicando se a meta de 15% foi atingida,
     está em risco ou não foi alcançada
   - Recomendação de sequência de execução (quick wins primeiro)
6. Feche com um parágrafo executivo de 3–5 linhas, em linguagem de diretoria,
   resumindo o cenário e a viabilidade da meta.
7. Com a saída (output) deste prompt, use o "Modelo.md" que esta no projeto
   "Frameworks-de-Prompt-Engineering", na pasta "Engenharia-de-Prompt-e-Contexto"
   para preencher os itens e adicione no arquivo
   "Questão 03 - Relatório de redução de custos cloud".
8. Solicite aprovação antes de executar.

# Goal
O relatório será apresentado por Doc Brown à diretora Goldie como insumo
para decisão executiva no próximo trimestre. O output deve ser objetivo,
baseado em dados, acionável e escrito em português. Evite jargões técnicos
sem explicação. O tom deve ser de consultoria — confiante, direto e
orientado a resultado.

CSV de entrada:

servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes
```

---

## Output completo da IA

```
## RELATÓRIO EXECUTIVO — OPORTUNIDADES DE REDUÇÃO DE CUSTOS AWS

**Elaborado por:** Doc Brown
**Destinatário:** Goldie (Diretora)
**Data:** 11 de maio de 2026

---

### 1. Resumo Financeiro

| Indicador                        | Valor   |
|----------------------------------|---------|
| Custo total mensal atual         | $41.800 |
| Meta de redução (15%)            | $6.270  |
| Economia total identificada      | $7.850  |
| Redução estimada alcançada       | 18,8%   |

---

### 2. Tabela de Oportunidades (ordenadas por economia)

| # | Serviço / Grupo    | Oportunidade                                                                                   | Economia (USD/mês) | % da Conta | Esforço | Riscos / Pré-requisitos                                                                                      |
|---|--------------------|------------------------------------------------------------------------------------------------|--------------------|------------|---------|--------------------------------------------------------------------------------------------------------------|
| 1 | EC2 On-Demand      | Rightsizing (utilização média 45%) + conversão de carga-base para Savings Plans de 1 ano      | $2.050             | 4,9%       | Médio   | Requer análise de pico de carga antes de reduzir instâncias; risco de impacto em workloads de pico           |
| 2 | RDS PostgreSQL     | Conversão de on-demand para Reserved Instances de 1 ano (multi-AZ)                            | $1.800             | 4,3%       | Baixo   | Compromisso financeiro de 12 meses; validar dimensionamento atual antes de fixar contrato                    |
| 3 | EKS                | Migração de workloads não-produção para Spot Instances + avaliação de consolidação de clusters | $1.400             | 3,3%       | Alto    | Spot Instances podem ser interrompidas pela AWS; não indicado para produção crítica sem tolerância a falhas   |
| 4 | CloudWatch Logs    | Redução de retenção de 90 para 30 dias + exportação de logs históricos para S3                | $800               | 1,9%       | Baixo   | Verificar requisitos de auditoria/compliance antes de reduzir retenção; exportar logs críticos antes         |
| 5 | S3 Standard        | Lifecycle Policies e Intelligent-Tiering para dados de acesso infrequente                     | $700               | 1,7%       | Baixo   | Necessário mapeamento de padrão de acesso por bucket antes da ativação                                       |
| 6 | ElastiCache Redis  | Rightsizing do cluster (utilização média 40%)                                                  | $500               | 1,2%       | Médio   | Risco de aumento de cache miss se subdimensionado; testes de carga obrigatórios antes da mudança             |
| 7 | NAT Gateway        | Criação de VPC Endpoints para serviços AWS (S3, DynamoDB) para eliminar tráfego via NAT       | $300               | 0,7%       | Médio   | Requer conhecimento de configuração de rede VPC; mudança na topologia de rede                                |
| 8 | Data Transfer Out  | Revisão de fluxos inter-regionais e uso de endpoints na mesma região onde possível            | $200               | 0,5%       | Alto    | Requer análise arquitetural dos fluxos entre regiões; pode envolver mudanças de aplicação                    |
| 9 | Lambda             | Rightsizing de memória via Lambda Power Tuning (utilização ~30%)                              | $100               | 0,2%       | Baixo   | Risco muito baixo; rollback imediato possível se performance for afetada                                     |

---

### 3. Consolidado de Economia

| Indicador                         | USD/mês | % da Conta |
|-----------------------------------|---------|------------|
| Total de economia identificada    | $7.850  | 18,8%      |
| Meta estabelecida (15%)           | $6.270  | 15,0%      |
| Superávit em relação à meta       | +$1.580 | +3,8%      |

**Semáforo:** 🟢 META ATINGIDA — A economia identificada ($7.850 / 18,8%)
supera a meta de 15% ($6.270), com margem de segurança de $1.580.

---

### 4. Sequência de Execução Recomendada

**Fase 1 — Quick Wins (semanas 1–4 | esforço baixo | sem janela de manutenção)**
- RDS Reserved Instances .............. $1.800
- CloudWatch Logs retenção ............. $800
- S3 Lifecycle Policies ................. $700
- Lambda rightsizing ..................... $100
  Subtotal Fase 1: $3.400/mês

**Fase 2 — Esforço Médio (semanas 5–8 | requer análise e validação)**
- EC2 rightsizing + Savings Plans ....... $2.050
- ElastiCache rightsizing ............... $500
- NAT Gateway → VPC Endpoints ........... $300
  Subtotal Fase 2: $2.850/mês

**Fase 3 — Transformacional (semanas 9–12 | maior esforço e análise)**
- EKS Spot + consolidação ............... $1.400
- Data Transfer optimization ............. $200
  Subtotal Fase 3: $1.600/mês

Total acumulado ao final das 3 fases: $7.850/mês

---

### 5. Parágrafo Executivo

A análise da infraestrutura AWS identificou $7.850 em economia mensal
recorrente — equivalente a uma redução de 18,8% sobre o custo atual de
$41.800 —, superando a meta de 15% com uma margem de segurança de $1.580.
O conjunto de oportunidades prioriza ações de baixo risco e alto retorno
nas primeiras quatro semanas, com potencial de $3.400 em economia já no
primeiro mês sem alterações arquiteturais ou janelas de manutenção. Os
serviços com maior retorno são EC2 on-demand e RDS, que juntos respondem
por mais de 38% da conta e apresentam utilização média abaixo de 55%,
sinalizando superprovisionamento estrutural. A execução do plano em três
fases garante progressão sem degradação de SLA, com as mudanças mais
invasivas realizadas somente após validação em ambiente não-produtivo.
A recomendação é aprovar a execução imediata da Fase 1, que não requer
janela de manutenção e captura sozinha mais de $3.400 mensais com risco
operacional mínimo.
```

---

## Análise do framework

**Justificativa:** O prompt aplica o framework **T-A-G (Task / Action / Goal)** com seções explicitamente nomeadas com os três elementos do acrônimo. Cada componente está claramente delimitado, tornando este um dos exemplos mais diretos de aplicação estruturada do framework. A Task define o objeto de análise e o critério de sucesso quantitativo; a Action especifica o processo em 8 passos numerados com formato de entrega; e o Goal define destinatário, tom, idioma e finalidade do output.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| T — Task | `"Analise o breakdown de custos AWS abaixo e gere um relatório executivo de oportunidades de redução de custos, priorizadas por impacto financeiro, com o objetivo de atingir pelo menos 15% de redução no custo total mensal sem degradar SLAs de produção."` |
| A — Action | `"Com base nos dados do CSV fornecido, execute as seguintes etapas: 1. Calcule o custo total... 2. Para cada serviço... 3. Para cada oportunidade identificada, estruture as informações em formato de tabela... 4. Ordene... 5. Ao final da tabela, inclua... 6. Feche com um parágrafo executivo..."` |
| G — Goal | `"O relatório será apresentado por Doc Brown à diretora Goldie como insumo para decisão executiva no próximo trimestre. O output deve ser objetivo, baseado em dados, acionável e escrito em português. Evite jargões técnicos sem explicação. O tom deve ser de consultoria — confiante, direto e orientado a resultado."` |
