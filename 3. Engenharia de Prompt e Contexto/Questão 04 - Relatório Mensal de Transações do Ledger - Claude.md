## Prompt
```
[Task] Atue como um DBA/Engenheiro de Dados. Escreva uma query para PostgreSQL baseada no DDL das tabelas 'transactions' e 'customers' fornecidas abaixo.

[Action] Gere um relatório consolidado aplicando as seguintes regras:
1. Filtre apenas transações com status = 'completed'.
2. Limite a busca aos últimos 6 meses corridos a partir de 2026-04-24 (inclusive).
3. Agrupe os dados por mês de finalização da transação (coluna completed_at no formato YYYY-MM) e por categoria.
4. Para cada linha, calcule duas métricas: quantidade de transações e volume total convertido para reais (a partir de amount_cents, exibindo 2 casas decimais).
5. Ordene a saída por mês de forma crescente e, em seguida, por categoria de forma crescente.

[Goal] Entregue apenas o código SQL otimizado, sem explicações adicionais, pronto para ser executado para extração do relatório.

DDL:
CREATE TABLE transactions (id BIGSERIAL PRIMARY KEY, customer_id BIGINT NOT NULL REFERENCES customers(id), category VARCHAR(32) NOT NULL, amount_cents BIGINT NOT NULL, status VARCHAR(16) NOT NULL, payment_method VARCHAR(16), created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), completed_at TIMESTAMPTZ);
CREATE TABLE customers (id BIGSERIAL PRIMARY KEY, segment VARCHAR(16) NOT NULL, country CHAR(2) NOT NULL, signup_at TIMESTAMPTZ NOT NULL DEFAULT NOW());
```
---
## Modelo
```
Claude Opus 4.7 Adaptativo
```
---
## Output
```
SELECT
    TO_CHAR(completed_at, 'YYYY-MM') AS month,
    category,
    COUNT(*) AS transaction_count,
    ROUND(SUM(amount_cents) / 100.0, 2) AS total_amount_brl
FROM transactions
WHERE status = 'completed'
  AND completed_at >= DATE '2026-04-24' - INTERVAL '6 months'
  AND completed_at <  DATE '2026-04-24' + INTERVAL '1 day'
GROUP BY TO_CHAR(completed_at, 'YYYY-MM'), category
ORDER BY month ASC, category ASC;
```
---
## Justificativa

O framework TAG (Task, Action, Goal) foi adequado para este caso porque a demanda era uma tarefa técnica determinística com saída bem definida: a Task estabele…O framework TAG (Task, Action, Goal) foi adequado para este caso porque a demanda era uma tarefa técnica determinística com saída bem definida: a Task estabeleceu o papel (DBA/Engenheiro de Dados) e o contexto técnico (PostgreSQL + DDL), a Action detalhou as regras de negócio em passos numerados e mensuráveis (filtros, janela temporal, agrupamentos e conversões), e o Goal definiu o formato de entrega (apenas SQL executável, sem explicações). Essa estrutura minimalista entrega o contexto necessário para um output preciso e reprodutível, sem as camadas exploratórias de frameworks como CRISPE ou RTF, que seriam desnecessárias em tarefas de engenharia focadas em automação e geração de scripts.
