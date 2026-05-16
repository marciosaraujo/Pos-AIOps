## Prompt
```
# Role
Atue como um Engenheiro SRE / DevOps Sênior especialista em Automação e AWS.

# Task
Crie um script bash robusto para automatizar o backup diário de um banco PostgreSQL com as seguintes especificações:
Conexão: Host ledger-db.internal.hvt.io, Porta 5432, Banco ledger_prod, Usuário backup_user. A senha será injetada em tempo de execução via variável de ambiente $PGPASSWORD.
Ambiente Local: SO Ubuntu 22.04 LTS. Diretório de trabalho local /var/backups/ledger. O arquivo gerado deve ser extraído via pg_dump e compactado em tempo real usando gzip.
Destino e Retenção: Fazer upload do dump final para o bucket S3 s3://hvt-ledger-backups na região us-east-1 via aws s3 cp. Implemente uma lógica no script para apagar arquivos no S3 que tenham mais de 30 dias.
Log e Tratamento de Erros: Todo o fluxo (sucesso e falha) deve ser logado em /var/log/ledger-backup.log com timestamp de execução. O script deve interromper e retornar exit codes diferentes de 0 em caso de falhas no dump ou no upload.

# Format
Entregue exclusivamente o código do script bash comentado e pronto para uso em produção, seguido do comando de agendamento diário para o crontab.
```
---
## Modelo
```
Gemini 3.1 Pro
```
---
## Output
```
Script de Automação (Bash)


#!/usr/bin/env bash

# ==============================================================================
# Script de Backup PostgreSQL + S3 Upload
# ==============================================================================
# Garante que o script falhe caso qualquer comando falhe ou variável não exista.
# O pipefail assegura que a falha do pg_dump seja propagada através do gzip.
set -euo pipefail

# ------------------------------------------------------------------------------
# Variáveis de Configuração
# ------------------------------------------------------------------------------
DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_NAME="ledger_prod"
DB_USER="backup_user"

BACKUP_DIR="/var/backups/ledger"
LOG_FILE="/var/log/ledger-backup.log"

S3_BUCKET="s3://hvt-ledger-backups"
S3_REGION="us-east-1"
RETENTION_DAYS=30

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
FILE_NAME="${DB_NAME}_${TIMESTAMP}.sql.gz"
FILE_PATH="${BACKUP_DIR}/${FILE_NAME}"

# ------------------------------------------------------------------------------
# Funções Auxiliares
# ------------------------------------------------------------------------------
log_msg() {
    local msg="$1"
    echo "$(date +'%Y-%m-%d %H:%M:%S') - ${msg}" | tee -a "${LOG_FILE}"
}

# ------------------------------------------------------------------------------
# Validações Iniciais
# ------------------------------------------------------------------------------
if [[ -z "${PGPASSWORD:-}" ]]; then
    log_msg "ERRO: Variável PGPASSWORD não está definida no ambiente."
    exit 1
fi

mkdir -p "${BACKUP_DIR}"

# ------------------------------------------------------------------------------
# Execução do Backup (Dump + Compressão em tempo real)
# ------------------------------------------------------------------------------
log_msg "INFO: Iniciando dump do banco de dados '${DB_NAME}' no host '${DB_HOST}'..."

if pg_dump -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" -F p | gzip > "${FILE_PATH}"; then
    log_msg "SUCESSO: Dump e compressão concluídos. Arquivo: ${FILE_PATH}"
else
    log_msg "ERRO: Falha durante o pg_dump ou compressão."
    exit 2
fi

# ------------------------------------------------------------------------------
# Upload para o AWS S3
# ------------------------------------------------------------------------------
log_msg "INFO: Iniciando upload do arquivo para o S3 (${S3_BUCKET})..."

if aws s3 cp "${FILE_PATH}" "${S3_BUCKET}/" --region "${S3_REGION}" --quiet; then
    log_msg "SUCESSO: Upload para o S3 finalizado."
else
    log_msg "ERRO: Falha ao fazer upload para o S3."
    exit 3
fi

# ------------------------------------------------------------------------------
# Retenção no S3
# ------------------------------------------------------------------------------
log_msg "INFO: Iniciando limpeza de backups mais antigos que ${RETENTION_DAYS} dias no S3..."

# Calcula a data limite de corte no formato YYYY-MM-DD
CUTOFF_DATE=$(date -d "${RETENTION_DAYS} days ago" +%Y-%m-%d)

# Lista os arquivos, filtra os gz, avalia a data e executa o delete
aws s3 ls "${S3_BUCKET}/" --region "${S3_REGION}" | awk '/\.sql\.gz$/ {print $1, $4}' | while read -r S3_DATE S3_FILE; do
    if [[ "${S3_DATE}" < "${CUTOFF_DATE}" ]]; then
        log_msg "INFO: Excluindo backup expirado no S3: ${S3_FILE} (Data: ${S3_DATE})"
        aws s3 rm "${S3_BUCKET}/${S3_FILE}" --region "${S3_REGION}" --quiet
    fi
done

log_msg "SUCESSO: Rotina de backup finalizada sem erros."
exit 0

Agendamento Crontab
Bash
# Execute 'crontab -e' como root ou usuário com permissões nas pastas locais e adicione:
# Agendamento diário às 03:00 AM.

0 3 * * * PGPASSWORD="<insira_a_senha_aqui>" /usr/local/bin/backup_ledger.sh >> /var/log/ledger-backup-cron.log 2>&1

```
---
## Justificativa

O framework R-T-F evidencia que a definição do Role (Engenheiro SRE/DevOps Sênior) calibra o modelo para incluir boas práticas operacionais de segurança e estabilidade, enquanto a delimitação da Task com variáveis e requisitos técnicos estritos previne alucinações da inteligência artificial, e a restrição de Format garante a entrega exclusiva do código e do agendamento, eliminando verbosidade desnecessária para gerar um artefato direto e pronto para produção.
