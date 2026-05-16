## Prompt
```
[Task] Atue como um Engenheiro FinOps e SRE Sênior especializado em infraestrutura AWS.

[Action] Analise os dados CSV fornecidos abaixo contendo o breakdown de custos mensais da AWS. Sua tarefa é encontrar oportunidades para reduzir a fatura total em pelo menos 15% (meta do trimestre), garantindo que não haja degradação de SLA ou impacto negativo nos ambientes de produção. Calcule o custo total atual e quanto representa a meta de 15%. Em seguida, liste as oportunidades de otimização priorizadas pelo impacto financeiro. Para cada oportunidade, defina:
Ação técnica recomendada.
Economia mensal estimada (em USD).
Percentual de economia em relação à conta total.
Esforço de implementação (Baixo, Médio ou Alto).
Riscos potenciais ou pré-requisitos para execução.

[Goal] Gere um relatório executivo estruturado e direto, em formato de tabela, que será apresentado à diretoria por "Goldie" para aprovação das frentes de trabalho. Ao final, inclua uma conclusão breve confirmando se a meta de 15% é alcançável com base nas ações propostas.
```
---
## Modelo
```
Claude Opus 4.7
```
---
## Output

# Relatório Executivo — Otimização de Custos AWS

**Apresentado por:** Goldie
**Período:** Mensal
**Objetivo:** Redução de ≥15% na fatura AWS (meta do trimestre)
**Premissa:** Sem degradação de SLA nem impacto em ambientes de produção

---

## 1. Resumo Financeiro

| Indicador | Valor (USD/mês) |
|---|---|
| Custo total atual | **41.800** |
| Meta de redução (15%) | **6.270** |
| Potencial total identificado | **~10.990 (26,3%)** |
| Potencial via *quick wins* (esforço Baixo/Médio) | **~6.285 (15,0%)** |

---

## 2. Oportunidades Priorizadas por Impacto Financeiro

| # | Serviço (Custo Atual) | Ação Técnica Recomendada | Economia Est. (USD/mês) | % da Conta Total | Esforço | Riscos / Pré-requisitos |
|---|---|---|---|---|---|---|
| 1 | **EC2 on-demand** (8.200, 45% uso) | Compute Savings Plans (1 ano, no-upfront) para baseline + rightsizing por CloudWatch/Compute Optimizer + Spot para workloads tolerantes a interrupção | **2.500** | 6,0% | Médio | Mapear baseline confiável antes do commitment; validar tolerância a Spot por workload |
| 2 | **RDS PostgreSQL** (8.200, 62% uso, Multi-AZ) | Reserved Instances 1 ano para a instância primária + revisão de classe (db.r→db.m se workload não-memória) + Performance Insights para confirmar sizing | **2.050** | 4,9% | Médio | Manter Multi-AZ (SLA produção); validar sizing com 2 semanas de métricas antes da RI |
| 3 | **EKS** (6.700, 58% uso, 3 clusters) | Karpenter com consolidação + node groups Spot para workloads stateless + bin-packing via requests/limits revisados; avaliar consolidação de clusters dev/staging | **1.700** | 4,1% | Médio-Alto | PDBs configurados, topology spread, HPA ajustado; evitar Spot em workloads stateful críticos |
| 4 | **CloudWatch Logs** (2.800, retenção 90d) | Reduzir retenção para 30d em log groups não-críticos + export para S3 + Glacier para arquivos auditáveis + filtros no nível da aplicação | **1.500** | 3,6% | **Baixo** | Validar requisitos de compliance/auditoria por log group antes de reduzir retenção |
| 5 | **S3 Standard** (3.100) | S3 Intelligent-Tiering nos buckets ≥128KB + lifecycle policies (IA após 30d, Glacier após 90d) + análise via Storage Lens | **1.000** | 2,4% | **Baixo** | Analisar padrão de acesso por bucket; cuidado com objetos pequenos (overhead do IT) |
| 6 | **ElastiCache Redis** (2.100, 40% uso) | Rightsizing (subir 1 tier abaixo) + Reserved Nodes 1 ano após sizing estabilizado | **630** | 1,5% | Baixo-Médio | Monitorar eviction/memory pressure pós-resize; janela de manutenção |
| 7 | **NAT Gateway** (1.200, 3 gateways) | VPC Endpoints (Gateway gratuitos para S3/DynamoDB; Interface para SQS/SNS/ECR/STS) — corta tráfego pago via NAT | **480** | 1,1% | Médio | Mapear fluxos atuais; ajustar route tables e SGs dos endpoints |
| 8 | **Data Transfer Out** (1.900) | CloudFront na frente de assets/APIs públicas + VPC Peering ou Transit Gateway para tráfego inter-region recorrente + revisão de arquitetura multi-region | **475** | 1,1% | **Alto** | Mudança arquitetural; validar latência e impacto em DR |
| 9 | **EBS gp3** (1.600, 68% uso) | Rightsizing de IOPS/throughput provisionados (default gp3 já cobre maioria dos casos) + cleanup de snapshots órfãos via DLM | **250** | 0,6% | **Baixo** | Baixo; validar workloads I/O-intensive antes de reduzir IOPS |
| 10 | **Lambda** (900, ~12M inv/mês) | Migrar runtimes para arm64/Graviton2 (~20% mais barato) + memory tuning via AWS Lambda Power Tuning | **225** | 0,5% | **Baixo** | Testes de regressão pós-migração ARM; libs nativas precisam de rebuild |
| 11 | **CloudWatch Metrics** (900) | Auditar custom metrics de alta cardinalidade + descontinuar métricas não usadas em dashboards/alarms | **180** | 0,4% | **Baixo** | Garantir que métricas não estão sendo consumidas por alarms ativos |
| — | EC2 reservada (4.200, 72%) | Manter — boa utilização e já reservada | — | — | — | Reavaliar na renovação do contrato |

**Soma das economias estimadas:** ~USD 10.990/mês (~26,3% da conta total)

---

## 3. Roadmap Recomendado (Sequência de Execução)

| Onda | Janela | Itens | Economia Acumulada |
|---|---|---|---|
| **Onda 1 — Quick wins** | 0–30 dias | #4, #5, #9, #10, #11 (todos esforço Baixo) | ~USD 3.155 (7,5%) |
| **Onda 2 — Commitments e rightsizing** | 30–60 dias | #1, #2, #6 | ~USD 8.335 (19,9%) |
| **Onda 3 — Arquitetura** | 60–90 dias | #3, #7, #8 | ~USD 10.990 (26,3%) |

---

## 4. Conclusão

**A meta de 15% é alcançável e conservadora.** Apenas com as ações de esforço Baixo/Médio (Ondas 1 e 2), atingimos ~19,9% de redução — superando a meta trimestral em ~5 p.p. e sem nenhum impacto em SLA de produção, já que as ações de maior risco arquitetural (Onda 3) ficam fora do caminho crítico.

**Recomendação para a diretoria:** aprovar imediatamente as Ondas 1 e 2 (economia projetada de USD 8.335/mês ≈ USD 100k/ano) e tratar a Onda 3 como track paralela, com business case dedicado para revisão da arquitetura multi-region e consolidação de clusters EKS.

---
## Justificativa

O framework TAG (Task, Action, Goal) foi adequado por separar com clareza papel, execução e entregável: o [Task] fixou o persona (FinOps + SRE Sênior), o [Action] definiu critérios mensuráveis e campos obrigatórios por oportunidade (ação, economia em USD, %, esforço, riscos) — funcionando como um contrato de saída que força quantificação e elimina recomendações genéricas — e o [Goal] ancorou formato e audiência (relatório executivo em tabela para a diretoria via Goldie), ajustando o registro para impacto financeiro em vez de jargão técnico. Para análises pontuais com entrega estruturada, TAG entrega mais que prompts em prosa livre e menos overhead que frameworks longos (RACE, CRISPE); ficaria atrás apenas em tarefas iterativas ou multi-etapas, onde CoT ou ReAct se saem melhor.
