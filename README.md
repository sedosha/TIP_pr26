# Седова Мария Александровна, ЭФМО-01-25

# Практическое занятие №10. Горизонтальное масштабирование: использование Load Balancer (NGINX)

# Цель занятия
Освоить базовый подход к горизонтальному масштабированию backend-приложения за счёт запуска нескольких экземпляров одного сервиса и распределения входящих HTTP-запросов через NGINX в роли балансировщика нагрузки.

## Архитектура решения

- **Backend-сервис** – HTTP-сервер на Go (tasks), реализующий эндпоинты:
  - `GET /health` – проверка состояния, возвращает JSON с полем `instance` и заголовок `X-Instance-ID`
  - `GET /v1/tasks` – возвращает список задач (статический), также добавляет заголовок `X-Instance-ID`
- **Балансировщик** – NGINX, настроенный через `upstream` на три реплики backend-сервиса.
- **Среда запуска** – Docker Compose:
  - три контейнера `tasks_1`, `tasks_2`, `tasks_3`
  - один контейнер `nginx_lb`, пробрасывающий порт 8080 наружу
- **Дополнительное задание** – добавлена третья реплика (`tasks_3`), все три участвуют в балансировке.

## Дерево проекта

<img width="795" height="477" alt="image" src="https://github.com/user-attachments/assets/233cee51-18dc-47f0-92e2-1059d42b27b4" />

## Конфигурация NGINX (upstream с тремя репликами)

```nginx
events {}

http {
    upstream tasks_backend {
        server tasks_1:8082;
        server tasks_2:8082;
        server tasks_3:8082;
    }

    server {
        listen 8080;
        location / {
            proxy_pass http://tasks_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Request-ID $request_id;
            proxy_set_header Authorization $http_authorization;
        }
    }
}
```

## Docker Compose (три реплики)

```yaml
version: "3.9"

services:
  tasks_1:
    build: ../../services/tasks
    container_name: tasks_1
    environment:
      APP_PORT: "8082"
      INSTANCE_ID: "tasks-1"

  tasks_2:
    build: ../../services/tasks
    container_name: tasks_2
    environment:
      APP_PORT: "8082"
      INSTANCE_ID: "tasks-2"

  tasks_3:
    build: ../../services/tasks
    container_name: tasks_3
    environment:
      APP_PORT: "8082"
      INSTANCE_ID: "tasks-3"

  nginx:
    image: nginx:1.27-alpine
    container_name: nginx_lb
    ports:
      - "8080:8080"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - tasks_1
      - tasks_2
      - tasks_3
```

## Выполнение работы 

### Запуск стенда

```bash
cd /root/pz10-load-balancer/deploy/lb
docker compose up -d --build
```
<img width="900" height="189" alt="image" src="https://github.com/user-attachments/assets/783fd584-4f0d-461d-b551-caeb06debc44" />


### Статус контейнеров

```bash
docker compose ps
```

<img width="1330" height="163" alt="image" src="https://github.com/user-attachments/assets/14e23e07-b1f3-4f1c-850a-435d80e3235b" />


### Проверка балансировки (health endpoint)

```bash
curl -i http://localhost:8080/health
```

<img width="900" height="969" alt="image" src="https://github.com/user-attachments/assets/94cec3ee-4f42-46e6-b381-72d077177ee7" />

Заголовок X-Instance-ID меняется между tasks-1, tasks-2, tasks-3

### Проверка основного эндпоинта `/v1/tasks`

```bash
curl -i http://localhost:8080/v1/tasks
```

**Аналогичный результат** – заголовки распределяются между `tasks-1`, `tasks-2`, `tasks-3`.

**Скриншот 4** – Чередование экземпляров при вызове `/v1/tasks`.

### 6.5. Проверка отказоустойчивости (остановка одной реплики)

```bash
docker compose stop tasks_1
```

Повторяем серию запросов:

```bash
for i in {1..6}; do curl -s -D - http://localhost:8080/v1/tasks -o /dev/null | grep "X-Instance-ID"; done
```

**Вывод (только `tasks-2` и `tasks-3`):**
```
X-Instance-ID: tasks-2
X-Instance-ID: tasks-3
X-Instance-ID: tasks-2
X-Instance-ID: tasks-3
...
```

**Скриншот 5** – После остановки `tasks_1` ответы приходят только от `tasks-2` и `tasks-3`.

### 6.6. Восстановление реплики

```bash
docker compose start tasks_1
```

Снова выполняем запросы:

```bash
for i in {1..6}; do curl -s -D - http://localhost:8080/v1/tasks -o /dev/null | grep "X-Instance-ID"; done
```

**Вывод** – снова видны все три `tasks-1`, `tasks-2`, `tasks-3`.

**Скриншот 6** – Балансировка восстановлена, все три реплики активны.
**Ключевые выводы:**
- Для горизонтального масштабирования сервис должен быть **stateless** (в учебном примере – статический список задач, но в реальности состояние должно храниться в общей БД или Redis).
- Balancer скрывает внутреннюю архитектуру от клиента, обеспечивая единый адрес.
- Health endpoint – обязательный элемент для мониторинга доступности экземпляров.
