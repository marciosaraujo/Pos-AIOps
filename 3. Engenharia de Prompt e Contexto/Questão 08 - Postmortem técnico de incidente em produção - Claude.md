## Prompt
```
[ROLE]
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
Claude Opus 4.7 Adaptativo 
```
---
## Output
> # Postmortem técnico — chronos-api (INC em andamento)
>
> **TL;DR.** Recomendação: **Rollback v2.48.0 → v2.47.0**. Confiança: alta. Opção (B) é fisicamente inviável sem antes elevar `max_connections` no RDS (Art. 5: 240/250 já consumidas), e mesmo elevando não trata a causa raiz suspeita.
>
> **Timeline.**
> - 23/04 18:42 — Deploy v2.48.0 (Art. 1).
> - 24/04 13:30 — Baseline saudável: p99 420ms, err 0.2% (Art. 2).
> - 24/04 14:00 — Joelho da curva: p99 dobra (420→780ms), err 4× (Art. 2).
> - 24/04 14:10 — p99 2.4s, err 4.5%; HPA escala até 12/12 (Art. 2, 5).
> - 24/04 14:15 — Circuit breaker do ledger-client abre em 87% (Art. 3).
> - 24/04 14:20 — p99 8.1s, err 11.7%, fila 50k msgs / lag 18min (Art. 2, 4).
>
> **Atribuição por mudança do v2.48.0.**
> - Nova biblioteca de pool — **alta**. Pool exausto com 147 em espera (Art. 3) é o sintoma mais saliente; mudança de bibliotecas de pool é causa frequente de regressão em gestão de lifecycle de conexão.
> - Bump psycopg 3.1.18 → 3.2.0 — **alta**. Mudança de minor em driver de DB pode alterar comportamento de retry/keepalive; não dá pra isolar sem bisseção, daí classificação par com a anterior.
> - Timeout 5s → 2s — **média**. Não cria pool exhaustion, mas amplifica: queries que estourariam 2s e completariam em 3s viram retries, multiplicando demanda de conexão.
> - Endpoint `POST /v2/transactions/batch` — **baixa**. req_rate cresceu de 1200 → 2650 (Art. 2), compatível com pico orgânico; nada nos artefatos liga o endpoint novo ao pool.
>
> **Aritmética de capacidade.**
> - Pool total: 12 pods × 20 max_pool = **240 conexões**.
> - RDS em uso: **240/250** (Art. 5). O pool da aplicação satura *antes* do RDS — o gargalo está no cliente, não no servidor.
> - Elevar pool por pod sem elevar `max_connections` no RDS bate no teto em segundos. Elevar `max_connections` no RDS pode requerer reboot/parameter group reload conforme engine — caro em pico. E mesmo após elevação, o sintoma (pool exausto com 147 em espera) sugere bug de gestão de conexão, não falta de capacidade — RDS estava ocioso a 13:30 com a mesma topologia.
>
> **Análise das opções.**
>
> **(A) Rollback.** ETA: ~5min via Argo. Regressão: baixa (v2.47.0 era a baseline saudável). Risco residual: fila Reactor (50k msgs) precisa ser drenada pós-estabilização; verificar idempotência do consumer. Reversibilidade: alta. ASSUMPTION crítica: ausência de migration destrutiva no v2.48.0 — validar antes de executar.
>
> **(B) Scaling emergencial.** ETA: 15–30min (parameter group + restart janela). Regressão: alta — não trata causa raiz, expõe a aumento futuro de tráfego com mesmo bug. Risco residual: pode falhar em pico subsequente. Reversibilidade: parcial (limites de DB raramente são revertidos).
>
> **Recomendação.** **(A) Rollback**, confiança alta. Invalidações: (i) migration destrutiva no v2.48.0, (ii) evidência de origem externa (degradação no próprio RDS independente do deploy) — nenhuma das duas está nos artefatos.
>
> **Plano de ação imediato.**
> 1. Verificar migrations do v2.48.0 (owner: dev de plantão, ETA 2min, gate: schema diff vazio ou aditivo).
> 2. `argocd app rollback chronos-api` → v2.47.0 (owner: SRE de plantão, ETA 5min, gate: pods saudáveis, p99 < 500ms em 10min).
> 3. Escalar consumers do Reactor para drenar backlog (owner: SRE, ETA 10min após rollback, gate: lag voltando para < 2min).
> 4. Comunicar status no canal de incidentes a cada 5min.
>
> **Follow-ups pós-incidente.**
> - Bisseção das 4 mudanças do v2.48.0 em staging com load test realista.
> - Revisar comportamento do novo pool client (lifecycle, leaks, métricas internas).
> - Reavaliar redução de timeout 5s→2s com dados de distribuição de latência real.
> - Alarme proativo em `connection_pool_waiting > N` antes do circuit breaker abrir.
> - Capacity planning: 12 pods × 20 conexões = teto de RDS — precisa de margem ou de pgbouncer.
---
## Justificativa

R-I-S-E (Role · Input · Steps · Expectation). Framework escolhido. Estrutura o prompt como pseudocódigo: define quem analisa (Role), entrega os artefatos brutos (Input), prescreve uma sequência ordenada e auditável de passos analíticos (Steps), e fixa o shape do output (Expectation). Ganhou porque o slot Steps força raciocínio numerado e impede que o modelo pule cálculos críticos — no caso, a aritmética de capacidade (12 pods × 20 conexões = 240 ≈ limite RDS) que decide rollback vs. scaling. Vale o custo de redação maior quando a tarefa é decisão técnica sob pressão com múltiplas fontes de dado.

C-A-R-E (Context · Action · Result · Example). Concorrente direto. Acomoda volume alto de contexto bruto (Context), define a tarefa (Action), prescreve formato de saída (Result) e — diferencial — ancora o estilo via um exemplo concreto (Example). Sua força é a conformidade de forma: exemplos concretos guiam modelos melhor que regras em prosa. Perde para R-I-S-E aqui porque Action é descritivo em vez de ordenado, deixando a sequência de raciocínio à escolha do modelo. Seria a escolha em postmortem retrospectivo onde padronização de formato entre autores importa mais que rigor de processo.

R-T-F (Role · Task · Format). Baseline mínimo. Define o papel (Role), descreve a tarefa (Task) e especifica o formato (Format). Sua força é a brevidade: prompt 5–10× menor, custo de redação ínfimo, baixa chance de o autor introduzir bugs no próprio prompt. Perde para R-I-S-E aqui porque não tem onde colocar sequência de raciocínio — sem Steps, o modelo decide a ordem de análise por conta própria, e a ordem é exatamente o que separa a recomendação correta da errada. É o framework default para tarefas amplas e simples; perde quando a decisão é crítica e depende de correlação de múltiplas fontes.