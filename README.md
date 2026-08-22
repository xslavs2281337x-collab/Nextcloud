# Nextcloud

Self-hosted файловое хранилище. Docker Compose, PHP-FPM за nginx, PostgreSQL, Redis.

**URL:**

---



## Расположение

| Параметр | Значение |
|---|---|
| Хост / гипервизор | |
| IP-адрес ВМ | |
| Подсеть docker-сети | |
| Расположение стека на диске | |
| Каталог пользовательских данных | |
| Диск под данные | |
| Ответственный | |

---

## Архитектура

Целевая схема:

```
клиент ──HTTPS/443──>─────          
                          │     
                          └──> web (nginx в контейнере)
                               │ FastCGI
                             app (php-fpm)
                               ├──> db     PostgreSQL
                               └──> redis  локи + кеш

                             cron ──> тот же образ, entrypoint /cron.sh
```

Сейчас:

```
клиент ──HTTPS──> https://cloud.loesk.ru ──> web ──> app ──> db / redis
```

Два nginx с разными ролями: контейнерный `web` разговаривает с PHP-FPM, хостовый — только проксирует и держит TLS.

---

## Состав

| Сервис | Образ | Назначение |
|---|---|---|
| `app` | | Nextcloud, PHP-FPM на :9000 |
| `cron` | тот же, что `app` | Фоновые задачи, `/cron.sh` |
| `web` | | nginx → FastCGI |
| `db` | | PostgreSQL |
| `redis` | | File locking + кеш |

Версия `app` и `cron` совпадает всегда — общий том с кодом.

---

## Структура каталога

```
<путь>/
├── compose.yaml
├── db.env          # креды PostgreSQL, не в git
└── web/
    └── nginx.conf  # подменяет /etc/nginx/nginx.conf целиком
```

Тома:

| Том | Что внутри |
|---|---|
| `nextcloud` | Код, `config/`, `apps/`, `themes/` |
| `db` | Данные PostgreSQL |

---

## Запуск

```bash
cd <путь к стеку>

docker compose config      # проверить конфигурацию
docker compose up -d       # поднять
docker compose ps          # состояние, ждать healthy
docker compose logs -f app # логи приложения
```

Первый старт долгий — распаковка кода в том.

```bash
docker compose down        # тома сохраняются
docker compose down -v     # ВМЕСТЕ С ДАННЫМИ
```

---

## Управление через occ

Всё администрирование — утилитой `occ`, от пользователя `www-data`:

```bash
docker compose exec -u www-data app php occ status
docker compose exec -u www-data app php occ config:list system
docker compose exec -u www-data app php occ user:list
docker compose exec -u www-data app php occ files:scan --all
docker compose exec -u www-data app php occ maintenance:mode --on
```

---

## Конфигурация

`config/config.php` внутри тома `nextcloud`.

| Параметр | Назначение |
|---|---|
| `trusted_domains` | Домены, с которых разрешён вход |
| `trusted_proxies` | Подсеть реверс-прокси. Указывать точно, не весь /12 |
| `overwriteprotocol` | `https`, когда TLS терминирует прокси |
| `overwrite.cli.url` | Внешний URL для фоновых задач |
| `memcache.local` | APCu |
| `memcache.distributed` | Redis |
| `memcache.locking` | Redis. В БД — не оставлять |

**Заполнено:**

```

```

---

## Сеть и доступ

| Пункт | Значение |
|---|---|
| Порт наружу | |
| Порт контейнера web | 127.0.0.1:8080 |
| Правила nftables | |
| Откуда разрешён доступ | |

---

## Обслуживание

### Бэкап

Три компонента, снимать **одномоментно** — иначе файлы на диске окажутся неизвестны базе:

1. `config.php`
2. дамп PostgreSQL
3. каталог данных

```bash
docker compose exec -u www-data app php occ maintenance:mode --on
docker compose exec db pg_dump -U nextcloud nextcloud > backup/nc-$(date +%F).sql
# rsync данных
docker compose exec -u www-data app php occ maintenance:mode --off
```

**Расписание:**

**Куда складывается:**

**Когда проверялось восстановление:**

### Обновление

Только по одному мажору: 31 → 32 → 33. Прыжок через версию ломает инсталляцию.

```bash
# 1. Бэкап
# 2. Поднять тег в compose.yaml у app И cron одновременно
docker compose pull
docker compose up -d
docker compose logs -f app
docker compose exec -u www-data app php occ status
```

---

## Диагностика

| Симптом | Куда смотреть |
|---|---|
| Редирект на http, битые ссылки | `overwriteprotocol`, `fastcgi_param HTTPS on` в web/nginx.conf |
| «Access through untrusted domain» | `trusted_domains` |
| Ложные баны brute-force | `trusted_proxies`, заголовок `X-Forwarded-For` на прокси |
| `413 Request Entity Too Large` | `client_max_body_size` — в обоих nginx |
| `Lock timeout` при синхронизации | Redis не подключён, локи ушли в БД |
| Растёт диск, корзина не чистится | Контейнер `cron` не работает |
| Стек не поднимается | Healthcheck `db` или `redis` не проходит — `docker compose ps` |
| Контейнер в цикле рестарта | `docker compose logs`, права на том |

Чеклист в админке: **Настройки → Основные сведения**. Довести до нуля предупреждений.

---

## Заметки

```

```
