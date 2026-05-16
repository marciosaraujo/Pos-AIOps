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
Gemini 3.1 Pro
```
---
## Output

**Resumo Financeiro:**
* **Custo Mensal Total:** USD 41.800
* **Meta de Redução (15%):** USD 6.270
* **Target Mensal:** USD 35.530

### Tabela de Oportunidades Priorizadas

| Prioridade | Serviço | Ação Recomendada | Economia Est. (USD) | Impacto (%) | Esforço | Riscos e Pré-requisitos |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | **EC2 On-demand** | Migração para Savings Plans ou Spot Instances (uso médio atual de apenas 45%). | ~$2.400 | 5,7% | Médio | Requer análise de resiliência de workloads stateless. |
| **2** | **RDS PostgreSQL** | Aquisição de Reserved Instances (RI) de 1 ano. | ~$1.600 | 3,8% | Baixo | Compromisso de longo prazo com o tipo de instância. |
| **3** | **CloudWatch Logs** | Ajustar retenção de 90 para 30 dias; mover logs frios para S3 Glacier. | ~$1.400 | 3,3% | Baixo | Alinhamento com políticas de compliance e auditoria. |
| **4** | **S3 Standard** | Implementação de Lifecycle Policies para S3-IA/Glacier. | ~$1.200 | 2,8% | Baixo | Análise do padrão de recuperação de dados. |
| **5** | **EKS** | Consolidação de clusters ou uso de Karpenter para right-sizing de nós. | ~$1.000 | 2,3% | Alto | Risco de downtime durante migração de pods; requer testes. |

**Conclusão:**
A meta de 15% é perfeitamente alcançável. As ações propostas somam aproximadamente **18,1% de economia (USD 7.600)**, permitindo uma margem de segurança para o SLA de produção.


---
## Justificativa

O uso do framework T-A-G se justifica por estruturar a análise técnica em três pilares fundamentais: a definição da autoridade e do contexto de engenharia FinOps na fase Task garante o foco em boas práticas de infraestrutura; a fase Action delimita o processo analítico, exigindo cálculos específicos e métricas de priorização que transformam os dados brutos em inteligência de negócio; e, por fim, a fase Goal estabelece o formato de entrega como uma tabela executiva direcionada à diretoria, o que assegura um tom profissional e torna as informações fáceis de consumir.

