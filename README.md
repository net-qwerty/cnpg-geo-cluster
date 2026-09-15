# Геораспределённый PostgreSQL с CloudNativePG

Пример конфигурации к статье «Геораспределённый PostgreSQL в Kubernetes:
CloudNativePG, Longhorn и S3». Манифесты предназначены для подключения
по компонентам в существующую инфраструктуру Flux CD.

| Кластер | PostgreSQL | Начальная роль |
| --- | --- | --- |
| `k8s-spb` | `app-spb-pg` | PRIMARY, создание базы `app` через `initdb` |
| `k8s-msk` | `app-msk-pg` | REPLICA, восстановление из backup SPB |

В каждом регионе работают три экземпляра PostgreSQL и три PgBouncer.
Между регионами репликация выполняется через архив WAL в S3.
Longhorn предоставляет тома с одной копией и `strict-local` размещением.

## Структура

В `clusters/k8s-spb/` и `clusters/k8s-msk/` находятся:

- `cert-manager/` — cert-manager и его HelmRepository.
- `cnpg-system/` — оператор CloudNativePG и Barman Cloud Plugin.
- `longhorn-system/` — Longhorn и StorageClass `longhorn-local`.
- `postgres/` — Cluster, Secrets, ObjectStore, backup, PgBouncer, LoadBalancer и мониторинг.

## Подключение через Flux

1. Подготовьте в каждом регионе три PostgreSQL-ноды с разными значениями
   `topology.kubernetes.io/zone`, меткой `node-role.kubernetes.io/postgres=postgres`
   и необходимыми для Longhorn дисками и зависимостями ОС.
2. Установите Flux и Prometheus Operator с CRD `PodMonitor`/`ServiceMonitor`.
3. Создайте S3 buckets `app-spb-pg` и `app-msk-pg`, доступные из обоих кластеров.
   Проверьте endpoints и замените демонстрационные credentials в Secrets.
   Пароли соответствующих PostgreSQL-пользователей должны совпадать между регионами;
   реальные секреты храните в Git в зашифрованном виде с настроенным Flux decryption.
4. Скопируйте папки компонентов в свой Git-репозиторий и подключите их через
   родительский `kustomization.yaml`. Скорректируйте `spec.path` и `sourceRef`
   ресурсов Flux Kustomization под свою инфраструктуру.
5. Сначала разверните SPB и дождитесь первого успешного base backup.
   Затем разверните MSK. Межрегиональное ожидание backup не обеспечивается `dependsOn`.

Backup запускается ежедневно: SPB — в 06:00, MSK — в 06:30 по timezone
scheduler оператора. LoadBalancer направляет TCP-трафик порта `5432` на PgBouncer;
по умолчанию IP выделяется провайдером.

Начальное состояние рассчитано на запись в SPB. Переключение ролей выполняйте
по процедуре статьи через Git; endpoint приложения переключается отдельно.
