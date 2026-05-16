## Prompt
```
# Role
Você é um SRE sênior com 10+ anos operando workloads críticos em Kubernetes/EKS,
especialista em diagnóstico de incidentes de memória e em redação de runbooks
operacionais. Escreve direto, sem rodeios, para plantonistas que executam sob
pressão sem conhecer o sistema previamente.

# Input
Contexto do incidente recorrente:
- Alerta: `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`
  disparado pelo Beacon ~4x/semana
- Tempo atual de resolução: 30–40min, alta variância por falta de documentação
- Objetivo: runbook que QUALQUER plantonista execute de ponta a ponta sem
  escalar prematuramente

Ambiente:
- EKS, namespace `production`
- Chronos API, 6 réplicas, HPA (min 4, max 12, CPU target 70%)
- Deploy via Argo CD, repo `hvt/chronos-api`
- Dependências: Ledger (PostgreSQL), Reactor (SQS)
- Observabilidade: `/metrics`, logs no Beacon, dashboards no Grafana
- Tooling do plantão: `kubectl`, `aws cli`, `argocd cli`
- Canal: `#oncall-chronos` no Slack
- Escalação: `@chronos-core` (SLA 15min comercial / 30min fora)

# Steps
Produza o runbook nesta ordem:
1. Abra com um bloco "TL;DR / Quando usar" identificando o alerta exato e
   quando este runbook NÃO se aplica.
2. Inclua a mensagem padrão a postar em `#oncall-chronos` ao assumir.
3. Estruture o diagnóstico em passos numerados e sequenciais. Cada passo deve ter:
   - Objetivo em uma frase
   - Comando(s) exato(s) com valores reais (namespace, selector), sem placeholders
   - O que observar na saída (verificação esperada)
   - Decisão condicional: se padrão X → próximo passo; se padrão Y → mitigação
4. Cubra a árvore de causas mais prováveis: deploy recente quebrado, vazamento
   de memória aplicacional, queries lentas no Ledger, backpressure do Reactor,
   HPA travado.
5. Inclua mitigações imediatas: restart controlado, rollback via Argo CD,
   ajuste manual do HPA — com os comandos.
6. Defina critérios objetivos de escalação para `@chronos-core` com THRESHOLDS
   NUMÉRICOS (não "se ficar complicado").
7. Defina critérios objetivos de encerramento (métricas em baseline por X min).
8. Encerre com seção de pós-incidente: o que registrar, onde, e tags para correlação.

# Expectation
- Markdown; `##` para seções, `###` para subpassos
- Comandos em blocos ```bash, com comentários quando a flag não for óbvia
- Tempo estimado por passo (minutos)
- Tom imperativo direto ("Rode X", "Verifique Y") — nunca condicional
- Nenhum passo pode depender de conhecimento tácito; contexto necessário em 1 linha
- Critérios de escalação e encerramento com THRESHOLDS NUMÉRICOS
- Tamanho alvo: 600–900 palavras. Cortar tudo fora desse orçamento
- NÃO incluir teoria sobre por que memória sobe — só o que fazer

Critério de sucesso: plantonista que nunca abriu `hvt/chronos-api` consegue,
lendo só este runbook, resolver em até 20 minutos ou escalar corretamente.
```
---
## Modelo
```
Claude Opus 4.7 Adaptativo 
```
---
## Output
## Runbook — High Memory Usage on Chronos API pods

### TL;DR — Quando usar
Use quando `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`
disparar em `#oncall-chronos`. NÃO use para alertas de CPU, latência ou 5xx —
runbooks próprios.

### Passo 0 — Comunicação inicial (1 min)
Poste em `#oncall-chronos`:
> 🟡 Assumindo alerta de memória alta no Chronos. Investigando — atualizo em 10min ou antes se escalar.

Inicie timer de 20min. Excedeu sem mitigação? Escale (ver seção).

### Passo 1 — Estado atual dos pods (2 min)
```bash
kubectl -n production top pods -l app=chronos-api --sort-by=memory
kubectl -n production get pods -l app=chronos-api -o wide
```
**Esperado:** lista ordenada por memória; anote nº de pods >85% e idade.
**Decisão:**
- Todos >85% → Passo 2 (sistêmico)
- 1–2 pods >85% → Passo 3 (pontual)

### Passo 2 — HPA (2 min)
```bash
kubectl -n production get hpa chronos-api
kubectl -n production describe hpa chronos-api | tail -30
```
**Esperado:** `REPLICAS` entre 4 e 12; `ScalingActive=True`.
**Decisão:**
- replicas=12 → HPA no teto → Passo 4
- replicas<12 e não sobe → HPA travado → Passo 5
- escalando normal → Passo 4

### Passo 3 — Pod individual (3 min)
```bash
POD=<nome-do-pod>
kubectl -n production describe pod $POD | grep -A5 "Last State\|Events"
kubectl -n production logs $POD --tail=200 --since=15m | grep -iE "oom|out of memory|panic"
```
**Esperado:** `Last State: Terminated, Reason: OOMKilled` indica OOM recente.
**Decisão:**
- OOMKilled recorrente → Passo 6 (restart)
- Sem OOM, memória crescente → Passo 4 (suspeita de leak)

### Passo 4 — Deploy recente (3 min)
```bash
argocd app get chronos-api
argocd app history chronos-api | head -10
```
**Esperado:** identifique timestamp e revision do último deploy; compare com o início do alerta no Beacon.
**Decisão:**
- Deploy <60min antes do alerta → Passo 7 (rollback)
- Deploy >24h antes → Passo 5

### Passo 5 — Dependências (4 min)
**Ledger (Postgres):** abra Grafana → dashboard `Chronos / Ledger latency`.
Threshold: P99 < 500ms e conexões < 80% do pool.

**Reactor (SQS):**
```bash
aws sqs get-queue-attributes \
  --queue-url <chronos-inbound-url> \
  --attribute-names ApproximateNumberOfMessages ApproximateAgeOfOldestMessage
```
**Esperado:** `Messages < 10.000` e `AgeOfOldest < 300s`.
**Decisão:**
- SQS acumulando → Chronos buferizando em memória → Passo 6
- Ledger lento → registre e siga para Passo 6

### Passo 6 — Mitigação: restart controlado (3 min)
```bash
kubectl -n production rollout restart deployment/chronos-api
kubectl -n production rollout status deployment/chronos-api --timeout=5m
```
**Esperado:** pods novos em `Running`/`Ready`; após 3 min, memória <70%.

### Passo 7 — Mitigação: rollback via Argo CD (3 min)
Use só se Passo 4 apontou deploy recente.
```bash
argocd app rollback chronos-api <revision-anterior>
argocd app wait chronos-api --health --timeout 300
```

### Escalação para `@chronos-core`
Escale IMEDIATAMENTE se qualquer:
- Tempo total > 20 min sem mitigação efetiva
- OOMKilled em ≥3 pods nos últimos 10 min após Passo 6
- HPA com `ScalingActive=False` e Passo 5 sem causa identificada
- Rollback (Passo 7) não estabilizou em 5 min
- SQS `AgeOfOldestMessage > 600s`

```
@chronos-core P1: memória Chronos. Passos executados: [X, Y, Z]. Achados: [...]
```
SLA: 15min comercial / 30min fora.

### Encerramento
Encerre quando TODAS forem verdadeiras:
- `kubectl top pods` < 70% em todos os pods
- Alerta no Beacon `resolved` por ≥15 min
- HPA com contagem constante por ≥10 min
- Zero `OOMKilled` nos últimos 15 min

Poste:
> ✅ Encerrado. Causa: [X]. Mitigação: [Y]. Ticket: [link].

### Pós-incidente
- Ticket no board `Chronos/Incidents`, tag `memory-alert`
- Anexar: saída do Passo 1, decisão em cada bifurcação, timestamps
- ≥5 ocorrências na semana → abrir issue em `hvt/chronos-api` com label `tech-debt`
---
## Justificativa

O framework funciona porque cada elemento resolve um problema distinto e ortogonal: o Role define a voz (SRE sênior, imperativo, sem teoria didática), o Input fornece o contexto factual específico do sistema (namespace, HPA, repo, ferramentas, SLA da escalação) que evita comandos genéricos com placeholders, os Steps travam a estrutura do runbook na ordem certa (TL;DR, comunicação, diagnóstico ramificado, mitigação, escalação, encerramento, pós-incidente) para que nenhuma fase seja esquecida ou misturada, e a Expectation define formato e qualidade mensurável (markdown, thresholds numéricos, 600–900 palavras, critério de sucesso testável). Remover qualquer um degrada o output de forma previsível, e é isso que torna o R-I-S-E útil também como checklist de revisão do próprio prompt.