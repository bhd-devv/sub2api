# Розгортання в Coolify з коду репозиторію

[`docker-compose.coolify.yml`](../docker-compose.coolify.yml) збирає застосунок
через кореневий [`Dockerfile`](../Dockerfile): Vue frontend вбудовується в Go
backend. PostgreSQL 18 і Redis 8 запускаються поруч. Залежності збірки
завантажуються з офіційних npm і Go registry.

## Налаштування Coolify

1. Створи ресурс **Application** із Git-репозиторію та вибери потрібну гілку.
2. Вибери **Build Pack: Docker Compose**.
3. Задай **Base Directory**: `/`.
4. Задай **Docker Compose Location**: `/docker-compose.coolify.yml`.
5. Збережи конфігурацію та заповни **Environment Variables** з таблиці нижче.
   Секрети потрібні під час запуску, не позначай їх як build variables.
6. Coolify автоматично генерує домен для `sub2api` через
   `SERVICE_URL_SUB2API_8080` і направляє запити на внутрішній порт `8080`.
   Щоб використати свій домен, відкрий **Domains → Add domain**, вибери
   `sub2api` і задай, наприклад, `https://api.example.com:8080`.
   Зовнішній HTTPS працює на звичайному порту `443`; суфікс `:8080`
   вказує проксі порт контейнера. DNS власного домену має вести на сервер Coolify.
7. Натисни **Deploy**. Після готовності PostgreSQL і Redis застосунок виконає
   міграції та створить адміністратора. Відкрий налаштований домен.

| Змінна | Значення |
|--------|----------|
| `POSTGRES_PASSWORD` | Окремий випадковий пароль PostgreSQL |
| `REDIS_PASSWORD` | Окремий випадковий пароль Redis |
| `ADMIN_EMAIL` | Email першого адміністратора |
| `ADMIN_PASSWORD` | Початковий пароль адміністратора |
| `JWT_SECRET` | Сталий випадковий секрет, щонайменше 32 символи |
| `TOTP_ENCRYPTION_KEY` | Сталий ключ із 64 hex-символів |

Для кожного пароля або ключа можна окремо виконати `openssl rand -hex 32`.
Збережи значення JWT і TOTP між розгортаннями, щоб зберегти сесії та доступ
через 2FA. Початкові `ADMIN_*` створюють користувача лише під час першого
налаштування; зміна цих змінних не скидає пароль наявного адміністратора.

Необов'язкові змінні: `POSTGRES_USER` і `POSTGRES_DB` (типово `sub2api`),
`TZ` (типово `Europe/Kyiv`), `RUN_MODE` (типово `standard`),
`SETUP_MIGRATION_TIMEOUT_SECONDS` та `MCP_PUBLIC_URL`.
Для додаткових параметрів застосунку додай відповідні змінні в
`services.sub2api.environment` за зразком наявних змінних і задай їх значення
в **Environment Variables** Coolify.

## Оновлення та дані

Кожне розгортання збирає образ із поточного checkout вибраної гілки.
Для автоматичного розгортання після push увімкни **Auto Deploy** і webhook
Git-провайдера в Coolify. Оновлюй застосунок через Git і Coolify; вбудований
механізм оновлення з upstream-релізів може замінити зміни цього репозиторію.

Томи `sub2api_data`, `postgres_data` і `redis_data` зберігають конфігурацію,
базу даних і Redis AOF між перезапусками та розгортаннями того самого ресурсу.
Налаштуй резервне копіювання перед перенесенням або видаленням ресурсу.
Застосунок доступний проксі на `8080`; порти на хості не публікуються.
Coolify керує іменами контейнерів і підключенням проксі до мережі Compose.
Перевірка готовності застосунку: `GET /health` на порту `8080`.
Налаштування доменів відповідає
[документації Docker Compose Coolify](https://coolify.io/docs/applications/builds/docker-compose).

## Локальна перевірка та збірка

Після експорту обов'язкових змінних у поточну оболонку:

```sh
docker compose -f docker-compose.coolify.yml config --quiet
docker compose -f docker-compose.coolify.yml build sub2api
docker compose -f docker-compose.coolify.yml up -d
docker compose -f docker-compose.coolify.yml ps
```

Локальний Compose не налаштовує домен і проксі Coolify. Готовність можна
перевірити всередині контейнера:

```sh
docker compose -f docker-compose.coolify.yml exec sub2api \
  wget -qO- http://127.0.0.1:8080/health
```

Якщо PostgreSQL і Redis уже керуються окремими ресурсами Coolify, можна
використати **Dockerfile** build pack із кореневими context `/` і Dockerfile
`/Dockerfile`, портом `8080` та постійним томом `/app/data`. У цьому варіанті
передай `AUTO_SETUP=true`, `SERVER_HOST=0.0.0.0`, `SERVER_PORT=8080`,
`DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USER`, `DATABASE_PASSWORD`,
`DATABASE_DBNAME`, `DATABASE_SSLMODE`, `REDIS_HOST`, `REDIS_PORT`,
`REDIS_PASSWORD`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `JWT_SECRET`,
`TOTP_ENCRYPTION_KEY` відповідно до своїх ресурсів.
