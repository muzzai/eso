# ESO — External Secrets Operator (ArgoCD Application)

ArgoCD-приложение, которое устанавливает **External Secrets Operator** и настраивает
интеграцию с **HashiCorp Vault** (AppRole auth, KV v2) для управления секретами
в Kubernetes-кластере.

## Структура

```
eso/
├── Chart.yaml                         # Umbrella Helm chart (ESO как dependency)
├── values.yaml                        # Параметры: Vault, SecretStore, ExternalSecret
├── README.md
└── templates/
    ├── _helpers.tpl                   # Helm-хелперы
    ├── secretstore.yaml               # SecretStore → Vault
    └── external-secret-grafana.yaml   # ExternalSecret → grafana-credentials
```

## Предварительные действия (до первого sync)

Перед тем как ArgoCD синхронизирует приложение, необходимо вручную создать два
Secret в кластере — они содержат конфиденциальные данные, которые **не должны**
храниться в Git.

### 1. AppRole credentials

```bash
kubectl create secret generic vault-approle-credentials \
  --namespace default \
  --from-literal=role-id=<VAULT_ROLE_ID> \
  --from-literal=secret-id=<VAULT_SECRET_ID>
```

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
    repoURL: <GIT_REPO_URL>
    path: eso
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

# SecretStore подключён к Vault
kubectl get secretstore -n default
# STATUS: Valid

# ExternalSecret синхронизирован
kubectl get externalsecret -n default
# STATUS: SecretSynced

# Kubernetes Secret создан
kubectl get secret grafana-credentials -n default -o yaml
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
3. **Role** с политикой, разрешающей чтение `secret/data/grafana`
4. **Секрет** `secret/grafana` с ключами `admin-user` и `admin-password`

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
EOF

# Создать роль
vault write auth/approle/role/eso-role \
  token_policies="eso-policy" \
  token_ttl=1h \
  token_max_ttl=4h

# Получить role-id и secret-id
vault read auth/approle/role/eso-role/role-id
vault write -f auth/approle/role/eso-role/secret-id

# Создать секрет для Grafana
vault kv put secret/grafana admin-user=admin admin-password=<PASSWORD>
```
