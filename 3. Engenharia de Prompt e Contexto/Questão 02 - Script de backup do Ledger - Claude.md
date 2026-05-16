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
Claude Opus 4.7 Adaptativo
```
---
## Output
```
#!/usr/bin/env bash
#
# ledger-backup.sh
# Backup diário do PostgreSQL ledger_prod com upload para S3 e retenção de 30 dias.
# Ambiente: Ubuntu 22.04 LTS
# Requisitos: postgresql-client (>=14), awscli v2, gzip
# Auth DB: senha injetada em runtime via variável de ambiente PGPASSWORD
# Auth AWS: IAM Role da instância OU credenciais via ~/.aws/credentials / env vars
#

set -euo pipefail

# ============================================================================
# CONFIGURAÇÕES
# ============================================================================
readonly PG_HOST="ledger-db.internal.hvt.io"
readonly PG_PORT="5432"
readonly PG_DB="ledger_prod"
readonly PG_USER="backup_user"

readonly LOCAL_DIR="/var/backups/ledger"
readonly LOG_FILE="/var/log/ledger-backup.log"

readonly S3_BUCKET="hvt-ledger-backups"
readonly AWS_REGION="us-east-1"
readonly RETENTION_DAYS=30

readonly TIMESTAMP="$(date -u +'%Y%m%dT%H%M%SZ')"
readonly DUMP_FILE="${LOCAL_DIR}/ledger_prod_${TIMESTAMP}.sql.gz"

# Connect timeout para evitar travamento indefinido caso o DB esteja inacessível
export PGCONNECT_TIMEOUT=15

# ============================================================================
# FUNÇÕES UTILITÁRIAS
# ============================================================================
log() {
    # Loga em stdout e em $LOG_FILE com timestamp UTC
    local level="$1"; shift
    printf '[%s] [%s] %s\n' "$(date -u +'%Y-%m-%d %H:%M:%S UTC')" "${level}" "$*" \
        | tee -a "${LOG_FILE}"
}

die() {
    # Encerra o script com log de erro e exit code específico (não-zero)
    local exit_code="$1"; shift
    log "ERROR" "$*"
    log "ERROR" "Backup ABORTADO com exit code ${exit_code}."
    exit "${exit_code}"
}

check_prereqs() {
    # Garante que diretórios e dependências críticas existem antes de iniciar
    mkdir -p "${LOCAL_DIR}"
    mkdir -p "$(dirname "${LOG_FILE}")"
    touch "${LOG_FILE}"

    command -v pg_dump >/dev/null 2>&1 || die 10 "pg_dump não encontrado (apt install postgresql-client)."
    command -v gzip    >/dev/null 2>&1 || die 11 "gzip não encontrado."
    command -v aws     >/dev/null 2>&1 || die 12 "AWS CLI não encontrado (apt install awscli)."

    # PGPASSWORD DEVE vir do ambiente (systemd EnvironmentFile, Vault, SSM, etc.)
    [[ -n "${PGPASSWORD:-}" ]] || die 13 "Variável PGPASSWORD não definida no ambiente."
}

# ============================================================================
# 1) DUMP + COMPRESSÃO EM STREAMING
# ============================================================================
do_dump() {
    log "INFO" "Iniciando pg_dump de ${PG_DB}@${PG_HOST}:${PG_PORT}"
    log "INFO" "Destino local: ${DUMP_FILE}"

    # pipefail garante que falha no pg_dump (lado esquerdo do pipe) propague o erro
    # gzip -9 maximiza compressão; ajuste para -6 se CPU for restrição.
    set -o pipefail
    if ! pg_dump \
            --host="${PG_HOST}" \
            --port="${PG_PORT}" \
            --username="${PG_USER}" \
            --dbname="${PG_DB}" \
            --no-owner \
            --no-privileges \
            --format=plain \
            --verbose 2>>"${LOG_FILE}" \
        | gzip -9 > "${DUMP_FILE}"
    then
        rm -f "${DUMP_FILE}"
        die 20 "Falha no pg_dump / gzip do banco ${PG_DB}."
    fi

    # Sanity check: dump não pode ser vazio
    if [[ ! -s "${DUMP_FILE}" ]]; then
        rm -f "${DUMP_FILE}"
        die 21 "Arquivo de dump foi gerado VAZIO: ${DUMP_FILE}"
    fi

    # Valida integridade do gzip antes de subir
    if ! gzip -t "${DUMP_FILE}" 2>>"${LOG_FILE}"; then
        rm -f "${DUMP_FILE}"
        die 22 "Arquivo gzip corrompido: ${DUMP_FILE}"
    fi

    log "INFO" "Dump concluído com sucesso ($(du -h "${DUMP_FILE}" | cut -f1))."
}

# ============================================================================
# 2) UPLOAD PARA S3
# ============================================================================
do_upload() {
    local s3_uri="s3://${S3_BUCKET}/$(basename "${DUMP_FILE}")"
    log "INFO" "Iniciando upload para ${s3_uri}"

    if ! aws s3 cp "${DUMP_FILE}" "${s3_uri}" \
            --region "${AWS_REGION}" \
            --storage-class STANDARD_IA \
            --only-show-errors 2>>"${LOG_FILE}"
    then
        die 30 "Falha no upload para ${s3_uri}."
    fi

    log "INFO" "Upload concluído: ${s3_uri}"
}

# ============================================================================
# 3) RETENÇÃO: APAGA OBJETOS NO S3 COM MAIS DE 30 DIAS
# ============================================================================
apply_retention() {
    log "INFO" "Aplicando retenção de ${RETENTION_DAYS} dias em s3://${S3_BUCKET}"

    local cutoff_epoch
    cutoff_epoch="$(date -u -d "${RETENTION_DAYS} days ago" +%s)"

    local removed=0
    local failed=0

    # Lista objetos do bucket e remove os com LastModified anterior ao cutoff.
    # process substitution evita subshell, preservando contadores.
    while IFS=$'\t' read -r last_modified key; do
        [[ -z "${key}" ]] && continue
        local obj_epoch
        obj_epoch="$(date -u -d "${last_modified}" +%s)"
        if (( obj_epoch < cutoff_epoch )); then
            if aws s3 rm "s3://${S3_BUCKET}/${key}" \
                    --region "${AWS_REGION}" >/dev/null 2>>"${LOG_FILE}"
            then
                log "INFO" "Removido por retenção: ${key} (LastModified=${last_modified})"
                removed=$(( removed + 1 ))
            else
                log "WARN" "Falha ao remover ${key} (segue execução)."
                failed=$(( failed + 1 ))
            fi
        fi
    done < <(aws s3api list-objects-v2 \
                --bucket "${S3_BUCKET}" \
                --region "${AWS_REGION}" \
                --query 'Contents[].[LastModified,Key]' \
                --output text 2>>"${LOG_FILE}" || true)

    log "INFO" "Retenção concluída. Removidos=${removed} Falhas=${failed}"
}

# ============================================================================
# 4) HOUSEKEEPING LOCAL: mantém apenas o dump mais recente em disco
# ============================================================================
cleanup_local() {
    find "${LOCAL_DIR}" -maxdepth 1 -type f -name 'ledger_prod_*.sql.gz' -mtime +1 -delete \
        2>>"${LOG_FILE}" || true
}

# ============================================================================
# FLUXO PRINCIPAL
# ============================================================================
main() {
    log "INFO" "=========================================="
    log "INFO" "Backup ledger_prod iniciado (job=${TIMESTAMP})"
    log "INFO" "=========================================="

    check_prereqs
    do_dump
    do_upload
    apply_retention
    cleanup_local

    log "INFO" "Backup finalizado com SUCESSO (job=${TIMESTAMP})."
    exit 0
}

# Trap global: captura erros não previstos e garante exit code != 0 com log.
trap 'die 99 "Erro inesperado em ${BASH_SOURCE[0]}:${LINENO} (status=$?)"' ERR

main "$@"

Instalação:

bash
sudo install -m 0750 -o root -g root ledger-backup.sh /usr/local/bin/ledger-backup.sh
sudo touch /var/log/ledger-backup.log && sudo chmod 0640 /var/log/ledger-backup.log
# Armazene a senha em um EnvironmentFile com permissão 0600 (não no crontab)
echo 'PGPASSWORD=********' | sudo tee /etc/ledger/backup.env >/dev/null
sudo chmod 600 /etc/ledger/backup.env

Agendamento diário no crontab (root) — execução às 02:30 UTC:

bash
cron30 2 * * * root set -a; . /etc/ledger/backup.env; set +a; /usr/local/bin/ledger-backup.sh >> /var/log/ledger-backup.log 2>&1


```
---
## Justificativa

O framework R-T-F (Role – Task – Format) funciona bem nesse exemplo porque cada componente elimina uma classe de ambiguidade: o Role condiciona o nível técnico e as premissas implícitas, fazendo o modelo assumir práticas em vez de chaves estáticas sem precisar explicá-las; a Task, quebrada em blocos funcionais com parâmetros explícitos, transforma cada decisão técnica evitando alucinações e impede que o modelo invente mecanismos inseguros ou abordagens divergentes; e o Format define o artefato final e fecha o ciclo operacional ao exigir também o comando de agendamento — entregando algo colável direto em produção. Em resumo: R-T-F é ideal aqui porque a tarefa tem saída determinística e requisitos fechados, cenário em que explicitar o contrato antes da geração rende mais qualidade do que qualquer iteração posterior.
