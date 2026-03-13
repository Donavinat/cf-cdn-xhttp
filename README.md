# CF CDN XHTTP VPN — VPN через Cloudflare CDN

Рабочий способ обхода DPI (ТСПУ) в России с использованием XHTTP (splithttp) транспорта через Cloudflare CDN.

**Проверено и работает на март 2026 года.**

---

## Как это работает

```
Клиент (v2rayNG/NekoBox)
    │
    ▼
HTTPS:443 → Cloudflare CDN (TLS termination)
    │
    ▼
HTTP:80 → Ваш сервер (Xray XHTTP inbound)
    │
    ▼
Интернет
```

### Принцип работы

- **XHTTP (splithttp)** — транспорт Xray, который использует обычные HTTP POST/GET запросы вместо WebSocket
- DPI **не может** обнаружить VPN-трафик — он выглядит как обычный просмотр сайта
- Cloudflare CDN **нельзя заблокировать** без поломки половины интернета (за CF стоят миллионы сайтов)
- Ваш домен (например, `cdn.cookglobe.ru`) выглядит безобидно для DPI — просто сайт за CDN
- **Flexible SSL**: Cloudflare сам терминирует TLS, до сервера трафик идёт по обычному HTTP:80
- Сервер не нуждается в сертификатах — CF всё берёт на себя

---

## Почему это работает (а WebSocket нет)

| | WebSocket | XHTTP (splithttp) |
|---|---|---|
| Протокол | WS Upgrade header | Обычные HTTP POST/GET |
| Детект DPI | Легко — по `Upgrade: websocket` | Невозможно — стандартный HTTP |
| ТСПУ блокировка | **Да**, блокирует WS upgrade | **Нет**, не отличить от сайта |
| CF лимиты | Worker: 30с на streaming | CDN: без лимитов |
| Стабильность | Разрывы из-за WS пингов | Стабильно — новый запрос каждый раз |

### Почему DPI не видит VPN:

1. **Клиент → CF**: обычный HTTPS на 443 порт, SNI = ваш домен. DPI видит "человек открыл сайт"
2. **CF → Сервер**: обычный HTTP на 80 порт. Провайдер сервера видит запросы от CF
3. **XHTTP** разбивает трафик на отдельные HTTP-запросы — нет постоянного соединения, нет паттернов VPN

---

## Что пробовали и не сработало

> Эти подходы были проверены лично. Не тратьте время — они НЕ работают.

### 1. VLESS + WebSocket через CF Worker
- **Проблема**: DPI (ТСПУ) блокирует WebSocket `Upgrade` заголовки с пользовательских устройств
- **Парадокс**: с сервера WS через CF Worker работает, с телефона/компа — нет
- **Вывод**: ISP фильтрует WS upgrade на уровне сессии пользователя

### 2. XHTTP через CF Worker
- **Проблема**: трафик доходит до сервера, но Worker убивает streaming response
- На бесплатном плане CF Worker отключает ответ через ~30 секунд
- CDN не имеет этого ограничения — там обычное проксирование
- **Вывод**: Workers ≠ CDN, Workers не подходят для VPN

### 3. CF CDN + Full SSL + TLS на сервере
- **Проблема**: Error 526 (Invalid SSL certificate)
- CF в режиме Full пытается установить TLS с сервером, но Let's Encrypt сертификат не совпадает с ожиданиями CF
- **Решение**: использовать Flexible SSL (CF→сервер по HTTP)

### 4. CF CDN + порт 8443
- **Проблема**: CF считает 8443 HTTPS-портом и пытается установить TLS с сервером
- Даже при Flexible SSL, порт 8443 = HTTPS в понимании CF
- **Решение**: использовать порт 80 (стандартный HTTP)

### 5. Custom domain на Workers (`cookglobe.ru` вместо `workers.dev`)
- **Проблема**: не помогло — WebSocket всё равно блокируется DPI
- DPI фильтрует по заголовкам, а не по домену
- **Вывод**: смена домена Worker не обходит блокировку WS

### 6. MTProto proxy
- **Проблема**: DPI детектирует и блокирует протокол MTProto
- Даже с obfs — паттерны трафика выдают прокси
- **Вывод**: MTProto не подходит для обхода ТСПУ

---

## Требования

- **VPS сервер** в любой стране за пределами России (Германия, Нидерланды, Финляндия и т.д.)
- **Домен**, добавленный в Cloudflare (бесплатный план работает!)
- **Xray** установленный на сервере (v1.8.24+, поддержка XHTTP/splithttp)
- **v2rayNG** или **NekoBox** на клиентском устройстве (Android/iOS/Windows/macOS)

---

## Настройка сервера (Xray)

### 1. Установка Xray

```bash
# Установка последней версии Xray
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### 2. Генерация UUID

```bash
xray uuid
```

Сохраните полученный UUID — он понадобится для конфига и клиента.

### 3. Конфигурация Xray

Отредактируйте `/usr/local/etc/xray/config.json`:

```json
{
  "log": {
    "loglevel": "warning",
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 80,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "YOUR-UUID-HERE",
            "flow": ""
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "xhttp",
        "xhttpSettings": {
          "mode": "auto",
          "path": "/",
          "noSSEHeader": true,
          "scMaxEachPostBytes": "1000000-2000000",
          "scMinPostsIntervalMs": "10-50",
          "xPaddingBytes": "100-1000"
        },
        "security": "none"
      },
      "sniffing": {
        "enabled": false
      },
      "tag": "inbound-xhttp-cdn"
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
```

#### Пояснения к настройкам:

| Параметр | Значение | Зачем |
|---|---|---|
| `port: 80` | HTTP порт | CF Flexible SSL отправляет трафик на 80. **Только 80!** |
| `security: "none"` | Без TLS | CF сам терминирует TLS, до сервера идёт HTTP |
| `network: "xhttp"` | XHTTP транспорт | splithttp — обычные HTTP запросы |
| `noSSEHeader: true` | Без SSE заголовков | Предотвращает проблемы с CDN проксированием |
| `scMaxEachPostBytes` | Размер POST | Рандомизация размера запросов для маскировки |
| `scMinPostsIntervalMs` | Интервал запросов | Рандомизация интервалов — имитация обычного браузинга |
| `xPaddingBytes` | Паддинг | Добавляет случайные байты — ломает паттерны DPI |
| `sniffing: false` | Без снифинга | Не нужен для этой конфигурации |
| `flow: ""` | Пустой | XHTTP не использует flow (это для XTLS) |

### 4. Открытие порта 80 в файрволе

```bash
# UFW
ufw allow 80/tcp

# Или iptables
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

### 5. Перезапуск Xray

```bash
systemctl restart xray
systemctl status xray
```

Убедитесь, что Xray слушает на порту 80:

```bash
ss -tlnp | grep :80
```

---

## Настройка Cloudflare

### 1. Добавление домена

- Зайдите на [dash.cloudflare.com](https://dash.cloudflare.com)
- Add a site → введите ваш домен → выберите **Free** план
- Обновите NS-записи у регистратора домена на указанные CF

### 2. DNS запись

- **DNS** → **Records** → **Add record**
  - Type: `A`
  - Name: `cdn` (или любое нейтральное имя)
  - IPv4 address: `IP вашего сервера`
  - Proxy status: **Proxied** (оранжевое облако — **ОБЯЗАТЕЛЬНО!**)
  - TTL: Auto

> **Важно!** Облако должно быть **оранжевым** (Proxied). Серое облако = DNS Only = трафик идёт напрямую, без CDN.

### 3. SSL/TLS настройки

- **SSL/TLS** → **Overview** → выберите **Flexible**

> **Flexible** означает: клиент→CF по HTTPS, CF→сервер по HTTP. Сервер не нуждается в сертификатах.

> **НЕ используйте Full или Full (Strict)!** Это вызовет Error 525/526, так как CF будет пытаться установить TLS с сервером.

### 4. Настройки безопасности

- **Security** → **Settings**:
  - Under Attack Mode: **OFF**
  - Bot Fight Mode: **OFF**
  - Security Level: **Essentially Off** или **Low**

> Если оставить Bot Fight Mode включённым, CF может показывать капчу мобильным клиентам, и VPN не подключится.

### 5. Дополнительно (опционально)

- **Speed** → **Optimization**:
  - Auto Minify: **OFF** (не ломайте бинарный трафик)
  - Brotli: **OFF**
  - Rocket Loader: **OFF**

---

## Ссылка для клиента

Формат VLESS ссылки:

```
vless://YOUR-UUID@cdn.yourdomain.com:443?encryption=none&security=tls&type=splithttp&path=/&sni=cdn.yourdomain.com&host=cdn.yourdomain.com#CDN-XHTTP
```

Замените:
- `YOUR-UUID` — ваш UUID из конфига Xray
- `cdn.yourdomain.com` — ваш поддомен в CF

### Пример:

```
vless://a1b2c3d4-e5f6-7890-abcd-ef1234567890@cdn.cookglobe.ru:443?encryption=none&security=tls&type=splithttp&path=/&sni=cdn.cookglobe.ru&host=cdn.cookglobe.ru#CDN-XHTTP
```

#### Разбор параметров:

| Параметр | Значение | Пояснение |
|---|---|---|
| `encryption` | `none` | VLESS не использует encryption поле |
| `security` | `tls` | Клиент→CF по TLS |
| `type` | `splithttp` | XHTTP транспорт (в клиентах называется splithttp) |
| `path` | `/` | Путь из конфига Xray |
| `sni` | `cdn.yourdomain.com` | TLS SNI — ваш домен |
| `host` | `cdn.yourdomain.com` | HTTP Host header |
| Порт | `443` | Стандартный HTTPS |

---

## Настройка клиента (v2rayNG)

### Android

1. Скопируйте VLESS ссылку
2. Откройте **v2rayNG**
3. Нажмите **+** → **Import config from clipboard**
4. Профиль появится в списке — нажмите на него для выбора
5. Нажмите кнопку **▶** (подключиться) внизу экрана
6. Откройте любой сайт для проверки

### Windows / macOS

1. Скачайте [v2rayN](https://github.com/2dust/v2rayN/releases) (Windows) или [v2rayU](https://github.com/yanue/V2rayU/releases) (macOS)
2. Скопируйте VLESS ссылку
3. Импортируйте из буфера обмена
4. Подключитесь

---

## Настройка клиента (NekoBox)

### Android

1. Скопируйте VLESS ссылку
2. Откройте **NekoBox**
3. **+** → **Import from clipboard**
4. Выберите профиль → **Connect**

### Примечание для NekoBox

В некоторых версиях NekoBox транспорт `splithttp` может называться `xhttp`. Если импорт не работает:

1. Создайте профиль вручную:
   - Protocol: VLESS
   - Address: `cdn.yourdomain.com`
   - Port: `443`
   - UUID: ваш UUID
   - Transport: SplitHTTP (или XHTTP)
   - Path: `/`
   - TLS: включен
   - SNI: `cdn.yourdomain.com`

---

## Важные моменты

### Порт — только 80!

Cloudflare поддерживает проксирование на ограниченный набор портов:
- HTTP: 80, 8080, 8880, 2052, 2082, 2086, 2095
- HTTPS: 443, 2053, 2083, 2087, 2096, 8443

При **Flexible SSL** CF отправляет трафик на **HTTP** порты. Порт 8443 CF считает **HTTPS** портом и будет пытаться установить TLS с сервером (что сломает всё). Используйте порт **80**.

### SSL — только Flexible!

| Режим | CF → Сервер | Работает? |
|---|---|---|
| **Flexible** | HTTP (порт 80) | **Да** |
| Full | HTTPS (нужен сертификат) | Нет — Error 525/526 |
| Full (Strict) | HTTPS (валидный сертификат) | Нет — Error 526 |

### Домен должен выглядеть нейтрально

- **Хорошо**: `cdn.cookglobe.ru`, `static.mysite.com`, `assets.example.org`
- **Плохо**: `vpn.myserver.com`, `proxy.tunnel.net`, `vless.bypass.ru`

DPI может фильтровать по подозрительным доменам.

### CDN ≠ Workers

| | CF CDN | CF Workers |
|---|---|---|
| Лимит ответа | Нет | ~30с на бесплатном плане |
| Streaming | Полноценное проксирование | Обрывается |
| Подходит для VPN | **Да** | Нет (для XHTTP) |
| Настройка | DNS + orange cloud | Код на JS |

### noSSEHeader: true

Параметр `noSSEHeader: true` в конфиге Xray **обязателен**. Без него сервер отправляет SSE (Server-Sent Events) заголовки, которые CF CDN может обрабатывать неправильно (буферизация, таймауты).

### Если на порту 80 уже есть веб-сервер

Если на сервере уже работает nginx/Apache на порту 80:

**Вариант 1**: Использовать другой HTTP-порт CF (например, 8080):
```json
"port": 8080
```
И в DNS создать запись на тот же IP — CF сам проксирует на 8080.

В ссылке клиента порт остаётся **443** (клиент→CF), а CF→сервер пойдёт на 8080.

> Примечание: нужно явно указать порт в CF Page Rules или использовать nginx reverse proxy на 80 → Xray.

**Вариант 2**: Nginx reverse proxy:
```nginx
server {
    listen 80;
    server_name cdn.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

В этом случае Xray слушает на `127.0.0.1:8080`, а nginx на 80 проксирует к нему.

---

## Диагностика

### Error 525: SSL handshake failed

**Причина**: CF пытается установить TLS с сервером, но сервер отвечает по HTTP.

**Решение**: Убедитесь, что SSL/TLS режим = **Flexible**.

### Error 526: Invalid SSL certificate

**Причина**: CF в режиме Full/Full(Strict) не принимает сертификат сервера.

**Решение**: Переключите на **Flexible**. Сервер не нуждается в TLS.

### "first record does not look like a TLS handshake"

**Причина**: CF отправляет HTTPS запрос, а Xray ожидает HTTP.

**Решение**: SSL режим НЕ Flexible. Переключите.

### 403 Forbidden с телефона

**Причина**: CF Bot Fight Mode блокирует запрос.

**Решение**:
1. Security → Bot Fight Mode: **OFF**
2. Security → Security Level: **Essentially Off**
3. Подождите 5 минут (кеш CF)

### Подключается, но сайты не открываются

**Возможные причины**:
1. Xray не запущен: `systemctl status xray`
2. Порт 80 закрыт: `ss -tlnp | grep :80`
3. Outbound не настроен: проверьте `"outbounds"` в конфиге (нужен `freedom`)
4. DNS проблемы: добавьте DNS outbound или используйте sniffing

### Медленная скорость

- XHTTP через CDN будет медленнее прямого подключения — это нормально
- Типичная скорость: 10-50 Мбит/с (зависит от CF PoP и сервера)
- Для ускорения: выберите сервер ближе к ближайшему CF дата-центру

---

## Проверка работы

### На сервере

```bash
# Проверить что Xray слушает на 80
ss -tlnp | grep :80

# Проверить логи Xray
journalctl -u xray -f

# Проверить что CF доходит до сервера
# (откройте https://cdn.yourdomain.com/ в браузере — должен быть ответ от Xray, обычно пустая страница или 400)
```

### С любого компьютера

```bash
# Проверить что домен резолвится через CF
nslookup cdn.yourdomain.com
# Должен показать IP из диапазона CF (104.x.x.x, 172.x.x.x), НЕ ваш сервер

# Проверить HTTP через CF
curl -v https://cdn.yourdomain.com/
# Ожидаемый ответ: HTTP 400 Bad Request (это нормально — XHTTP ожидает специальные запросы)
# Если 525/526 — проблема с SSL настройками
```

### На клиенте

1. Подключитесь к VPN
2. Откройте [https://ifconfig.me](https://ifconfig.me) — должен показать IP вашего сервера
3. Откройте [https://browserleaks.com/dns](https://browserleaks.com/dns) — проверьте DNS leak

---

## FAQ

**Q: Бесплатный план CF подходит?**
A: Да, бесплатного плана достаточно. CDN проксирование доступно на всех планах.

**Q: Могут ли заблокировать мой домен?**
A: Теоретически да, но это маловероятно. DPI видит только обращение к CF CDN. Если домен нейтральный — нет причин блокировать.

**Q: Это легально?**
A: VPN легален в большинстве стран. Проверьте законодательство вашей страны.

**Q: Сколько клиентов можно подключить?**
A: Ограничений нет. Один UUID может использоваться несколькими устройствами одновременно. Для разных пользователей лучше создать разные UUID.

**Q: Работает ли на iOS?**
A: Да, используйте Shadowrocket или Streisand. Импортируйте VLESS ссылку.

---

## Лицензия

MIT — используйте свободно.
