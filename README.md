# ESO — External Secrets Operator (ArgoCD Application)

ArgoCD-приложение, которое устанавливает **External Secrets Operator** и настраивает
интеграцию с **HashiCorp Vault** (AppRole auth, KV v2) для управления секретами
в Kubernetes-кластере.

## Структура

```
eso/
├── Chart.yaml                              # Umbrella Helm chart (ESO как dependency)
├── values.yaml                             # Параметры: AWS, Vault, SecretStore, ExternalSecret
├── README.md
└── templates/
    ├── _helpers.tpl                        # Helm-хелперы
    ├── secretstore-aws.yaml                # SecretStore → AWS SSM Parameter Store  (wave 1)
    ├── external-secret-vault-approle.yaml  # ExternalSecret: SSM → vault-approle-credentials (wave 2)
    ├── secretstore.yaml                    # ClusterSecretStore → Vault             (wave 3)
    ├── external-secret-grafana.yaml        # ExternalSecret → grafana (monitoring)  (wave 4)
    └── external-secret-ingress.yaml        # ExternalSecret → ingress-settings (etc-infra) (wave 4)
```

### Sync waves

| Wave | Ресурс | Описание |
|------|--------|----------|
| 0 | ESO Operator | CRDs + контроллер (Helm subchart) |
| 1 | SecretStore `aws-ssm-backend` | Подключение к AWS SSM (через IAM роль EC2) |
| 2 | ExternalSecret `vault-approle-credentials` | AppRole creds из SSM → K8s Secret |
| 3 | ClusterSecretStore `vault-backend` | Подключение к Vault (использует Secret из wave 2) |
| 4 | ExternalSecret `grafana` | Секреты Grafana из Vault → K8s Secret (ns: monitoring) |
| 4 | ExternalSecret `ingress-settings` | Секреты Ingress из Vault → K8s Secret (ns: etc-infra) |

## Предварительные действия (до первого sync)

### 1. AppRole credentials в AWS SSM (автоматически)

AppRole `role-id` и `secret-id` хранятся в **AWS SSM Parameter Store** и автоматически
загружаются в Kubernetes Secret через ESO. Ручное создание Secret **не требуется**.

**Пути параметров в SSM:**
- `/monitoring_cluster/app_role_id` → Vault AppRole Role ID
- `/monitoring_cluster/secret_id` → Vault AppRole Secret ID

**Требования:**
- EC2 инстанс должен иметь IAM роль с политикой `ssm:GetParameter` для указанных параметров
- Параметры должны быть созданы в SSM (регион `us-east-2`)

**Цепочка:** SSM Parameter Store → ESO (ExternalSecret wave 2) → K8s Secret `vault-approle-credentials` → Vault SecretStore (wave 3)

### 2. CA-сертификат для Vault (self-signed TLS)

Vault использует самоподписанный сертификат. Чтобы ESO мог установить TLS-соединение,
нужно передать корневой CA-сертификат.

#### Получение сертификата

Если CA-сертификат ещё не получен, его можно извлечь с сервера Vault:

```bash
# Вариант 1: через openssl
openssl s_client -connect 172.31.16.128:8200 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -outform PEM > vault-ca.crt

# Вариант 2: если есть доступ к серверу Vault — скопировать CA оттуда
# По умолчанию для integrated storage:
#   /opt/vault/tls/tls.ca (или CA, которым был подписан сертификат)
```

#### Создание Secret с CA-сертификатом

```bash
kubectl create secret generic vault-ca-cert \
  --namespace default \
  --from-file=ca.crt=vault-ca.crt
```

> **Имя Secret** (`vault-ca-cert`) и **ключ** (`ca.crt`) должны совпадать
> со значениями `vault.caSecretName` и `vault.caSecretKey` в `values.yaml`.

#### Альтернатива: отключить проверку TLS (не рекомендуется)

Если по какой-то причине невозможно использовать CA-сертификат, можно
настроить SecretStore без `caProvider` — но для этого потребуется
изменить `templates/secretstore.yaml`, заменив блок `caProvider` на:

```yaml
# НЕ РЕКОМЕНДУЕТСЯ для production
# tls:
#   insecureSkipVerify: true  # доступно только в некоторых версиях ESO
```

> ⚠️ **Это снижает безопасность** — без проверки сертификата возможна
> MITM-атака. Используйте CA-сертификат.

## Деплой через ArgoCD

### Способ 1: ArgoCD Application manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eso
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:muzzai/eso.git
    path: "."
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Способ 2: argocd CLI

```bash
argocd app create eso \
  --repo <GIT_REPO_URL> \
  --path eso \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Проверка

```bash
# ESO operator запущен
kubectl get pods -n default -l app.kubernetes.io/name=external-secrets

# Оба SecretStore подключены
kubectl get secretstore -n default
# NAME                STATUS
# aws-ssm-backend     Valid
# vault-backend       Valid

# Все ExternalSecret синхронизированы
kubectl get externalsecret -A
# NAMESPACE     NAME                           STATUS
# default       vault-approle-credentials      SecretSynced
# monitoring    grafana                        SecretSynced
# etc-infra     ingress-settings               SecretSynced

# Kubernetes Secrets созданы
kubectl get secret vault-approle-credentials -n default
kubectl get secret grafana -n monitoring
kubectl get secret ingress-settings -n etc-infra
```

## Добавление новых секретов

Чтобы добавить ExternalSecret для другого приложения:

1. Создайте новый файл в `templates/`, например `external-secret-myapp.yaml`
2. Добавьте параметры в `values.yaml`
3. Используйте sync-wave `"2"` или выше
4. Commit + push — ArgoCD синхронизирует автоматически

## Настройка Vault

Для работы приложения в Vault должны быть настроены:

1. **KV v2 engine** по пути `secret/`
2. **AppRole auth method** включён
3. **Role** с политикой, разрешающей чтение `secret/data/grafana` и `secret/data/etc-infra`
4. **Секреты:**
   - `secret/grafana` — ключи: `admin-user`, `admin-password`, `teamsWebHook`
   - `secret/etc-infra` — ключи: `domain`, `grafana_hostname`, `logs_hostname`, `metrics_hostname`

```bash
# Включить KV v2 (если ещё не включён)
vault secrets enable -path=secret kv-v2

# Включить AppRole
vault auth enable approle

# Создать политику
vault policy write eso-policy - <<EOF
path "secret/data/grafana" {
  capabilities = ["read"]
}

path "secret/data/etc-infra" {
  capabilities = ["read"]
}
EOF

# Создать роль
vault write auth/approle/role/eso-role \
  token_policies="eso-policy" \
  token_ttl=1h \
  token_max_ttl=4h

# Получить role-id и secret-id
vault read auth/approle/role/eso-role/role-id
vault write -f auth/approle/role/eso-role/secret-id

# Создать секреты
vault kv put secret/grafana \
  admin-user=admin \
  admin-password=<PASSWORD> \
  teamsWebHook=<TEAMS_WEBHOOK_URL>

vault kv put secret/etc-infra \
  domain=<DOMAIN> \
  grafana_hostname=<GRAFANA_HOSTNAME> \
  logs_hostname=<LOGS_HOSTNAME> \
  metrics_hostname=<METRICS_HOSTNAME>
```
