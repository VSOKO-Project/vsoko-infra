# VSOKO Infra

Инфраструктура VSOKO в k3s, namespace `vsoko`, одно окружение production.
Traefik и External Secrets Operator (ESO) уже установлены в кластере.
GitLab Agent `vsoko` настроен для всей группы. Деплой выполняется вручную
через pipeline; push сам по себе PostgreSQL и Redis не запускает.

```text
.gitlab/agents/vsoko/config.yaml # доступ всей группы VSOKO
bootstrap/gitlab-agent/        # установка агента и его права
k8s/namespace.yaml
k8s/secrets/store.yaml  # SecretStore → Variables infra
k8s/secrets/external-secrets.yaml     # GitLab Variables → Kubernetes Secrets
k8s/postgres/                 # Service, StatefulSet, PVC, инициализация пользователя
k8s/redis/                    # Service, StatefulSet, PVC
```

## Секреты

Источник — Variables проекта `vsoko/vsoko-infra` (ID `79795329`).
Настройки: **Masked**, **Protected**, scope `*`, expansion выключен.
Не выбирай Hidden: API возвращает `null` вместо скрытого значения, ESO его не получит.

| GitLab Variable | Kubernetes Secret / поле | Назначение |
|---|---|---|
| `POSTGRES_ADMIN_PASSWORD` | `postgres-admin / password` | Администратор `postgres` |
| `POSTGRES_APP_PASSWORD` | `postgres-app / password` | Пользователь API `vsoko_app` |
| `REDIS_PASSWORD` | `redis / password` | Авторизация Redis |
| `REGISTRY_USERNAME` / `REGISTRY_TOKEN` | `gitlab-registry / .dockerconfigjson` | Group Deploy Token с `read_registry` для скачивания образов API и фронта |

Токен GitLab уже создан вручную в Secret `vsoko/gitlab-token`, поле `token`.
Это fine-grained PAT: доступ к группе `vsoko`, разрешение `Variable → Read`.
Он не хранится в Git; его нужно заменить перед истечением срока действия.
`SecretStore gitlab-vsoko` ссылается на этот Secret и разрешает использование
только из namespace `vsoko`. Он читает переменные конкретного проекта, а не всей группы.
Отдельный `SecretStore gitlab-vsoko-api` описан в репозитории API (`k8s/secrets/store.yaml`),
использует тот же токен, но читает
Variables проекта `vsoko-api` (JWT и OpenRouter). Это разделение источников,
а не дополнительная граница прав доступа.

ESO сверяет значения раз в минуту. `Orphan` сохраняет созданный Secret при удалении
ExternalSecret; `Retain` сохраняет данные при исчезновении переменной из GitLab.
Это не бэкап и не защита от удаления самого Secret или namespace.

## CI/CD

Pipeline работает на GitLab-hosted runner `saas-linux-small-amd64`.
На Free используется [месячная квота compute minutes](https://docs.gitlab.com/ci/pipelines/compute_minutes/).
SSH-ключи и личный kubeconfig в CI не нужны: подключение предоставляет GitLab Agent.

- `validate` — при merge request и на `main`: проверка ресурсов приложения через
  Kubernetes API (`--dry-run=server`), без сохранения изменений в кластере.
- `deploy` — только кнопкой из защищённой `main`, после успешной проверки:
  SecretStore + ExternalSecrets → ожидание синхронизации → PostgreSQL/Redis → ожидание готовности.
  `resource_group` не позволяет двум deploy jobs выполняться одновременно.

Первый запуск: **Build → Pipelines → pipeline последнего коммита → deploy → Play**.
Дождись успешного `validate`. Деплой требует подтверждения и автоматически не стартует.
Pipeline со статусом blocked/manual после проверки ожидает твоего нажатия.

## Первоначальная настройка администратором

Уже подготовлены ESO, GitLab Agent, namespace `vsoko`, токен `vsoko/gitlab-token`
и `SecretStore gitlab-vsoko`. При восстановлении кластера сначала создаются
namespace и токен (из менеджера паролей), затем подключение к GitLab:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secrets/store.yaml
kubectl wait --for=condition=Ready secretstore/gitlab-vsoko --timeout=120s
```

Namespace и RBAC не применяются pipeline группы — ими управляет
администратор. После изменения этих файлов нужно отдельно проверить и применить их
администраторским доступом. SecretStore и ExternalSecrets применяются и проверяются pipeline соответствующего проекта.

## PostgreSQL и Redis

| Сервис | Адрес внутри namespace | Хранилище |
|---|---|---|
| PostgreSQL 16 | `postgres:5432` | PVC 20 GiB, `local-path` |
| Redis 7.4 | `redis:6379` | PVC 2 GiB, `local-path`, AOF |

Порты наружу не публикуются. Headless Services дают DNS-имена готовых Pod;
это не сетевая изоляция от других namespace.

На **пустом томе** PostgreSQL создаёт администратора `postgres` и базу `vsoko`,
затем `postgres-init` создаёт `vsoko_app` и передаёт ему владение этой базой.
Пользователь приложения не superuser, не может создавать роли или другие базы.
Он может изменять схему своей базы: миграции запускает отдельный Job с EF bundle
перед обновлением API. В production API не меняет схему при старте.
Отдельные credentials для миграций можно выделить позже.

API должен получать только `postgres-app` и `redis`, без `postgres-admin`:

```text
ConnectionStrings__Database=Host=postgres;Port=5432;Database=vsoko;Username=vsoko_app;Password=<postgres-app/password>
ConnectionStrings__Redis=redis:6379,password=<redis/password>
```

Init-скрипт не запускается повторно на существующих данных. Изменение ConfigMap,
перезапуск Pod и обновление Secret не изменяют пользователей или пароли в самой БД.
Для смены пароля нужно согласованно изменить пароль роли в PostgreSQL, значение
в GitLab и перезапустить использующие env приложения. Redis также читает пароль
из env при запуске: его и клиентов требуется согласованно перезапустить.

Восстановление старого дампа выполняется отдельно с учётом владения объектами:
объекты приложения должны принадлежать `vsoko_app`, чтобы миграции могли их менять.

Redis хранит кеш и ключи ASP.NET Data Protection. Поэтому включены PVC и AOF
с `appendfsync everysec` (при аварии возможна потеря примерно секунды записей).
`noeviction` защищает ключи от вытеснения, но при лимите памяти записи могут
завершаться ошибкой. AOF не заменяет бэкап.

## Хранение и восстановление

По одному экземпляру PostgreSQL и Redis, без HA. `local-path` привязывает данные
к диску выбранной ВМ; при её отказе данные не переедут на другую ВМ.
PVC сохраняются при удалении StatefulSet, но удаление PVC или namespace
с текущим StorageClass `Delete` может удалить данные. Размер PVC не является
дисковой квотой для `local-path`; необходимо следить за свободным местом на ВМ.
Перед загрузкой рабочих данных нужны бэкапы обеих БД вне ВМ и проверка восстановления.
Автоматические бэкапы пока не настроены.

## GitLab Agent

Один агент `vsoko` для всей группы VSOKO, зарегистрирован в `vsoko/vsoko-infra`.
Namespace агента: `gitlab-agent-vsoko`. Контекст CI: `vsoko/vsoko-infra:vsoko`.
Все проекты и подгруппы VSOKO используют одинаковые права внутри namespace `vsoko`.
Namespace и права RBAC остаются под управлением администратора.
Установка и параметры: [bootstrap/gitlab-agent](bootstrap/gitlab-agent/README.md).
Агент предоставляет подключение; команды из `.gitlab-ci.yml` выполняет GitLab-hosted Runner.

## System guide

[Product, architecture, integration and operations](https://gitlab.com/vsoko/specs).
