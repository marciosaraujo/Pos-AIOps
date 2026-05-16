# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Sobre o repositório

Repositório acadêmico da Pós-Graduação em **AIOps e IA na Engenharia de Cloud**. Cada módulo contém exercícios de engenharia de prompt com respostas geradas por diferentes modelos de IA (Claude e Gemini), para fins de comparação e análise crítica.

## Estrutura e convenções

Os módulos são numerados e nomeados como `<número>. <Nome do Módulo>/`. Dentro de cada módulo, os arquivos seguem o padrão:

```
Questão <NN> - <Descrição do tema> - <Modelo>.md
```

Cada arquivo `.md` obedece à estrutura:
1. `## Prompt` — o prompt exato enviado ao modelo
2. `## Modelo` — identificação do modelo utilizado
3. `## Output` — resposta gerada pelo modelo
4. `## Justificativa` — análise do framework de prompt adotado e por que foi escolhido

## Frameworks de prompt estudados

| Framework | Estrutura | Melhor uso |
|---|---|---|
| **RTF** | Role · Task · Format | Tarefas objetivas com output delimitado |
| **RISE** | Role · Input · Steps · Expectation | Análises complexas com raciocínio sequencial obrigatório |
| **CARE** | Context · Action · Result · Example | Tarefas com requisito forte de conformidade de formato |

## Contexto de domínio

Os exercícios simulam um ambiente fictício com os seguintes componentes recorrentes:
- **Chronos API** — serviço Python/Flask em EKS, 6–12 réplicas via HPA
- **Lift** — aplicação Python/Flask containerizada
- **Ledger** — banco de dados PostgreSQL/RDS
- **Reactor** — fila de mensagens via SQS
- **Beacon** — sistema de observabilidade/alertas
- **Argo CD** — deploy contínuo via GitOps, repo `hvt/chronos-api`

## Adicionando novo conteúdo

Ao criar novos arquivos de questão, use a mesma estrutura de seções (`## Prompt`, `## Modelo`, `## Output`, `## Justificativa`) e nomeie os arquivos seguindo o padrão existente. Modelos ativos: `Claude Opus 4.7 Adaptativo` e `Gemini 3.1 Pro`.
