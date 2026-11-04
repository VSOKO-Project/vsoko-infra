# GitLab Agent для группы VSOKO

Один агент `vsoko`, зарегистрированный в `vsoko/vsoko-infra`, обслуживает группу
`vsoko` и её подгруппы. Все разрешённые jobs используют одни Kubernetes-права.
Контекст CI: `vsoko/vsoko-infra:vsoko`.

Агент работает в `gitlab-agent-vsoko`, имеет Role только в `vsoko`.
Он может управлять workloads, конфигурацией, Secrets, PVC, SecretStores и ExternalSecrets,
но не namespace, ClusterSecretStore, RBAC или workloads своего namespace.
Certificates в `vsoko` доступны для чтения и ожидания готовности TLS;
ClusterIssuer управляет администратор.
В служебном namespace разрешены только Lease для координации работы агента
и запись служебных Events.
Группа считается одной зоной доверия: её pipeline имеют доступ к секретам VSOKO.
RBAC не заменяет Pod Security и не изолирует недоверенные контейнеры на общих ВМ.

Подключение использует `wss://kas.gitlab.com`: стандартный gRPC endpoint
возвращал Cloudflare 403 из этой сети.

Конфигурация доступа лежит в `.gitlab/agents/vsoko/config.yaml` в `main`.
Регистрация и агентский токен создаются через GitLab Kubernetes Agent API.
Токен записывается в Secret `gitlab-agent-vsoko/gitlab-agent-token`, поле `token`,
без сохранения в Git или Helm values. Это не GitLab PAT для ESO.

Первичная установка выполняется администратором. Перед Helm должны существовать
namespace `vsoko`, namespace агента и Secret с его токеном.
Из корня репозитория:

```bash
kubectl apply -f bootstrap/gitlab-agent/rbac.yaml
helm upgrade --install vsoko gitlab-agent \
  --repo https://charts.gitlab.io --version 2.33.0 \
  --namespace gitlab-agent-vsoko --create-namespace \
  -f bootstrap/gitlab-agent/values.yaml --wait --timeout 5m
kubectl -n gitlab-agent-vsoko get pods
```

Namespace приложения и Secret `gitlab-token` создаёт администратор перед первым
деплоем. Pipeline группы применяет SecretStores, ExternalSecrets и ресурсы
приложений внутри `vsoko` — ему не нужны cluster-admin права.
GitLab Runner остаётся отдельным компонентом, исполняющим jobs.
