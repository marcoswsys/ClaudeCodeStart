# DevOps Infra — Diário Técnico (Mai/2026)

Registro pessoal detalhado das primeiras ações de infraestrutura realizadas com o [Claude Code](https://claude.ai/code) (Anthropic). Contém referências técnicas, comandos completos e caminhos de arquivo para uso futuro.

> ⚠️ Não commitar com credenciais reais. Substituir todos os `<PLACEHOLDERS>` antes de versionar.

---

## Infraestrutura de Rede

```
Hyper-V Host
├── VM Development   <IP_DEV>    Ubuntu 24.04   ~35 containers
├── VM Staging       <IP_STAG>   Ubuntu 24.04   ~17 containers
├── VM QA            <IP_QA>     Ubuntu 24.04   ~14 containers
└── VM Production    <IP_PROD>   Ubuntu 24.04   ~11 containers

Windows Servers
├── <WIN_SERVER_1>   <IP_WIN_1>
└── <WIN_SERVER_2>   <IP_WIN_2>

Outros
├── <SERVER_3>       <IP_SRV_3>
├── AD / DNS         <IP_DNS>
└── NAS              <IP_NAS>
```

Acesso SSH:
```bash
ssh <SSH_USER>@<IP_DEV>    # dev
ssh <SSH_USER>@<IP_STAG>   # staging
ssh <SSH_USER>@<IP_PROD>   # prod
ssh <SSH_USER>@<IP_QA>     # qa
```

---

## 1. Docker Contexts

```bash
# Criar
docker context create infra_development --docker "host=ssh://<SSH_USER>@<IP_DEV>"  --description "VM Desenvolvimento"
docker context create infra_staging     --docker "host=ssh://<SSH_USER>@<IP_STAG>" --description "VM Staging"
docker context create infra_production  --docker "host=ssh://<SSH_USER>@<IP_PROD>" --description "VM Producao"
docker context create infra_qa          --docker "host=ssh://<SSH_USER>@<IP_QA>"   --description "VM QA"

# Ativar
docker context use infra_development

# Remover antigos
docker context rm remoto-dev remoto-homolog
```

---

## 2. SonarQube — Renomear container postgres

**Situação:** container chamado `postgres` era ambíguo. Renomeado para `postgres-sonarqube`.

```bash
# Parar SonarQube
docker stop sonarqube
docker rm sonarqube

# Renomear postgres (sem perda de dados — mesmo volume)
docker rename postgres postgres-sonarqube

# Recriar SonarQube apontando pro novo nome
docker run -d \
  --name sonarqube \
  --restart unless-stopped \
  --network sonarnet \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -e SONAR_JDBC_USERNAME=sonarqube \
  -e SONAR_JDBC_PASSWORD=<SONAR_DB_PASS> \
  -e SONAR_JDBC_URL='jdbc:postgresql://postgres-sonarqube:5432/sonarqube' \
  sonarqube:latest

# Verificar (aguardar ~2 min)
docker logs -f sonarqube
```

**Compose final:** `/home/<SSH_USER>/sonarqube/docker-compose.yml`

---

## 3. Grafana — Migração SQLite → PostgreSQL

**Problema:** `database is locked` no SQLite → falha em alertas e erros 500.

### 3.1 Backup

```bash
mkdir -p /home/<SSH_USER>/migration_backup_20260517

# Backup SQLite do Grafana
docker exec grafana cp /var/lib/grafana/grafana.db /var/lib/grafana/grafana.db.bak_20260517
docker cp grafana:/var/lib/grafana/grafana.db /home/<SSH_USER>/migration_backup_20260517/grafana.db

# Backup PostgreSQL do SonarQube (207 MB)
docker exec postgres-sonarqube pg_dump -U sonarqube sonarqube \
  > /home/<SSH_USER>/migration_backup_20260517/sonarqube_backup.sql
```

### 3.2 Criar postgres-grafana

```bash
docker network create grafana-net

docker run -d \
  --name postgres-grafana \
  --restart unless-stopped \
  --network grafana-net \
  -v postgres_grafana_data:/var/lib/postgresql/data \
  -e POSTGRES_USER=grafana \
  -e POSTGRES_PASSWORD=<GRAFANA_DB_PASS> \
  -e POSTGRES_DB=grafana \
  postgres:16
```

> **Atenção:** Não usar `@` na senha — o pgloader faz URL encoding e quebra a conexão.

### 3.3 Migrar com pgloader

```bash
docker stop grafana

docker run --rm \
  --network grafana-net \
  -v grafana-storage:/data \
  dimitri/pgloader \
  pgloader sqlite:///data/grafana.db \
  'postgresql://grafana:<GRAFANA_DB_PASS>@postgres-grafana/grafana'
```

Resultado esperado: ~7782 registros, ~4.5 MB. Erros não-críticos em `cache_data` e `dashboard_public` são normais (índice duplicado do SQLite).

### 3.4 Corrigir colunas boolean (obrigatório)

O pgloader migra colunas BOOLEAN do SQLite como `bigint`. O Grafana quebra com `operator does not exist: bigint = boolean`. Executar no postgres-grafana:

```sql
-- Padrão para cada coluna:
ALTER TABLE <tabela> ALTER COLUMN <coluna> DROP DEFAULT;
ALTER TABLE <tabela> ALTER COLUMN <coluna> TYPE boolean USING <coluna> != 0;
ALTER TABLE <tabela> ALTER COLUMN <coluna> SET DEFAULT false;
```

**Lista completa das 34 colunas corrigidas:**

| Tabela | Coluna |
|---|---|
| alert_notification | disable_resolve_message, is_default, send_reminder |
| alert_rule | is_paused |
| alert_rule_version | is_paused |
| api_key | is_revoked |
| correlation | provisioned |
| dashboard | has_acl, is_folder, is_public |
| dashboard_public | annotations_enabled, is_enabled, time_selection_enabled |
| dashboard_snapshot | external |
| data_keys | active |
| data_source | basic_auth, is_default, is_prunable, read_only, with_credentials |
| migration_log | success |
| plugin_setting | enabled, pinned |
| resource_migration_log | success |
| role | hidden |
| sso_setting | is_deleted |
| team | is_provisioned |
| team_member | external |
| temp_user | email_sent |
| user | email_verified, is_admin, is_disabled, is_provisioned |
| user_auth_token | auth_token_seen |

### 3.5 Recriar Grafana com PostgreSQL

```bash
docker rm grafana

docker run -d \
  --name grafana \
  --restart unless-stopped \
  --network grafana-net \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  -e TZ=America/Sao_Paulo \
  -e GF_DATABASE_TYPE=postgres \
  -e GF_DATABASE_HOST=postgres-grafana:5432 \
  -e GF_DATABASE_NAME=grafana \
  -e GF_DATABASE_USER=grafana \
  -e GF_DATABASE_PASSWORD=<GRAFANA_DB_PASS> \
  -e GF_DATABASE_SSL_MODE=disable \
  -e GF_SMTP_ENABLED=true \
  -e GF_SMTP_HOST=<SMTP_HOST> \
  -e GF_SMTP_USER=<SMTP_USER> \
  -e GF_SMTP_PASSWORD=<SMTP_PASS> \
  -e GF_SMTP_FROM_ADDRESS=<SMTP_FROM> \
  -e GF_SMTP_FROM_NAME=Grafana \
  grafana/grafana:latest
```

### 3.6 Verificar migração

```sql
SELECT 'dashboards'   AS item, COUNT(*) FROM dashboard;
SELECT 'datasources'  AS item, COUNT(*) FROM data_source;
SELECT 'alert_rules'  AS item, COUNT(*) FROM alert_rule;
SELECT 'annotations'  AS item, COUNT(*) FROM annotation;
SELECT 'org_users'    AS item, COUNT(*) FROM org_user;
```

Esperado: 18 / 3 / 4 / 6243 / 4

---

## 4. Prometheus — HTTP Probes e DNS

### prometheus.yml (job http_probe_sites)

```yaml
- job_name: 'http_probe_sites'
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://<AUTH_PROD_URL>
      - https://<AUTH_HOMOLOG_URL>
      - https://<GATEWAY_HOMOLOG_URL>
      - https://<GED_HOMOLOG_URL>
      - http://<CTRM_HOMOLOG_URL>
      - https://<SITE_PRINCIPAL_URL>
      - https://<SITE_INSTITUCIONAL_URL>
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: <IP_DEV>:9115
```

### Painéis Grafana — HTTP Probe

Queries PromQL úteis nos painéis:

```promql
# Disponibilidade (1 = up, 0 = down)
probe_success{job="http_probe_sites"}

# Latência total da requisição
probe_duration_seconds{job="http_probe_sites"}

# Tempo de resolução DNS — foi aqui que identificamos o problema
probe_dns_lookup_time_seconds{job="http_probe_sites"}

# Tempo de conexão TCP
probe_connect_duration_seconds{job="http_probe_sites"}

# Tempo de handshake TLS
probe_tls_version_info{job="http_probe_sites"}

# Código HTTP retornado
probe_http_status_code{job="http_probe_sites"}
```

### Diagnóstico DNS Interno

Identificado via `probe_dns_lookup_time_seconds` com valores consistentemente elevados nas resoluções de domínios internos. O problema era o servidor DNS interno antigo.

**Correção aplicada:** VMs reconfiguradas para usar o novo servidor DNS interno (`<IP_DNS>`).

Para verificar o DNS configurado em cada VM:
```bash
resolvectl status
# ou
cat /etc/resolv.conf
```

---

## 5. Backup — Containers Dev

**Script:** `/opt/<SSH_USER>/postgres/scripts/backup_dbs_dev.sh`
**Credenciais:** `/opt/<SSH_USER>/postgres/scripts/.dev_db_secrets` (chmod 600)
**Crontab:** `30 01 * * *`
**Destino:** `<NAS_REMOTE>:<NAS_PATH>/Dev/`
**Retenção:** 15 dias

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%Hh%Mm)
REMOTE_PATH="<NAS_REMOTE>:<NAS_PATH>/Dev"
LOCAL_TEMP="/tmp/backup_dbs_dev"
SECRETS="/opt/<SSH_USER>/postgres/scripts/.dev_db_secrets"
source $SECRETS
mkdir -p $LOCAL_TEMP

# Grafana
docker exec -e PGPASSWORD=$GRAFANA_DB_PASS postgres-grafana \
  pg_dump -U grafana grafana > $LOCAL_TEMP/grafana_dev_$DATE.sql

# SonarQube
docker exec -e PGPASSWORD=$SONAR_DB_PASS postgres-sonarqube \
  pg_dump -U sonarqube sonarqube > $LOCAL_TEMP/sonarqube_dev_$DATE.sql

# NPM (MySQL)
docker exec npm_db_dev mysqldump -u root -p"$NPM_ROOT_PASS" npm \
  > $LOCAL_TEMP/npm_dev_$DATE.sql

gzip $LOCAL_TEMP/*.sql
rclone copy $LOCAL_TEMP "$REMOTE_PATH"
rm -rf $LOCAL_TEMP
rclone delete "$REMOTE_PATH" --min-age 15d
```

**Teste de restore:**
```bash
# Baixar do NAS
rclone copy '<NAS_REMOTE>:<NAS_PATH>/Dev/grafana_dev_<DATA>.sql.gz' /tmp/restore_test/

# Container temporário
docker run -d --name grafana-restore-test \
  -e POSTGRES_USER=grafana \
  -e POSTGRES_PASSWORD=<GRAFANA_DB_PASS> \
  -e POSTGRES_DB=grafana \
  postgres:16

# Restaurar
gunzip -c /tmp/restore_test/grafana_dev_<DATA>.sql.gz | \
  docker exec -i grafana-restore-test psql -U grafana -d grafana -q

# Limpar
docker stop grafana-restore-test && docker rm grafana-restore-test
rm -rf /tmp/restore_test
```

---

## 6. Limpeza Docker

```bash
# Remover volumes órfãos
docker volume prune -f

# Remover imagens sem uso
docker image prune -f

# Remover build cache
docker builder prune -f

# Ver uso de disco Docker
docker system df
```

**Resultado em 17/05/2026:**

| VM | Espaço Liberado | Origem |
|---|---|---|
| Dev | ~19,6 GB | Volumes Greenbone/OpenVAS (instalação antiga) |
| QA | ~6,6 GB | Volumes anônimos + images dangling |
| Staging | ~2,5 GB | Build cache |
| Prod | — | Já estava limpa |
| **Total** | **~28,7 GB** | |

---

## Caminhos Importantes (Dev VM)

| Caminho | Descrição |
|---|---|
| `/home/<SSH_USER>/grafana/docker-compose.yml` | Stack Grafana + postgres-grafana |
| `/home/<SSH_USER>/sonarqube/docker-compose.yml` | Stack SonarQube + postgres-sonarqube |
| `/home/<SSH_USER>/migration_backup_20260517/` | Backups pré-migração |
| `/opt/<SSH_USER>/postgres/scripts/backup_dbs_dev.sh` | Script backup containers dev |
| `/opt/<SSH_USER>/postgres/scripts/.dev_db_secrets` | Credenciais backup (chmod 600) |
| `/home/<SSH_USER>/prometheus/prometheus.yml` | Config Prometheus (volume-mounted) |
