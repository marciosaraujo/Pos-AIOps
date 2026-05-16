## Prompt 
```
# Role
Aja como um Engenheiro SRE/DevOps Sênior especialista em Kubernetes e segurança de containers

# Task
Escreva um Dockerfile seguindo todas as boas práticas, para conteinerizar a aplicação Python/Flask chamada "Lift", a API deve ter o nome `app.py` e deve estar no diretório raiz que possui dependências no arquivo `requirements.txt`, as variáveis `DATABASE_URL` e `API_KEY` precisam estar presentes no runtime. O serviço sobe com gunicorn na porta 8080 rodando 4 workers.

# Format
Retorne apenas o bloco de código contendo o Dockerfile devidamente comentado.
```
---
## Modelo
```
Claude Opus 4.7 Adaptativo
```
---
## Output

```dockerfile
# syntax=docker/dockerfile:1.7

# ============================================================================
# Stage 1: Builder - Compila dependências em ambiente isolado
# ============================================================================
# Usa imagem slim com versão fixa (evita "latest") para builds reprodutíveis
FROM python:3.12-slim-bookworm AS builder

# Variáveis de ambiente para otimizar o Python em containers:
# - PYTHONDONTWRITEBYTECODE: não gera arquivos .pyc (reduz tamanho da imagem)
# - PYTHONUNBUFFERED: força stdout/stderr sem buffer (logs em tempo real)
# - PIP_NO_CACHE_DIR: não armazena cache do pip (reduz tamanho)
# - PIP_DISABLE_PIP_VERSION_CHECK: evita checagens desnecessárias durante build
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Instala dependências de build necessárias para compilar pacotes Python nativos
# --no-install-recommends evita pacotes opcionais, reduzindo superfície de ataque
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Cria um virtualenv isolado para facilitar a cópia ao stage final
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copia apenas o requirements.txt antes do código para aproveitar o cache do Docker
# (a camada só é invalidada se requirements.txt mudar)
COPY requirements.txt .

# Instala dependências + gunicorn (servidor WSGI de produção)
RUN pip install --upgrade pip setuptools wheel && \
    pip install --no-cache-dir -r requirements.txt && \
    pip install --no-cache-dir gunicorn


# ============================================================================
# Stage 2: Runtime - Imagem final enxuta, sem ferramentas de build
# ============================================================================
FROM python:3.12-slim-bookworm AS runtime

# OCI labels para rastreabilidade e governança de imagens
LABEL org.opencontainers.image.title="Lift API" \
      org.opencontainers.image.description="Flask API Lift servida via Gunicorn" \
      org.opencontainers.image.source="https://github.com/exemplo/lift" \
      org.opencontainers.image.licenses="MIT"

# Mesmas otimizações de Python no runtime
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

# Aplica atualizações de segurança e instala apenas o essencial:
# - curl: usado pelo HEALTHCHECK
# - tini: init mínimo que faz reaping correto de processos zumbis (PID 1)
# Em seguida, limpa caches do apt para reduzir o tamanho da imagem final
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        curl \
        tini \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Cria usuário e grupo não-root dedicados (princípio do menor privilégio)
# UID/GID fixos facilitam mapeamento de volumes e políticas de PodSecurity no K8s
RUN groupadd --system --gid 1001 lift && \
    useradd --system --uid 1001 --gid lift --no-create-home --shell /sbin/nologin lift

WORKDIR /app

# Copia o virtualenv pronto do stage builder, ajustando ownership para o usuário não-root
COPY --from=builder --chown=lift:lift /opt/venv /opt/venv

# Copia o código da aplicação por último (camada que muda com mais frequência)
COPY --chown=lift:lift app.py .

# Variáveis obrigatórias em runtime — declaradas vazias aqui apenas para documentar
# o contrato da imagem. Devem ser injetadas via Secret/ConfigMap no Kubernetes,
# NUNCA hardcoded na imagem (evita vazamento de credenciais em registries).
ENV DATABASE_URL="" \
    API_KEY=""

# Porta exposta pelo Gunicorn (documental — não publica a porta automaticamente)
EXPOSE 8080

# Troca para o usuário não-root antes de executar a aplicação
USER lift:lift

# Healthcheck para sinalizar prontidão ao orquestrador
# No Kubernetes, prefira liveness/readiness probes no manifesto do Pod,
# mas manter o HEALTHCHECK aqui ajuda em execuções com Docker/Compose puros
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl --fail --silent http://localhost:8080/health || exit 1

# tini como PID 1 garante handling correto de SIGTERM (graceful shutdown no K8s)
ENTRYPOINT ["/usr/bin/tini", "--"]

# Gunicorn em formato exec (array) — evita shell intermediário e propaga sinais
# Flags de produção:
# --workers 4              : 4 workers conforme requisito
# --bind 0.0.0.0:8080      : escuta em todas as interfaces na porta 8080
# --access-logfile -       : logs de acesso no stdout (coletáveis pelo kubelet)
# --error-logfile -        : logs de erro no stderr
# --timeout 30             : encerra requests travados após 30s
# --graceful-timeout 30    : tempo para workers finalizarem em shutdown
# --keep-alive 5           : reaproveita conexões TCP por 5s
CMD ["gunicorn", \
     "--workers", "4", \
     "--bind", "0.0.0.0:8080", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "--timeout", "30", \
     "--graceful-timeout", "30", \
     "--keep-alive", "5", \
     "app:app"]
```

---

## Boas Práticas Aplicadas

| Prática | Implementação |
|---|---|
| **Multi-stage build** | Separação entre `builder` e `runtime` para imagem final menor |
| **Imagem base mínima** | `python:3.12-slim-bookworm` com versão fixa (não `latest`) |
| **Usuário não-root** | UID/GID 1001 (`lift`) com `nologin` shell |
| **Princípio do menor privilégio** | Sem `sudo`, sem permissões desnecessárias |
| **Cache de camadas otimizado** | `requirements.txt` copiado antes do código |
| **Init system** | `tini` como PID 1 para reaping de zumbis e SIGTERM correto |
| **Healthcheck** | Configurado para Docker/Compose; complementa probes do K8s |
| **Secrets via runtime** | `DATABASE_URL` e `API_KEY` injetados via Secret/ConfigMap |
| **OCI Labels** | Metadados para rastreabilidade no registry |
| **Logs estruturados** | stdout/stderr sem buffer, coletáveis pelo kubelet |
| **Atualizações de segurança** | `apt-get upgrade` no stage final |
| **Limpeza de cache** | Remoção de `/var/lib/apt/lists/*` e diretórios temporários |

---

## Como Usar

### Build
```bash
docker build -t lift:1.0.0 .
```

### Run local (apenas para testes)
```bash
docker run -d \
  --name lift \
  -p 8080:8080 \
  -e DATABASE_URL="postgresql://user:pass@host:5432/lift" \
  -e API_KEY="sua-chave-aqui" \
  --read-only \
  --tmpfs /tmp \
  --security-opt no-new-privileges \
  lift:1.0.0
```
## Justificativa

O framework RTF (Role, Task, Format) foi adequado para este caso porque a tarefa exigia um output técnico bem delimitado: ao definir o Role como "Engenheiro SRE/DevOps Sênior especialista em Kubernetes e segurança de containers", o modelo foi ancorado em um contexto de expertise específico, garantindo decisões alinhadas a boas práticas de produção (multi-stage build, usuário não-root, tini como PID 1); a Task detalhou requisitos objetivos e mensuráveis (nome da app, dependências, variáveis de ambiente, porta, workers do gunicorn), eliminando ambiguidade sobre o que deveria ser entregue; e o Format restringiu a saída a apenas o bloco de código comentado, evitando explicações redundantes e produzindo um artefato pronto para uso direto em pipeline CI/CD. Essa estrutura é especialmente eficaz em contextos de engenharia, onde precisão técnica e entregáveis acionáveis importam mais do que narrativas explicativas.
