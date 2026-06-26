# Проектная работа дисциплины по Docker

Используются бэкенд на Go (порт `8081`) и фронтенд на Vue.js,
который раздаётся через nginx. Оркестрация — Docker Compose.

## Как запускать

```bash
docker compose up --build
```

- Доступ к frontend: <http://localhost:80>
- Доступ к backend идет через nginx-балансировщик: <http://localhost:8081>

Как остановить и очистить:

```bash
docker compose down
```

## Описание сервисов

| Сервис         | Назначение                              | Публикуемый порт |
|----------------|-----------------------------------------|------------------|
| `frontend`     | Собранный Vue SPA на unprivileged nginx | `80 -> 8080`     |
| `backend`      | Go API, хранилище в памяти              | только внутренний|
| `backend-lb`   | nginx-балансировщик перед `backend`     | `8081 -> 8080`   |
| `frontend-dev` | Dev-сервер Vue (профиль `dev`)          | `8080`           |

Сеть `backend` помечена `internal: true`, поэтому backend не доступен с хоста
напрямую и весь трафик к нему идёт через `backend-lb`.

### Как разрабатывать frontend

```bash
docker compose --profile dev up --build frontend-dev
```

### Как масштабировать backend

`backend-lb` распределяет нагрузку по всем репликам `backend`, которые
определены на старте:

```bash
docker compose up --build --scale backend=3
```

## Конфигурация

Аргументы сборки (со значениями по умолчанию):

- `GO_VERSION` (`1.26`), `ALPINE_VERSION` (`3.24`) — базовые образы backend.
- `NODE_VERSION` (`24.18`), `NODE_ALPINE_VERSION` (`3.24`),
  `NGINX_VERSION` (`1.31.2`), `NGINX_ALPINE_VERSION` (`3.23`) — образы frontend.
- `VUE_APP_API_URL` (`http://localhost:8081`) — адрес API, зашиваемый в SPA.
- `PUBLIC_PATH` (`/`) — базовый путь, с которого раздаются ассеты SPA.

Во время выполнения:

- `APP_SECRET_FILE` — путь к файлу Docker-секрета
  (по умолчанию `./secrets/app_secret.example.txt`).

## Работа с секретами

Backend получает `app_secret` как Docker секрет, который монтируется в
`/run/secrets/app_secret` и никогда не зашивается в образ. Свежий клон
запускается из коробки с `secrets/app_secret.example.txt`. Для реального
локального секрета:

```bash
echo -n 'secret' > secrets/app_secret.txt
APP_SECRET_FILE=./secrets/app_secret.txt docker compose up --build
```

Файл `secrets/app_secret.txt` не добавлен в `.gitignore` по требованиям 
его неизменности.

## Описание образов

Оба образа используют многоступенчатую сборку и работают не от root:

- backend- статический бинарник Go в минимальном образе Alpine (`USER app`);
- frontend - сборка Vue, раздаваемая из `nginxinc/nginx-unprivileged`
  (`USER 101`).

```bash
docker images 'momo-store-*'
```
### Дополнительная оптимизация

- **Backend.** Бинарник собирается статически (`CGO_ENABLED=0`) и копируется
  в минимальный образ Alpine. Флаги `-ldflags="-s -w"` убирают таблицу
  символов и отладочную информацию DWARF (бинарник меньше примерно на 30%),
  `-trimpath` убирает пути сборочной машины. В итоговый образ попадает только
  бинарник — без Go-тулчейна и исходников.
- **Frontend.** Node.js и dev-зависимости остаются в builder-стадии; в финальный
  nginx-образ копируется только собранный `dist`.
- **Контекст сборки.** `.dockerignore` исключает `node_modules` и `dist`, чтобы
  они не копировались и не попадали в builder-стадию через `COPY . .`. На
  размер конечного образа это не влияет, но ускоряет сборку.

## Безопасность

- Непривилегированные пользователи, `read_only` корневая ФС, `tmpfs` для `/tmp`.
- `cap_drop: ALL`, `no-new-privileges`, `pids_limit`, `mem_limit`, `cpus`.
- Наружу публикуются только порты `80` и `8081`; сеть бэкенда внутренняя.
- Заголовки безопасности и CSP заданы в конфиге nginx фронтенда.

Локальная проверка уязвимостей:

```bash
trivy image momo-store-backend:local
trivy image momo-store-frontend:local
```

## Описание GitHub CI/CD

GitHub Actions находится в файле `.github/workflows/deploy.yaml` и выполняет:

- собирает и пушит образы в Docker Hub;
- поднимает стек через Docker Compose и ждёт, пока он не станет healthy;
- сканирует собранные образы через Trivy (`CRITICAL,HIGH`).

Необходимые секреты репозитория: `DOCKER_USER`, `DOCKER_PASSWORD`, `APP_SECRET`.
