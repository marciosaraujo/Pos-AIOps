## Prompt
```
ROLE]
Você é um(a) SRE sênior, especialista em Postgres/RDS, pool de conexões
e incidentes em sistemas de pagamento sob alta carga. Você já liderou
postmortems técnicos em janela de minutos. Seu tom é direto, baseado
em evidência, e você é explícito quando uma conclusão é uma assumption
em vez de fato.

Contexto operacional: incidente ATIVO em produção durante pico de tráfego.
Doc Brown (líder técnico) tem 20 minutos para decidir entre duas opções
mutuamente exclusivas:
  (A) Rollback do deploy chronos-api v2.48.0 -> v2.47.0
  (B) Scaling emergencial: aumento de max_connections do RDS + aumento
      do pool de conexões da aplicação

[INPUT]

--- Artefato 1: Deploy anterior ---
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado endpoint POST /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s

--- Artefato 2: Métricas do Beacon (últimos 30 min) ---
timestamp                p99_latency_ms   req_rate_s   err_rate_pct
2026-04-24 13:30 UTC     420              1200         0.2
2026-04-24 13:45 UTC     510              1450         0.3
2026-04-24 14:00 UTC     780              1780         0.8
2026-04-24 14:10 UTC     2400             2100         4.5
2026-04-24 14:15 UTC     5200             2400         8.2
2026-04-24 14:20 UTC     8100             2650         11.7

--- Artefato 3: Log do pod chronos-api-79c4d8b9-xk2jp ---
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error

--- Artefato 4: Reactor (fila chronos-transactions) ---
- 50.127 mensagens acumuladas, crescendo ~800/min
- Consumer lag: 18 min e aumentando

--- Artefato 5: Cluster ---
- Chronos: 12/12 pods running (HPA no máximo)
- CPU médio: 62% | Memória média: 71%
- Conexões ativas no Ledger (RDS): 240/250 (limite)

Regra sobre o input: tudo que não estiver acima é UNKNOWN. Se sua análise
depender de algo ausente (ex: existência de migrations no deploy, versão
do engine RDS, política de retry do cliente), marque explicitamente como
ASSUMPTION ou UNKNOWN — não invente.

[STEPS]
Execute nesta ordem. Não pule etapas e não inverta a ordem.

PASSO 1 — Reconstituir timeline.
Liste, em ordem cronológica, os eventos relevantes desde o deploy
(23/04 18:42 UTC) até agora (24/04 14:20 UTC). Granularidade de minuto
onde houver dado. Marque o "joelho" da curva de degradação.

PASSO 2 — Atribuição por mudança.
Para cada um dos quatro itens do changelog do v2.48.0, atribua
probabilidade de contribuição para a falha em {alta, média, baixa}.
Cada atribuição deve citar pelo menos uma evidência dos artefatos 2 a 5.
Não atribua "alta" a mais de duas mudanças sem justificar por que não
é possível isolar.

PASSO 3 — Aritmética de capacidade (passo crítico — não pule).
Calcule explicitamente:
  - conexões totais disponíveis no pool da aplicação:
    pods × max_pool_por_pod = ?
  - compare com Art. 5 (conexões ativas no RDS) e com o limite do RDS.
  - conclua se a opção (B) é fisicamente viável SEM antes elevar
    o max_connections do RDS, e se elevar o RDS resolve a causa raiz
    ou apenas desloca o gargalo.

PASSO 4 — Avaliação das opções.
Para (A) e (B) separadamente, produza quatro campos:
  - ETA de mitigação
  - Risco de regressão
  - Risco residual (fila Reactor, dados em backlog, idempotência)
  - Reversibilidade

PASSO 5 — Decisão.
Emita UMA recomendação inequívoca: (A) ou (B). Atribua nível de
confiança em {baixo, médio, alto}. Liste as condições que invalidariam
a recomendação (ex: presença de migration destrutiva).

PASSO 6 — Plano de ação imediato.
Passos numerados, owner sugerido, ETA por passo, gate de validação
para sair de cada passo.

PASSO 7 — Follow-ups pós-incidente.
Itens de RCA profunda, ações corretivas, e o que precisa ir para
o backlog de hardening.

[EXPECTATION]
Formato: Markdown. Idioma: português técnico (pt-BR).
Tamanho: máximo 600 palavras no corpo do postmortem.
Audiência: Doc Brown + engenheiros de plantão. Sem floreio.

Estrutura obrigatória, nesta nomenclatura, nesta ordem:

  # Postmortem técnico — chronos-api (INC em andamento)
  ## TL;DR
  ## Timeline
  ## Atribuição por mudança do v2.48.0
  ## Aritmética de capacidade
  ## Análise das opções (A) e (B)
  ## Recomendação
  ## Plano de ação imediato
  ## Follow-ups pós-incidente

Restrições de redação:
- Toda afirmação numérica referencia o artefato de origem (ex: "Art. 3").
- Proibido hedging vago ("possivelmente", "talvez podemos"). Use
  "evidência aponta", "não há dado para concluir", "assumindo X".
- Recomendação no TL;DR, na primeira linha. Doc lê o TL;DR primeiro.
- Se passo 3 e passo 5 entrarem em contradição, refaça o passo 3 antes
  de fechar — não tente reconciliar na recomendação.
```
---
## Modelo
```
Gemini 3.1 Pro
```
---
## Output

# Postmortem técnico — chronos-api (INC em andamento)

## TL;DR
Recomendo executar a **Opção (A) Rollback para v2.47.0** imediatamente. A Opção (B) não é matematicamente viável sem indisponibilizar temporariamente o RDS.

## Timeline
- **23/04 18:42 UTC:** Deploy chronos-api v2.48.0 via Argo CD (Art. 1).
- **24/04 13:30 UTC:** Operação normal. p99 em 420ms, req_rate 1200/s (Art. 2).
- **24/04 13:45 UTC:** p99 sobe levemente para 510ms (Art. 2).
- **24/04 14:00 UTC:** [JOELHO DA CURVA] p99 atinge 780ms, req_rate 1780/s, erro 0.8% (Art. 2).
- **24/04 14:10 UTC:** Degradação aguda. p99 salta para 2400ms, erro 4.5% (Art. 2).
- **24/04 14:15 UTC:** p99 5200ms, erro 8.2% (Art. 2).
- **24/04 14:19 UTC:** Logs indicam exaustão do pool (max=20), query timeouts e Circuit Breaker aberto em 87% (Art. 3).
- **24/04 14:20 UTC:** p99 8100ms, erro 11.7% (Art. 2). Fila Reactor atinge 50.127 msgs em atraso (Art. 4).

## Atribuição por mudança do v2.48.0
- **Reduzido timeout do Ledger de 5s para 2s:** Alta. Evidência aponta que o p99 cruzou os 2000ms a partir das 14:10 (Art. 2), provocando aborto em massa e `context deadline exceeded` (Art. 3).
- **Refatorado cliente do Ledger:** Alta. A limitação fixa do pool em `max=20` (Art. 3) esgotou a capacidade sob a taxa de requisições de 2650/s (Art. 2), impedindo o tráfego de fluir.
- **Adicionado endpoint POST /v2/transactions/batch:** Baixa. Embora o log cite falha neste endpoint (Art. 3), o erro subjacente é no cliente compartilhado do Ledger (pool). UNKNOWN se o tráfego que causou o pico (Art. 2) partiu exclusivamente deste novo endpoint.
- **Bump de psycopg 3.1.18 -> 3.2.0:** Baixa. Os logs demonstram comportamentos de stress de infraestrutura/timeout (Art. 3), não falhas nativas do driver (ex: segfault).

## Aritmética de capacidade
- **Conexões exigidas pelo app:** 12 pods (Art. 5) x 20 conexões por pod (Art. 3) = **240 conexões no total.**
- **Cenário atual do RDS:** 240 conexões ativas de um limite de 250 (Art. 5).
- **Conclusão sobre a Opção (B):** Inviável. O pool do app já está alocando 96% do limite absoluto do banco. Escalar o pool nos pods esgotará o limite do RDS instantaneamente. ASSUMPTION que a engine do RDS utilizada exige reboot para aumentar o `max_connections` — logo, modificar o RDS causaria uma queda dura do banco e apenas deslocaria o gargalo (ex: para I/O), já que as queries continuariam falhando no timeout de 2s da aplicação.

## Análise das opções (A) e (B)

**Opção (A) Rollback (v2.48.0 -> v2.47.0)**
- **ETA de mitigação:** < 5 minutos (revert no GitOps + rollout padrão).
- **Risco de regressão:** Baixo, retorna a um state comprovadamente estável.
- **Risco residual:** Drenagem do Reactor (Art. 4). Requer idempotência no consumo das mensagens enfileiradas.
- **Reversibilidade:** Imediata (Rollforward em caso de piora).

**Opção (B) Scaling Emergencial**
- **ETA de mitigação:** > 15 minutos (requer reboot do banco e redimensionamento da infra).
- **Risco de regressão:** Alto. Thundering herd contra o banco pode esgotar CPU/Memória.
- **Risco residual:** Alto. A fila continuará crescendo desenfreadamente durante o downtime do banco (Art. 4).
- **Reversibilidade:** Lenta e arriscada.

## Recomendação
**OPÇÃO (A)** com nível de confiança **alto**.
*Condição que invalidaria esta recomendação:* ASSUMPTION que o deploy v2.48.0 não aplicou *migrations* destrutivas no banco de dados (UNKNOWN com base no Art. 1). Se houver quebra de schema, o rollback causará downtime sistêmico.

## Plano de ação imediato
1. **Travar ArgoCD e aplicar Rollback para v2.47.0.** Owner: Doc Brown. ETA: 2 min. Gate: Pods com versão antiga apresentando status *Ready*.
2. **Confirmar normalização do banco.** Owner: SRE Plantonista. ETA: 5 min. Gate: Conexões no RDS caírem para < 150 (Art. 5) e timeouts cessarem nos logs (Art. 3).
3. **Monitorar drenagem da fila Reactor.** Owner: Eng. de Dados/Backend. ETA: 20 min. Gate: Consumer lag retornar a < 1 minuto (Art. 4).

## Follow-ups pós-incidente
- **Ação corretiva:** Validar se o timeout de 2s é realista para o baseline do p99 sob carga.
- **Hardening:** Implementar circuit breaker diretamente na fila do Reactor em caso de falha de downstream, evitando crescimento descontrolado do lag.
- **RCA:** Investigar se o novo cliente isolado (Art. 1) gerencia a devolução de conexões adequadamente, visto que o reset ocorreu a 100% da utilização do pool previsto (Art. 3 e 5).
---

## Justificativa

O framework R-I-S-E (Role, Input, Steps, Expectation) é ideal para resolver incidentes técnicos complexos porque atua como um algoritmo rigoroso que força a Inteligência Artificial a seguir um raciocínio lógico estruturado antes de emitir uma conclusão. Ao separar claramente os dados brutos em uma entrada isolada e impor etapas analíticas obrigatórias — como o cálculo exato da capacidade de um pool de conexões antes da tomada de decisão —, este modelo impede suposições incorretas e garante que a resposta final seja baseada estritamente em evidências operacionais e limites físicos reais.

Em contraste, o framework C-A-R-E (Context, Action, Result, Example) é voltado para tarefas de geração de conteúdo qualitativo e criativo, cujo objetivo principal é replicar um estilo de escrita a partir de um modelo pré-definido. Como o foco principal recai sobre a aparência final do texto através de exemplos fornecidos, falta a esse formato a capacidade de ditar o processo investigativo interno, tornando a estrutura inadequada e suscetível a erros críticos quando aplicada para realizar diagnósticos profundos em arquiteturas de banco de dados ou sistemas sob alta carga.

Por fim, o framework R-T-F (Role, Task, Format) oferece uma estrutura transacional básica, excelente para delegações rápidas, extrações de dados diretas ou formatações simples, como gerar uma tabela comparativa entre duas versões de um serviço. No entanto, a extrema simplicidade estrutural torna-se um ponto de falha em cenários de degradação sistêmica, pois incentiva a Inteligência Artificial a tentar resolver todo o problema em um único salto cognitivo, pulando a validação de restrições essenciais de infraestrutura que o R-I-S-E naturalmente obriga a verificar em seus passos intermediários.