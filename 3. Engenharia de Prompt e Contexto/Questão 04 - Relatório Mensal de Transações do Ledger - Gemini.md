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
Gemini 3.1 Pro
```
---
## Output
```
SELECT
    TO_CHAR(completed_at, 'YYYY-MM') AS mes,
    category,
    COUNT(id) AS quantidade_transacoes,
    ROUND(SUM(amount_cents) / 100.0, 2) AS volume_total_reais
FROM
    transactions
WHERE
    status = 'completed'
    AND completed_at >= '2025-10-24 00:00:00+00' -- Início da janela de 6 meses
    AND completed_at <= '2026-04-24 23:59:59+00' -- Data de corte da solicitação
GROUP BY
    TO_CHAR(completed_at, 'YYYY-MM'),
    category
ORDER BY
    mes ASC,
    category ASC;
```
---
## Justificativa

A aplicação do framework T-A-G neste caso estrutura o prompt definindo a Tarefa (Task) para que a inteligência artificial atue como um DBA especializado em PostgreSQL, o que garante a utilização de funções nativas do banco. A Ação (Action) mapeia as regras de negócio para instruções técnicas, detalhando a filtragem de status, o recorte temporal, as agregações e as conversões matemáticas de centavos para reais. Por fim, o Objetivo (Goal) restringe o formato da entrega, exigindo apenas o código SQL puro e otimizado, visando focar na prontidão para execução e eficiência do fluxo de trabalho.
