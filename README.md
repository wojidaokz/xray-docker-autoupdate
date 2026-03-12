# Xray Docker с автообновлением

Готовое решение для запуска [Xray](https://github.com/XTLS/Xray-core) прокси-сервера в Docker-контейнере с автоматическим обновлением через [Watchtower](https://github.com/containrrr/watchtower).

## Что включено

| Компонент | Описание |
|-----------|----------|
| **Xray** | Прокси-сервер на базе официального образа `ghcr.io/xtls/xray-core` |
| **Watchtower** | Автоматически проверяет наличие новых версий образа Xray раз в сутки, обновляет контейнер и удаляет старые образы |

## Требования

- Linux-сервер (Ubuntu 20.04+, Debian 11+, CentOS 8+ или аналогичный)
- Docker Engine 20.10+
- Docker Compose v2+
- Открытый порт 443 (или другой, указанный в конфигурации)
- Доменное имя (опционально, зависит от выбранного протокола)

## Установка

### 1. Установка Docker и Docker Compose

Если Docker ещё не установлен:

```bash
curl -fsSL https://get.docker.com | sh
```

Docker Compose v2 устанавливается вместе с Docker Engine. Проверьте:

```bash
docker compose version
```

### 2. Клонирование репозитория

```bash
git clone https://github.com/wojidaokz/xray-docker-autoupdate.git
cd xray-docker-autoupdate
```

### 3. Генерация ключей

Сгенерируйте UUID для клиента:

```bash
docker run --rm ghcr.io/xtls/xray-core:latest xray uuid
```

Сгенерируйте пару ключей для Reality:

```bash
docker run --rm ghcr.io/xtls/xray-core:latest xray x25519
```

Команда выведет `Private key` и `Public key`. Сохраните оба — приватный ключ нужен для сервера, публичный — для клиента.

Сгенерируйте Short ID (случайный hex, 8 символов):

```bash
openssl rand -hex 8
```

### 4. Настройка конфигурации

Откройте файл `config.json` и замените плейсхолдеры:

```bash
nano config.json
```

| Плейсхолдер | Что подставить |
|---|---|
| `YOUR-UUID-HERE` | UUID, сгенерированный на шаге 3 |
| `YOUR-PRIVATE-KEY-HERE` | Private key из вывода `xray x25519` |
| `YOUR-SHORT-ID-HERE` | Short ID, сгенерированный на шаге 3 |

#### Пример заполненного конфига (значения вымышленные):

```json
{
  "clients": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "flow": "xtls-rprx-vision"
    }
  ]
}
```

```json
{
  "privateKey": "aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789abcdef0",
  "shortIds": ["a1b2c3d4e5f6g7h8"]
}
```

### 5. Запуск

```bash
docker compose up -d
```

Проверьте, что оба контейнера запущены:

```bash
docker compose ps
```

Ожидаемый вывод:

```
NAME        IMAGE                            STATUS
xray        ghcr.io/xtls/xray-core:latest    Up
watchtower  containrrr/watchtower            Up
```

## Настройка клиента

Для подключения к серверу на клиентской стороне используйте следующие параметры:

| Параметр | Значение |
|---|---|
| Протокол | VLESS |
| Адрес | IP-адрес или домен вашего сервера |
| Порт | 443 |
| UUID | Тот же UUID, что в `config.json` |
| Flow | `xtls-rprx-vision` |
| Безопасность | Reality |
| SNI | `www.google.com` |
| Public Key | Публичный ключ из вывода `xray x25519` |
| Short ID | Тот же Short ID, что в `config.json` |
| Fingerprint | `chrome` |

### Рекомендуемые клиенты

| Платформа | Клиент |
|---|---|
| Windows | [Hiddify](https://github.com/hiddify/hiddify-app), [v2rayN](https://github.com/2dust/v2rayN) |
| macOS | [Hiddify](https://github.com/hiddify/hiddify-app), [V2BOX](https://apps.apple.com/app/v2box-v2ray-client/id6446814690) |
| Linux | [Hiddify](https://github.com/hiddify/hiddify-app), [Nekoray](https://github.com/MatsuriDayo/nekoray) |
| Android | [Hiddify](https://github.com/hiddify/hiddify-app), [v2rayNG](https://github.com/2dust/v2rayNG) |
| iOS | [Hiddify](https://github.com/hiddify/hiddify-app), [Streisand](https://apps.apple.com/app/streisand/id6450534064) |

## Управление

### Основные команды

```bash
# Запуск
docker compose up -d

# Остановка
docker compose down

# Перезапуск Xray (например, после изменения конфига)
docker compose restart xray

# Просмотр логов Xray
docker compose logs xray

# Просмотр логов Watchtower
docker compose logs watchtower

# Просмотр логов в реальном времени
docker compose logs -f xray
```

### Проверка работы автообновления

Watchtower проверяет наличие обновлений раз в 24 часа (86400 секунд). Чтобы принудительно проверить обновление:

```bash
docker compose restart watchtower
```

Посмотреть лог обновлений:

```bash
docker compose logs watchtower
```

При успешном обновлении в логах будет запись вида:

```
Found new ghcr.io/xtls/xray-core:latest image
Stopping xray ...
Creating xray ...
Removing old image ...
```

## Логи Xray

Логи доступны в директории `./logs/`:

```bash
# Лог доступа
cat logs/access.log

# Лог ошибок
cat logs/error.log
```

Для изменения уровня логирования отредактируйте `config.json`:

| Уровень | Описание |
|---|---|
| `none` | Логирование отключено |
| `error` | Только ошибки |
| `warning` | Предупреждения и ошибки |
| `info` | Информационные сообщения |
| `debug` | Детальная отладочная информация |

## Настройка интервала обновлений

По умолчанию Watchtower проверяет обновления раз в сутки. Чтобы изменить интервал, отредактируйте `WATCHTOWER_POLL_INTERVAL` в `docker-compose.yml`:

| Интервал | Значение (секунды) |
|---|---|
| Каждые 6 часов | `21600` |
| Каждые 12 часов | `43200` |
| Раз в сутки | `86400` |
| Раз в неделю | `604800` |

## Структура проекта

```
xray-docker-autoupdate/
├── docker-compose.yml   # Конфигурация контейнеров
├── config.json          # Конфигурация Xray
├── logs/                # Директория логов (создаётся автоматически)
│   ├── access.log
│   └── error.log
├── .gitignore
└── README.md
```

## Безопасность

- Не публикуйте ваш `config.json` — он содержит приватные ключи
- Регулярно проверяйте логи на наличие подозрительной активности
- Используйте файрвол для ограничения доступа к серверу
- Рекомендуется настроить SSH-доступ по ключу и отключить вход по паролю

## Решение проблем

### Контейнер Xray не запускается

```bash
# Проверьте логи
docker compose logs xray

# Проверьте валидность конфигурации
docker run --rm -v $(pwd)/config.json:/etc/xray/config.json ghcr.io/xtls/xray-core:latest xray -test -config /etc/xray/config.json
```

### Порт 443 занят

Убедитесь, что никакой другой сервис не использует порт 443:

```bash
sudo ss -tlnp | grep 443
```

Если порт занят, остановите занимающий его процесс или измените порт в `config.json`.

### Watchtower не обновляет контейнер

Проверьте, что Watchtower имеет доступ к Docker-сокету:

```bash
docker compose logs watchtower
```

Убедитесь, что в `docker-compose.yml` корректно прописан volume `/var/run/docker.sock`.

## Лицензия

MIT
