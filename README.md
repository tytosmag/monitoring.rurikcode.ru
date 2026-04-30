# Rurik Monitoring

Инфраструктурный проект мониторинга для игры **«Код Рюрика»**.

Проект поднимает отдельный monitoring-stack в Docker:

- Prometheus
- Grafana
- Loki
- Alertmanager

Игровой проект `rurikcode.ru` остается отдельным Docker-проектом и отдает метрики/логи через:

- `node_exporter`
- `cAdvisor`
- `postgres_exporter`
- `nginx_exporter`
- `alloy-agent`

---

## 1. Архитектура

```text
rurikcode.ru project

  frontend / gateway / backend services
  PostgreSQL
  Nginx

  node_exporter        :9100   → CPU / RAM / disk / network
  cAdvisor             :18080  → Docker container metrics
  postgres_exporter    :9187   → PostgreSQL metrics
  nginx_exporter       :9113   → Nginx metrics
  alloy-agent                  → Docker logs → Loki


rurik-monitoring project

  Prometheus           :9090   → metrics storage and alert rules
  Grafana              :3000   → dashboards
  Loki                 :3100   → log storage
  Alertmanager         :9093   → alert routing
```

---

## 2. Что мониторим

### Инфраструктура

- CPU
- RAM
- диск
- сеть
- состояние Docker-контейнеров
- рестарты контейнеров
- PostgreSQL
- Nginx

### Приложение

После добавления `/metrics` в backend-сервисы можно мониторить:

- доступность frontend/backend
- HTTP 5xx ошибки
- время ответа API
- ошибки авторизации
- количество регистраций
- количество логинов
- игровые события
- сохранение результатов в leaderboard

### Логи

Через Loki собираются:

- логи Node.js сервисов
- логи Nginx
- ошибки авторизации
- ошибки backend-сервисов
- события приложения

---

## 3. Структура проекта

```text
rurik-monitoring/
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── README.md
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       └── rurik-alerts.yml
├── alertmanager/
│   └── alertmanager.yml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasources.yml
│   │   └── dashboards/
│   │       └── dashboards.yml
│   └── dashboards/
├── loki/
│   └── loki.yml
└── data/
    ├── prometheus/
    ├── grafana/
    └── loki/
```

---

## 4. Подготовка `.env`

Скопировать шаблон:

```bash
cp .env.example .env
```

Пример `.env` для локальной разработки:

```env
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin

PROMETHEUS_PORT=9090
GRAFANA_PORT=3000
LOKI_PORT=3100
ALERTMANAGER_PORT=9093

RURIK_APP_HOST=host.docker.internal

NODE_EXPORTER_PORT=9100
CADVISOR_PORT=18080
POSTGRES_EXPORTER_PORT=9187
NGINX_EXPORTER_PORT=9113

AUTH_SERVICE_PORT=3000
LEADERBOARD_SERVICE_PORT=3001
GAME_SERVICE_PORT=3002
```

Для production вместо `host.docker.internal` указывается IP сервера с игровым проектом:

```env
RURIK_APP_HOST=192.168.1.20
```

---

## 5. Запуск

Проверить конфигурацию:

```bash
docker compose config
```

Запустить stack:

```bash
docker compose up -d
```

Если используется старая команда:

```bash
docker-compose up -d
```

Проверить контейнеры:

```bash
docker compose ps
```

Ожидаемые контейнеры:

```text
rurik_monitoring_prometheus
rurik_monitoring_grafana
rurik_monitoring_loki
rurik_monitoring_alertmanager
```

---

## 6. Web-интерфейсы

| Сервис | URL |
|---|---|
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 |
| Loki ready endpoint | http://localhost:3100/ready |
| Alertmanager | http://localhost:9093 |

Доступ в Grafana по умолчанию:

```text
login: admin
password: admin
```

Пароль задается в `.env`:

```env
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin
```

Для production пароль нужно заменить.

---

## 7. Проверка Prometheus

Открыть:

```text
http://localhost:9090/targets
```

Должны быть `UP`:

```text
prometheus
rurik_node
rurik_cadvisor
rurik_postgres
rurik_nginx
```

Могут быть временно `DOWN`, пока не добавлены `/metrics` endpoints в Node.js сервисы:

```text
rurik_auth_service
rurik_leaderboard_service
rurik_game_service
```

---

## 8. Проверка exporters со стороны игрового проекта

На сервере/машине с `rurikcode.ru`:

```bash
curl -s http://localhost:9100/metrics | grep node_cpu_seconds_total | head
curl -s http://localhost:18080/metrics | grep container_cpu_usage_seconds_total | head
curl -s http://localhost:9187/metrics | grep pg_up
curl -s http://localhost:9113/metrics | grep nginx_up
```

Ожидаемый результат:

```text
pg_up 1
nginx_up 1
```

---

## 9. Проверка Loki

Проверить готовность Loki:

```bash
curl -s http://localhost:3100/ready
```

Ожидаемо:

```text
ready
```

Если в игровом проекте `alloy-agent` настроен на этот Loki, в `.env` игрового проекта должно быть:

```env
LOKI_URL=http://localhost:3100/loki/api/v1/push
```

Если monitoring-stack находится на отдельном сервере:

```env
LOKI_URL=http://192.168.1.50:3100/loki/api/v1/push
```

После изменения `LOKI_URL` перезапустить Alloy в проекте игры:

```bash
cd /path/to/rurikcode.ru
docker compose restart alloy-agent
```

Проверить логи Alloy:

```bash
docker logs rurik_alloy_agent --tail=100
```

---

## 10. Проверка логов в Grafana

Открыть Grafana:

```text
http://localhost:3000
```

Перейти:

```text
Explore → Loki
```

Примеры LogQL-запросов:

```logql
{project="rurikcode"}
```

```logql
{job="rurikcode-docker"}
```

```logql
{job="rurikcode-docker"} |= "error"
```

```logql
{job="rurikcode-docker"} |= "auth"
```

```logql
{job="rurikcode-docker"} |= "login"
```

---

## 11. Grafana datasources

Datasource provisioning лежит здесь:

```text
grafana/provisioning/datasources/datasources.yml
```

Подключаются два источника:

```text
Prometheus
Loki
```

Проверить в интерфейсе Grafana:

```text
Connections → Data sources
```

---

## 12. Alertmanager

Alertmanager доступен по адресу:

```text
http://localhost:9093
```

Конфигурация:

```text
alertmanager/alertmanager.yml
```

Пока используется базовый receiver:

```yaml
receivers:
  - name: "default"
```

На первом этапе алерты отображаются в UI Alertmanager без отправки в Telegram/email.

Позже можно добавить:

- Telegram
- Email
- Webhook
- Slack-compatible endpoint

---

## 13. Prometheus alert rules

Файл правил:

```text
prometheus/rules/rurik-alerts.yml
```

Базовые алерты:

```text
RurikTargetDown
RurikHighCpuUsage
RurikHighMemoryUsage
RurikDiskAlmostFull
RurikPostgresDown
RurikNginxDown
RurikBackendTargetDown
RurikHighHttp5xxRate
RurikSlowApiResponses
```

Проверить правила в Prometheus:

```text
http://localhost:9090/rules
```

Проверить активные алерты:

```text
http://localhost:9090/alerts
```

---

## 14. Перезагрузка Prometheus после изменения правил

Если менялись файлы:

```text
prometheus/prometheus.yml
prometheus/rules/*.yml
```

Можно перезапустить Prometheus:

```bash
docker compose restart prometheus
```

Или, если включен lifecycle endpoint:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## 15. Полезные PromQL-запросы

### CPU usage

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{job="rurik_node",mode="idle"}[5m])) * 100)
```

### Memory usage

```promql
(1 - (node_memory_MemAvailable_bytes{job="rurik_node"} / node_memory_MemTotal_bytes{job="rurik_node"})) * 100
```

### Disk usage

```promql
100 - ((node_filesystem_avail_bytes{job="rurik_node",fstype!~"tmpfs|overlay|squashfs"} * 100) / node_filesystem_size_bytes{job="rurik_node",fstype!~"tmpfs|overlay|squashfs"})
```

### PostgreSQL status

```promql
pg_up{job="rurik_postgres"}
```

### Nginx status

```promql
nginx_up{job="rurik_nginx"}
```

### Nginx requests

```promql
rate(nginx_http_requests_total{job="rurik_nginx"}[5m])
```

### Container CPU

```promql
rate(container_cpu_usage_seconds_total{job="rurik_cadvisor"}[5m])
```

### Container memory

```promql
container_memory_usage_bytes{job="rurik_cadvisor"}
```

---

## 16. План следующих этапов

### Этап 1. Infrastructure monitoring

Готово:

- CPU / RAM / disk
- Docker containers
- PostgreSQL
- Nginx
- Loki logs
- базовые alerts

### Этап 2. Application metrics

Нужно добавить `/metrics` в backend-сервисы:

```text
auth-service
leaderboard-service
game-service
```

Метрики:

```text
http_requests_total
http_request_duration_seconds
rurik_auth_login_total
rurik_auth_login_failed_total
rurik_auth_registration_total
rurik_game_started_total
rurik_game_completed_total
rurik_leaderboard_score_saved_total
```

### Этап 3. Dashboards

Добавить Grafana dashboards:

```text
Rurik Infrastructure
Rurik Docker Containers
Rurik PostgreSQL
Rurik Nginx
Rurik Backend API
Rurik Auth Metrics
Rurik Game Events
Rurik Logs
```

### Этап 4. Alert delivery

Добавить отправку алертов:

```text
Telegram
Email
Webhook
```

---

## 17. Остановка

Остановить stack:

```bash
docker compose down
```

Остановить stack и удалить локальные данные:

```bash
docker compose down -v
```

Важно: `-v` удалит данные Prometheus, Grafana и Loki.

---

## 18. Безопасность

Не рекомендуется публиковать наружу без защиты:

```text
Prometheus    :9090
Grafana       :3000
Loki          :3100
Alertmanager  :9093
```

Для production использовать:

- VPN / WireGuard
- reverse proxy с HTTPS
- basic auth
- firewall allowlist
- закрытый доступ только с админских IP

---

## 19. Git

`.env` и `data/` не коммитим.

`.gitignore`:

```gitignore
.env
data/
*.log
```

Коммитим:

```text
docker-compose.yml
.env.example
README.md
prometheus/
alertmanager/
grafana/
loki/
.gitignore
```

Первый коммит:

```bash
git init
git add docker-compose.yml .env.example README.md prometheus/ alertmanager/ grafana/ loki/ .gitignore
git commit -m "Initial monitoring stack"
```

---

## 20. Быстрый check

```bash
docker compose config
docker compose up -d
docker compose ps

curl -s http://localhost:9090/-/ready
curl -s http://localhost:3100/ready
curl -s http://localhost:9093/-/ready
```

Ожидаемо:

```text
Prometheus: ready
Loki: ready
Alertmanager: ready
```