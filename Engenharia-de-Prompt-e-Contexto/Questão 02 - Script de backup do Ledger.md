# Questão 02 — Script de backup automatizado do banco Ledger

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** claude-sonnet-4-6
**Framework:** R-T-F

---

## Contexto

> Automação de backup do banco PostgreSQL `ledger_prod` para S3, em ambiente AWS com autenticação via IAM role.
> Requisitos de produção: compactação, retenção de 30 dias, log estruturado e integração com AWS Secrets Manager.

---

## Prompt usado

```
# Role
Engenheiro SRE / DevOps sênior, Especialista em automação de infraestrutura Linux, PostgreSQL e AWS, com experiência em scripts Bash de produção, estratégias de backup e integração com S3.

# Task
Criar script Bash de backup automatizado
Escrever um script Bash de produção que realize as seguintes etapas, nessa ordem:
1. Exportar o banco ledger_prod via pg_dump conectando em ledger-db.internal.hvt.io:5432 com o usuário backup_user (senha via variável de ambiente PGPASSWORD, injetada pelo AWS Secrets Manager através da IAM role da instância).
2. Compactar o dump com gzip, gerando arquivo nomeado com timestamp (ledger_prod_YYYYMMDD_HHMMSS.sql.gz) em /var/backups/ledger.
3. Enviar o arquivo para o bucket S3 hvt-ledger-backups na região us-east-1 via aws s3 cp, autenticando pela IAM role (sem credenciais hardcoded).
4. Aplicar política de retenção de 30 dias: listar e remover do S3 arquivos com mais de 30 dias via aws s3 ls + filtragem por data.
5. Registrar cada etapa (início, sucesso, erro) em /var/log/ledger-backup.log com timestamp ISO-8601.
6. Sair com exit 0 em sucesso e exit 1 em qualquer falha, interrompendo imediatamente ao primeiro erro (set -euo pipefail).

# Format
Retorne EXCLUSIVAMENTE:

1. Um único arquivo .sh com shebang #!/usr/bin/env bash.
2. Variáveis de configuração agrupadas no topo, comentadas, fáceis de ajustar.
3. Função de log reutilizável (log_info, log_error) usada em todas as etapas.
4. Tratamento de erros explícito em cada comando crítico (pg_dump, gzip, aws s3 cp, limpeza S3).
5. Limpeza do arquivo local temporário ao final (sucesso ou falha) via trap.
6. Comentários inline explicando decisões não óbvias (ex: por que --no-password no pg_dump, como o Secrets Manager injeta o PGPASSWORD).
7. Após o script: bloco de instruções de instalação mostrando como registrar no crontab para rodar diariamente às 02:00 UTC.
```

---

## Output completo da IA

```bash
#!/usr/bin/env bash
# ==============================================================================
# ledger-backup.sh — Backup automatizado do banco PostgreSQL ledger_prod para S3
# Autenticação: IAM role da instância EC2 (sem credenciais hardcoded)
# Secrets Manager: injeta PGPASSWORD via variável de ambiente antes da execução
# ==============================================================================
set -euo pipefail
# set -e: sai imediatamente em qualquer erro não tratado
# set -u: trata variáveis não definidas como erro (evita execução silenciosa com vazios)
# set -o pipefail: propaga erros em pipes — sem isso, "pg_dump | gzip" mascara falha do pg_dump

# ------------------------------------------------------------------------------
# CONFIGURAÇÃO — ajuste aqui sem tocar no restante do script
# ------------------------------------------------------------------------------
DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_NAME="ledger_prod"
DB_USER="backup_user"

BACKUP_DIR="/var/backups/ledger"
LOG_FILE="/var/log/ledger-backup.log"

S3_BUCKET="hvt-ledger-backups"
S3_REGION="us-east-1"
S3_PREFIX="backups"           # prefixo (pasta lógica) dentro do bucket
RETENTION_DAYS=30

TIMESTAMP=$(date -u +"%Y%m%d_%H%M%S")
DUMP_FILE="${BACKUP_DIR}/ledger_prod_${TIMESTAMP}.sql.gz"

# ------------------------------------------------------------------------------
# FUNÇÕES DE LOG — ISO-8601 UTC em todas as entradas
# ------------------------------------------------------------------------------
log_info() {
    echo "$(date -u +"%Y-%m-%dT%H:%M:%SZ") [INFO]  $*" | tee -a "${LOG_FILE}"
}

log_error() {
    echo "$(date -u +"%Y-%m-%dT%H:%M:%SZ") [ERROR] $*" | tee -a "${LOG_FILE}" >&2
}

# ------------------------------------------------------------------------------
# LIMPEZA — executada em qualquer saída (sucesso, falha, SIGTERM, SIGINT)
# trap EXIT garante que o arquivo local temporário não fique em disco
# mesmo se o script for interrompido externamente
# ------------------------------------------------------------------------------
cleanup() {
    if [[ -f "${DUMP_FILE}" ]]; then
        log_info "Removendo arquivo local temporário: ${DUMP_FILE}"
        rm -f "${DUMP_FILE}"
    fi
}
trap cleanup EXIT

# ------------------------------------------------------------------------------
# PRÉ-REQUISITOS
# ------------------------------------------------------------------------------
mkdir -p "${BACKUP_DIR}"

# Valida que PGPASSWORD foi injetado antes de chamar este script.
# O wrapper (systemd unit ou entrypoint) deve executar:
#   export PGPASSWORD=$(aws secretsmanager get-secret-value \
#     --secret-id ledger/backup_user --query SecretString --output text)
# Nunca armazene a senha no crontab, no ambiente do sistema ou no próprio script.
if [[ -z "${PGPASSWORD:-}" ]]; then
    log_error "PGPASSWORD não definido. Certifique-se de que o Secrets Manager injetou a variável antes da execução."
    exit 1
fi

# ------------------------------------------------------------------------------
# ETAPAS 1 + 2: pg_dump em pipeline com gzip
# ------------------------------------------------------------------------------
log_info "Iniciando backup de ${DB_NAME}@${DB_HOST}:${DB_PORT}"

# --no-password: impede prompt interativo — a autenticação vem exclusivamente de PGPASSWORD.
# Pipeline direto para gzip evita escrever o dump em texto plano no disco,
# reduzindo janela de exposição de dados e uso de I/O em disco.
# gzip -9 prioriza compactação máxima (adequado para execução noturna off-peak).
if pg_dump \
    --host="${DB_HOST}" \
    --port="${DB_PORT}" \
    --username="${DB_USER}" \
    --no-password \
    --format=plain \
    --no-acl \
    --no-owner \
    "${DB_NAME}" | gzip -9 > "${DUMP_FILE}"; then
    log_info "Dump + compactação concluídos: ${DUMP_FILE} ($(du -sh "${DUMP_FILE}" | cut -f1))"
else
    log_error "Falha no pg_dump ou gzip para ${DB_NAME}"
    exit 1
fi

# ------------------------------------------------------------------------------
# ETAPA 3: upload para S3 via IAM role (sem credenciais hardcoded)
# ------------------------------------------------------------------------------
S3_KEY="${S3_PREFIX}/ledger_prod_${TIMESTAMP}.sql.gz"
S3_URI="s3://${S3_BUCKET}/${S3_KEY}"

log_info "Enviando ${DUMP_FILE} para ${S3_URI}"

# --sse aws:kms: criptografia server-side obrigatória para dados financeiros em repouso.
# --storage-class STANDARD_IA: custo ~40% menor que STANDARD para backups acessados raramente.
# A autenticação usa automaticamente a IAM role da instância (via metadata endpoint 169.254.169.254).
if aws s3 cp "${DUMP_FILE}" "${S3_URI}" \
    --region "${S3_REGION}" \
    --sse aws:kms \
    --storage-class STANDARD_IA; then
    log_info "Upload concluído: ${S3_URI}"
else
    log_error "Falha no upload para S3: ${S3_URI}"
    exit 1
fi

# ------------------------------------------------------------------------------
# ETAPA 4: retenção de 30 dias — remove objetos antigos do S3
# ------------------------------------------------------------------------------
log_info "Aplicando política de retenção de ${RETENTION_DAYS} dias no bucket ${S3_BUCKET}/${S3_PREFIX}/"

# Data-limite: objetos com LastModified anterior a esta data serão removidos.
# Comparação lexicográfica funciona porque o formato ISO-8601 (YYYY-MM-DD) é ordenável.
# "date -d" requer GNU coreutils (disponível em Amazon Linux, Ubuntu, Debian).
CUTOFF_DATE=$(date -u -d "${RETENTION_DAYS} days ago" +"%Y-%m-%d")
log_info "Removendo objetos com data anterior a ${CUTOFF_DATE}"

DELETED=0
ERRORS=0

# aws s3 ls retorna linhas no formato: "YYYY-MM-DD HH:MM:SS    <bytes>  <key>"
# Lemos via process substitution (< <(...)) para não executar o loop em subshell,
# preservando as variáveis DELETED e ERRORS no escopo atual.
while IFS= read -r line; do
    FILE_DATE=$(echo "${line}" | awk '{print $1}')
    FILE_KEY=$(echo "${line}" | awk '{print $4}')

    [[ -z "${FILE_KEY}" ]] && continue

    if [[ "${FILE_DATE}" < "${CUTOFF_DATE}" ]]; then
        log_info "Removendo objeto antigo: s3://${S3_BUCKET}/${FILE_KEY}"
        if aws s3 rm "s3://${S3_BUCKET}/${FILE_KEY}" --region "${S3_REGION}"; then
            DELETED=$((DELETED + 1))
        else
            log_error "Falha ao remover s3://${S3_BUCKET}/${FILE_KEY}"
            ERRORS=$((ERRORS + 1))
        fi
    fi
done < <(aws s3 ls "s3://${S3_BUCKET}/${S3_PREFIX}/" --region "${S3_REGION}")

log_info "Limpeza concluída: ${DELETED} objeto(s) removido(s), ${ERRORS} erro(s)"

if [[ "${ERRORS}" -gt 0 ]]; then
    log_error "A limpeza S3 terminou com ${ERRORS} falha(s). Verifique o log."
    exit 1
fi

# ------------------------------------------------------------------------------
# SUCESSO
# ------------------------------------------------------------------------------
log_info "Backup do ${DB_NAME} concluído com sucesso. Arquivo remoto: ${S3_URI}"
exit 0
```

---

**Instruções de instalação — crontab diário às 02:00 UTC**

```bash
# 1. Copiar o script e torná-lo executável
sudo cp ledger-backup.sh /usr/local/sbin/ledger-backup.sh
sudo chmod 750 /usr/local/sbin/ledger-backup.sh
sudo chown root:root /usr/local/sbin/ledger-backup.sh

# 2. Criar arquivo wrapper que injeta PGPASSWORD via Secrets Manager antes de chamar o script.
#    O wrapper roda com as permissões da IAM role da instância (sem credenciais em disco).
sudo tee /usr/local/sbin/ledger-backup-wrapper.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
export PGPASSWORD=$(aws secretsmanager get-secret-value \
    --secret-id ledger/backup_user \
    --region us-east-1 \
    --query SecretString \
    --output text)
exec /usr/local/sbin/ledger-backup.sh
EOF
sudo chmod 750 /usr/local/sbin/ledger-backup-wrapper.sh
sudo chown root:root /usr/local/sbin/ledger-backup-wrapper.sh

# 3. Criar arquivo de log com permissões adequadas
sudo touch /var/log/ledger-backup.log
sudo chmod 640 /var/log/ledger-backup.log

# 4. Registrar no crontab do root para execução diária às 02:00 UTC
#    PATH mínimo garante que aws, pg_dump e gzip sejam encontrados
(sudo crontab -l 2>/dev/null; echo "0 2 * * * PATH=/usr/local/bin:/usr/bin:/bin /usr/local/sbin/ledger-backup-wrapper.sh >> /var/log/ledger-backup.log 2>&1") | sudo crontab -

# 5. Verificar entrada gravada
sudo crontab -l | grep ledger-backup
```

---

## Análise do framework

**Justificativa:** O prompt segue o framework **R-T-F** (Role / Task / Format). Os três elementos estão delimitados por cabeçalhos de seção explícitos (`# Role`, `# Task`, `# Format`), tornando a estrutura imediatamente legível e auditável. O **Role** define o perfil técnico que condiciona o nível de sofisticação do output (decisões como `set -euo pipefail`, pipeline `pg_dump | gzip`, e `trap EXIT` são respostas diretas ao perfil SRE sênior declarado). O **Task** detalha as etapas em ordem sequencial obrigatória, impedindo que a IA reordene ou omita etapas. O **Format** especifica a estrutura de entrega com granularidade suficiente para eliminar variações indesejadas (ex.: exigir comentários inline, `trap`, variáveis agrupadas no topo).

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| **R** — Role | `"Engenheiro SRE / DevOps sênior, Especialista em automação de infraestrutura Linux, PostgreSQL e AWS, com experiência em scripts Bash de produção, estratégias de backup e integração com S3."` |
| **T** — Task | `"Escrever um script Bash de produção que realize as seguintes etapas, nessa ordem: 1. Exportar o banco ledger_prod via pg_dump... 6. Sair com exit 0 em sucesso e exit 1 em qualquer falha..."` |
| **F** — Format | `"Retorne EXCLUSIVAMENTE: 1. Um único arquivo .sh com shebang... 7. Após o script: bloco de instruções de instalação mostrando como registrar no crontab para rodar diariamente às 02:00 UTC."` |
