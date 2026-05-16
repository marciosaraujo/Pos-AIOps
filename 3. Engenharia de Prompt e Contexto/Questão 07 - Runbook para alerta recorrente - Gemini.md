## Prompt
```
# Role
Você é um SRE sênior com 10+ anos operando workloads críticos em Kubernetes/EKS, especialista em diagnóstico de incidentes de memória e em redação de runbooks operacionais. Escreve direto, sem rodeios, para plantonistas que executam sob pressão sem conhecer o sistema previamente.

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
Gemini 3.1 Pro
```
---
## Output
```
TL;DR / Quando Usar
Aplica-se a: Alerta exato [CRITICAL] High memory usage on Chronos API pods (>85% for 10min). Indica saturação de memória nos pods da aplicação.
NÃO se aplica a: Alertas isolados de CPU, perda de conectividade de rede do node, ou alertas de OOMKilled ocorrendo instantaneamente sem o tempo de maturação de 10 minutos (neste caso, indica falha catastrófica e requer intervenção de plataforma).

Comunicação Inicial
Ao assumir o alerta, poste imediatamente a seguinte mensagem no canal #oncall-chronos no Slack:

"Assumindo alerta [CRITICAL] High memory usage on Chronos API. Iniciando triagem em produção."

Passos de Diagnóstico
1. Verificar Deploy Recente (Tempo estimado: 1 min)
Objetivo: Identificar se uma alteração recente de código causou vazamento de memória ou uso excessivo.

Bash
argocd app history chronos-api
O que observar: O timestamp da última REVISION com STATUS=Succeeded.

Decisão: Se o deploy ocorreu nos últimos 30 minutos → Execute Mitigação B (Rollback). Senão → Próximo passo.

2. Avaliar Estado do HPA (Tempo estimado: 2 min)
Objetivo: Determinar se o tráfego está saturando as réplicas ou se o HPA está travado limitando o escalonamento horizontal.

Bash
kubectl get hpa chronos-api -n production
O que observar: A coluna REPLICAS e TARGETS.

Decisão: Se REPLICAS atingiu 12 (máximo) e TARGETS de CPU estiver abaixo de 70% → Execute Mitigação C (Ajuste de HPA). Senão → Próximo passo.

3. Analisar Distribuição de Memória (Tempo estimado: 2 min)
Objetivo: Diferenciar entre um vazamento de memória isolado em pods antigos e um pico sistêmico.

Bash
kubectl top pods -n production -l app=chronos-api --sort-by=memory
O que observar: O consumo de memória reportado em Mi por cada pod individualmente.

Decisão: Se apenas 1 ou 2 pods estiverem isolados acima de 85% e os demais estáveis → Execute Mitigação A (Restart Controlado). Se todos estiverem crescendo uniformemente e próximos de 85% → Próximo passo.

4. Checar Backpressure do SQS / Reactor (Tempo estimado: 2 min)
Objetivo: Confirmar se o acúmulo de mensagens não processadas está inflando as filas em memória da aplicação.

Bash
kubectl logs -n production -l app=chronos-api --tail=100 | grep -i -E "sqs|reactor|timeout|backpressure"
O que observar: Ocorrências repetidas de timeout, Connection reset by peer ou SQS limit exceeded.

Decisão: Se houver erros contínuos apontando para o Reactor/SQS → Acione escalação (@chronos-core). Senão → Próximo passo.

5. Investigar Conexões com PostgreSQL / Ledger (Tempo estimado: 3 min)
Objetivo: Verificar se queries lentas estão segurando conexões ativas e retendo grandes objetos em memória no backend.

Bash
kubectl logs -n production -l app=chronos-api --tail=100 | grep -i -E "database|ledger|pool|exhausted"
O que observar: Mensagens como connection pool exhausted, lock timeout ou demoras anômalas em transações.

Decisão: Se houver evidências de gargalo no Ledger → Acione escalação (@chronos-core). Senão, a causa provável é degradação de memória em longo prazo (Memory Leak) → Execute Mitigação A (Restart Controlado).

Mitigações
Mitigação A: Restart Controlado (Limpeza de Memória)
Inicia uma substituição progressiva dos pods, limpando o uso de memória sem derrubar o serviço.

Bash
kubectl rollout restart deployment chronos-api -n production
kubectl rollout status deployment chronos-api -n production
Mitigação B: Rollback via Argo CD
Reverte o deploy para a versão estável anterior. Requer sincronização imediata.

Bash
argocd app rollback chronos-api
argocd app sync chronos-api
Mitigação C: Ajuste Manual de HPA
Aumenta temporariamente o teto do HPA para acomodar tráfego anômalo que não está estourando a CPU, mas está estourando a memória.

Bash
kubectl patch hpa chronos-api -n production --patch '{"spec":{"maxReplicas": 20}}'
Critérios de Escalação
O evento exige escalação imediata marcando @chronos-core no canal #oncall-chronos caso qualquer das condições numéricas abaixo seja atendida:

Tempo total de triagem e mitigação no runbook ultrapassar 15 minutos sem redução de uso de memória.

Contagem de restarts (RESTARTS em kubectl get pods) de qualquer pod exceder 3 vezes consecutivas em uma janela de 5 minutos (indicativo de CrashLoopBackOff ou OOMKilled severo).

Logs apresentarem mais de 50 falhas de conexão com Ledger (PostgreSQL) ou Reactor (SQS) nos últimos 5 minutos.

Comando de rollback (Mitigação B) retornar falha ou o Argo CD reportar status Degraded na aplicação.

Critérios de Encerramento
O alerta pode ser considerado resolvido e o ticket fechado quando:

O uso de memória em 100% dos pods da chronos-api permanecer abaixo de 70% (limite seguro) durante 15 minutos ininterruptos.

A métrica kube_pod_container_status_restarts_total for igual a 0 nos últimos 15 minutos.

Procedimentos Pós-Incidente
Ao finalizar as ações de mitigação ou escalação:

Extrair os últimos logs dos pods problemáticos antes de expurgá-los, direcionando a saída para arquivo.

Registrar no sistema de tickets interno (Beacon/Jira) o horário exato da ocorrência, o gatilho provável e as mitigações aplicadas.

Classificar o ticket com as seguintes tags obrigatórias: [chronos-api], [memory-leak], [action-taken].

Avisar no canal #oncall-chronos: "Alerta normalizado. Memória estabilizada. Detalhes no ticket de plantão."
```
---
## Justificativa

O uso do framework R-I-S-E justifica-se por eliminar ambiguidades e garantir precisão na criação de documentos operacionais críticos, atuando em quatro frentes estruturais. O Role impõe uma postura técnica e pragmática na redação; o Input restringe o escopo ao ambiente exato do incidente, bloqueando informações genéricas; os Steps definem um fluxo de diagnóstico lógico e sequencial, vital para decisões rápidas sob pressão; e, por fim, a Expectation funciona como um contrato rígido de qualidade que exige comandos exatos, formatação estrita e limiares numéricos, garantindo a entrega de um material padronizado e imediatamente acionável durante um plantão.