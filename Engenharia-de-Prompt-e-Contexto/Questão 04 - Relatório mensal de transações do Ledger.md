# Questão 04 — Relatório mensal de transações do Ledger

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** Sonnet 4.6
**Framework:** T-A-G

---

## Contexto

> Necessidade de gerar um relatório mensal consolidado de transações por categoria
> para uma reunião de negócios sobre crescimento nos últimos 6 meses, com base
> na tabela `transactions` do Ledger (PostgreSQL).

---

## Prompt usado

```
# Task
Escreva uma query SQL em PostgreSQL que gere um relatório mensal de transações consolidadas por categoria, para ser apresentado em uma reunião de negócios sobre crescimento nos últimos 6 meses.

# Action
A query deve:

1. Filtrar apenas transações com status = 'completed'
2. Recortar os últimos 6 meses corridos a partir de 2026-04-24 (inclusive o mês corrente)
3. Agrupar os resultados por mês (formato YYYY-MM) e por category
4. Calcular por grupo:
   - transaction_count: quantidade de transações
   - total_amount_brl: soma de amount_cents convertida para reais com 2 casas decimais (amount_cents / 100.0)
5. Ordenar por mês crescente e, dentro do mesmo mês, por categoria crescente
6. Considerar apenas as categorias: subscription, one_time, refund e credit_adjustment

Esquema disponível:

CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

CREATE TABLE customers (
  id        BIGSERIAL PRIMARY KEY,
  segment   VARCHAR(16) NOT NULL,
  country   CHAR(2) NOT NULL,
  signup_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

# Goal
Produzir uma única query SQL pronta para execução no PostgreSQL, retornando exatamente estas colunas:

- month (TEXT, formato YYYY-MM): mês de referência
- category (TEXT): categoria da transação
- transaction_count (BIGINT): quantidade de transações
- total_amount_brl (NUMERIC 12,2): volume total em reais

Retorne apenas o bloco SQL, sem explicações ou texto adicional.
A query deve aproveitar os índices existentes em created_at, status e category,
usando DATE_TRUNC ou TO_CHAR para formatar o mês.
```

---

## Output completo da IA

```sql
SELECT
    TO_CHAR(DATE_TRUNC('month', created_at), 'YYYY-MM')  AS month,
    category,
    COUNT(*)::BIGINT                                      AS transaction_count,
    ROUND(SUM(amount_cents) / 100.0, 2)                  AS total_amount_brl
FROM transactions
WHERE status = 'completed'
  AND category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
  AND created_at >= DATE_TRUNC('month', '2026-04-24'::DATE) - INTERVAL '5 months'
  AND created_at <  DATE_TRUNC('month', '2026-04-24'::DATE) + INTERVAL '1 month'
GROUP BY DATE_TRUNC('month', created_at), category
ORDER BY DATE_TRUNC('month', created_at), category;
```

---

## Análise do framework

**Justificativa:** O prompt segue o framework T-A-G com três seções explicitamente nomeadas: `# Task` define o que deve ser produzido e para qual propósito de negócio; `# Action` detalha os requisitos técnicos da query (filtros, agrupamentos, cálculos, ordenação); `# Goal` especifica o contrato de saída — colunas, tipos, formato e restrições de execução.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| T — Task | `"Escreva uma query SQL em PostgreSQL que gere um relatório mensal de transações consolidadas por categoria, para ser apresentado em uma reunião de negócios sobre crescimento nos últimos 6 meses."` |
| A — Action | `"A query deve: 1. Filtrar apenas transações com status = 'completed' … 6. Considerar apenas as categorias: subscription, one_time, refund e credit_adjustment"` |
| G — Goal | `"Produzir uma única query SQL pronta para execução no PostgreSQL, retornando exatamente estas colunas: month, category, transaction_count, total_amount_brl"` |
