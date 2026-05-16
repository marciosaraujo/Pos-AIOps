
## Prompt 
```
#Role
Aja como um Engenheiro SRE/DevOps Sênior especialista em Kubernetes e segurança de containers

#Task
Escreva um Dockerfile seguindo todas as boas praticas, para conteinerizar a aplicação Python/Flask chamada \"Lift\", a API deve ter o nome app.py e deve estar no diretorio raiz que possui dependencias no arquivo requirements.txt, as variáveis DATABASE_URL e API_KEY precisam estar presentes no runtime. O serviço sobe com gunicorn na porta 8080 rodando 4 workers.

#Format
Retorne apenas o bloco de código contendo o Dockerfile devidamente comentado.
```
---

## Modelo
```
Gemini 3.1 Pro
```
---

## Output
```
# Estágio de Build: Isola ferramentas de compilação da imagem final para reduzir superfície de ataque
FROM python:3.11-slim-bookworm AS builder

# Evita a gravação de arquivos .pyc e força o log a ser enviado diretamente ao stdout/stderr
ENV PYTHONDONTWRITEBYTECODE=1 \\
    PYTHONUNBUFFERED=1

WORKDIR /build

# Copia apenas o arquivo de dependências primeiro para maximizar o uso de cache de camadas do Docker
COPY requirements.txt .

# Instala as dependências em um diretório específico (--prefix) sem armazenar cache do pip
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /build/wheels -r requirements.txt


# Estágio Final: Imagem de runtime enxuta e segura
FROM python:3.11-slim-bookworm

# Mantém as variáveis de otimização do Python
ENV PYTHONDONTWRITEBYTECODE=1 \\
    PYTHONUNBUFFERED=1

# Declara as variáveis de ambiente exigidas no runtime (valores reais devem ser injetados via Secrets/ConfigMaps no AKS/Rancher)
ENV DATABASE_URL=\"\" \\
    API_KEY=\"\"

# Criação de um usuário e grupo não-root para mitigar escalonamento de privilégios (Security Best Practice)
RUN groupadd -r liftgroup && useradd -r -g liftgroup liftuser

WORKDIR /app

# Copia os wheels compilados do estágio de build e instala
COPY --from=builder /build/wheels /wheels
COPY --from=builder /build/requirements.txt .
RUN pip install --no-cache /wheels/* && rm -rf /wheels

# Copia o código-fonte da aplicação para o diretorio raiz do container
COPY app.py .

# Transfere a propriedade dos arquivos para o usuário não-root
RUN chown -R liftuser:liftgroup /app

# Define o contexto de execução para o usuário restrito criado
USER liftuser

# Sinaliza a porta em que o gunicorn estará escutando
EXPOSE 8080

# Entrypoint via CMD executando o Gunicorn com 4 workers em porta não privilegiada
CMD [\"gunicorn\", \"--workers\", \"4\", \"--bind\", \"0.0.0.0:8080\", \"app:app\"]
```
---

## Justificativa

O uso do framework RTF (Role-Task-Format) na solicitação anterior garante precisão técnica e a entrega de um artefato imediatamente utilizável, requisitos críticos na orquestração de containers. Ao estipular o Role de Engenheiro SRE/DevOps Sênior, força-se a adoção de padrões avançados de segurança e otimização para Kubernetes, como a execução com privilégios restritos. A Task fornece as restrições arquiteturais exatas, englobando variáveis de ambiente, topologia estrita e concorrência do Gunicorn. Por fim, o Format exigindo apenas o bloco de código comentado elimina ruídos textuais e explicações redundantes. Essa estrutura resulta em um Dockerfile limpo, de fácil revisão e pronto para integração contínua, refletindo a objetividade e a necessidade de aplicações práticas exigidas em rotinas complexas de infraestrutura.
