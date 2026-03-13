# АВАРИЙНОЕ ВОССТАНОВЛЕНИЕ И РЕЗЕРВНАЯ ИНФРАСТРУКТУРА

**Дата: 13 марта 2026**
**Для: Наташа (fintech)**

Этот документ — полное руководство на случай, если текущая инфраструктура перестанет работать.
Читай нужный сценарий и выполняй шаги по порядку.

---

## ОГЛАВЛЕНИЕ

1. [Мост упал/заблокирован — переключение на CF CDN](#сценарий-1)
2. [Домен cookglobe.ru заблокирован — новый домен](#сценарий-2)
3. [Приложения (Нави/Чат) нужно перенести на новый домен](#сценарий-3)
4. [Сервер 64 умер — VPN на другой сервер](#сценарий-4)
5. [Сервер 93 умер — приложения на другой сервер](#сценарий-5)
6. [Telegram-бот не работает](#сценарий-6)
7. [Что НЕ нужно делать (важно!)](#что-не-нужно-делать)
8. [Все рабочие ссылки VPN](#все-рабочие-ссылки-vpn)
9. [Контакты серверов](#контакты-серверов)
10. [Быстрая проверка что всё работает](#быстрая-проверка)

---

<a id="сценарий-1"></a>
## СЦЕНАРИЙ 1: Мост (81.90.31.187) упал или заблокирован

Мост — это российский VPS, который проксирует трафик. Если он умер или заблокирован — ничего страшного, есть два варианта.

### Вариант А: VPN через Cloudflare CDN (уже настроено и работает!)

Это самый простой вариант — просто переключись на другой профиль в v2rayNG.

**На телефоне (v2rayNG):**

1. Открой v2rayNG
2. Найди профиль **CookGlobe-CDN-XHTTP** (если его нет — добавь, см. ниже)
3. Нажми на него, чтобы выбрать
4. Нажми кнопку подключения (V внизу экрана)
5. Готово! VPN работает через Cloudflare CDN

**Если профиля нет — добавь ссылку:**

1. В v2rayNG нажми **+** (плюс) вверху справа
2. Выбери **Импорт из буфера обмена**
3. Скопируй эту ссылку (целиком, одной строкой):

```
vless://0663b071-0e5f-40aa-9b2d-5654a5dbff21@cdn.cookglobe.ru:443?encryption=none&security=tls&type=splithttp&path=/&sni=cdn.cookglobe.ru&host=cdn.cookglobe.ru#CookGlobe-CDN-XHTTP
```

4. Вставь и импортируй
5. Выбери этот профиль и подключись

**Как это работает:**
```
Телефон → HTTPS → Cloudflare CDN (cdn.cookglobe.ru) → HTTP → Сервер 64 → Интернет
```
Провайдер видит только обращение к Cloudflare — обычный HTTPS-трафик.

**Через VPN все приложения работают как обычно:**
- https://donavi-cors.fun — Нави
- https://chat.donavi-cors.fun — Нави Чат (или https://m.donavi-cors.fun)
- Telegram-бот @donavi_clawd_bot

---

### Вариант Б: Приложения напрямую через CF CDN (без VPN)

Если VPN не нужен, а нужен только доступ к Нави/Чат — можно пропустить весь трафик приложений через Cloudflare CDN.

**Шаги (в Cloudflare Dashboard):**

1. Открой https://dash.cloudflare.com
2. Войди в аккаунт
3. Выбери домен **donavi-cors.fun**
4. Перейди в раздел **DNS** → **Records**
5. Найди запись `donavi-cors.fun` (тип A, значение `93.185.157.128`)
6. Нажми **Edit** (карандаш)
7. Переключи **Proxy status** — нажми на серое облачко, чтобы оно стало **оранжевым**
8. Нажми **Save**
9. То же самое для `chat.donavi-cors.fun` и `m.donavi-cors.fun` — все на оранжевое облако
10. Перейди в раздел **SSL/TLS** → **Overview**
11. Убедись что режим **Full** (НЕ Flexible, НЕ Full Strict) — на сервере 93 есть сертификаты Let's Encrypt
12. Подожди 1-2 минуты
13. Проверь в браузере: https://donavi-cors.fun

**Важно:** после этого сайты будут работать через CF CDN. Когда мост починится — можно вернуть серые облачка обратно (но и с оранжевыми тоже всё работает).

---

<a id="сценарий-2"></a>
## СЦЕНАРИЙ 2: Домен cookglobe.ru заблокирован / нужен новый домен для VPN

Если cookglobe.ru заблокировали (или домен .ru не подходит), нужно купить новый домен и настроить CDN заново.

### Шаг 1: Купи новый домен на Cloudflare

1. Открой https://dash.cloudflare.com
2. В левом меню нажми **Domain Registration** → **Register Domains**
3. Введи любое нейтральное имя (НЕ содержащее слова vpn, proxy, tunnel!)
   - Хорошие примеры: `cookbox.fun`, `bakeshop.xyz`, `gardentool.click`
   - Плохие примеры: ~~vpnservice.fun~~, ~~proxyfast.xyz~~
4. Выбери самый дешевый домен ($1-2/год, зоны .fun, .xyz, .click)
5. Оплати картой
6. Домен автоматически добавится в Cloudflare

### Шаг 2: Подожди активации (5-15 минут)

Cloudflare сам настроит NS-записи (домен куплен у них).

### Шаг 3: Добавь DNS-запись

1. В Cloudflare выбери новый домен
2. Перейди в **DNS** → **Records**
3. Нажми **Add record**
4. Заполни:
   - Type: **A**
   - Name: **cdn**
   - IPv4 address: **64.188.118.135**
   - Proxy status: **Proxied** (оранжевое облако — обязательно!)
   - TTL: Auto
5. Нажми **Save**

### Шаг 4: Настрой SSL

1. Перейди в **SSL/TLS** → **Overview**
2. Выбери режим: **Flexible**
   (CF сам делает HTTPS для клиента, а к серверу ходит по HTTP на порт 80)

### Шаг 5: Отключи лишнюю защиту

1. Перейди в **Security** → **Settings**
2. **Security Level**: установи **Essentially Off** или **Low**
3. **Bot Fight Mode**: **OFF** (выключить!)
4. **Under Attack Mode**: убедись что **OFF**

### Шаг 6: На сервере 64 ничего менять НЕ нужно!

Xray уже слушает на порту 80 с правильными настройками. Он принимает любой домен.

### Шаг 7: Создай новую ссылку для v2rayNG

Просто замени `cdn.cookglobe.ru` на `cdn.НОВЫЙДОМЕН` в ссылке:

```
vless://0663b071-0e5f-40aa-9b2d-5654a5dbff21@cdn.НОВЫЙДОМЕН:443?encryption=none&security=tls&type=splithttp&path=/&sni=cdn.НОВЫЙДОМЕН&host=cdn.НОВЫЙДОМЕН#CDN-XHTTP
```

**Пример** (если купила домен `bakeshop.xyz`):
```
vless://0663b071-0e5f-40aa-9b2d-5654a5dbff21@cdn.bakeshop.xyz:443?encryption=none&security=tls&type=splithttp&path=/&sni=cdn.bakeshop.xyz&host=cdn.bakeshop.xyz#CDN-XHTTP
```

### Шаг 8: Добавь в v2rayNG

1. Скопируй новую ссылку
2. В v2rayNG: **+** → **Импорт из буфера обмена**
3. Подключись и проверь

---

<a id="сценарий-3"></a>
## СЦЕНАРИЙ 3: Приложения (Нави/Чат) нужно перенести на новый домен

Если домен donavi-cors.fun заблокирован и нужен новый домен для веб-приложений.

### Шаг 1: Купи новый домен (если ещё нет)

См. Сценарий 2, Шаг 1.

### Шаг 2: Добавь DNS-записи в Cloudflare

В разделе DNS нового домена добавь записи:

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| A | @ (корень) | 93.185.157.128 | Серое облако (DNS only) |
| A | chat | 93.185.157.128 | Серое облако (DNS only) |
| A | m | 93.185.157.128 | Серое облако (DNS only) |

Серое облако — потому что сервер 93 сам имеет SSL-сертификаты.
(Можно и оранжевое, но тогда SSL/TLS = Full)

### Шаг 3: Подключись к серверу 93 по SSH

Открой терминал на компьютере и выполни:

```bash
ssh root@93.185.157.128
```

Пароль: `yEwL5jvAu92N`

### Шаг 4: Добавь новый домен в nginx (Нави)

```bash
nano /etc/nginx/sites-enabled/navi-web
```

Найди строку `server_name donavi-cors.fun;` (она встречается 3 раза — в блоках listen 80, listen 443, и listen 8888).

В блоках listen 80 и listen 443 — добавь новый домен через пробел:

**Было:**
```
server_name donavi-cors.fun;
```

**Стало:**
```
server_name donavi-cors.fun НОВЫЙДОМЕН.com;
```

### Шаг 5: Добавь новый домен в nginx (Нави Чат)

```bash
nano /etc/nginx/sites-enabled/navi-chat
```

Найди `server_name chat.donavi-cors.fun;` (в обоих блоках — listen 80 и listen 443).

**Было:**
```
server_name chat.donavi-cors.fun;
```

**Стало:**
```
server_name chat.donavi-cors.fun chat.НОВЫЙДОМЕН.com;
```

### Шаг 6: Получи SSL-сертификаты

```bash
certbot certonly --nginx -d НОВЫЙДОМЕН.com -d chat.НОВЫЙДОМЕН.com -d m.НОВЫЙДОМЕН.com
```

Certbot спросит email (можно ввести любой) и согласие с условиями (Y).

### Шаг 7: Добавь SSL-конфиг для нового домена

Нужно добавить в nginx новые server-блоки с SSL для нового домена. Самый простой способ — скопировать существующий блок listen 443 и заменить в нём домен и пути сертификатов.

Или — если оба домена используют один сертификат (certbot может так сделать) — достаточно просто добавить server_name и сертификаты автоматически подхватятся.

Путь к новым сертификатам будет:
```
/etc/letsencrypt/live/НОВЫЙДОМЕН.com/fullchain.pem
/etc/letsencrypt/live/НОВЫЙДОМЕН.com/privkey.pem
```

### Шаг 8: Проверь и перезагрузи nginx

```bash
nginx -t
```

Если видишь `syntax is ok` и `test is successful`:

```bash
systemctl reload nginx
```

Если ошибка — внимательно прочитай сообщение, обычно указана строка с проблемой.

### Шаг 9: Проверь в браузере

- https://НОВЫЙДОМЕН.com — должна открыться Нави
- https://chat.НОВЫЙДОМЕН.com — должен открыться Нави Чат

---

<a id="сценарий-4"></a>
## СЦЕНАРИЙ 4: Сервер 64 умер — VPN на другой сервер

Если сервер 64 (64.188.118.135) полностью недоступен.

### Шаг 1: Купи новый VPS

Любой VPS в Европе/США с Linux (Ubuntu 22/24 или Debian 12).
Минимум: 1 CPU, 512MB RAM, 5GB диск.
Хостинги: Hetzner, Contabo, BuyVM, RackNerd.

### Шаг 2: Подключись к новому серверу

```bash
ssh root@НОВЫЙ_IP
```

### Шаг 3: Установи Xray

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### Шаг 4: Создай конфиг Xray

```bash
nano /usr/local/etc/xray/config.json
```

Вставь этот конфиг (это конфиг для CDN через Cloudflare — порт 80, без TLS):

```json
{
  "log": {
    "loglevel": "info"
  },
  "dns": {
    "servers": ["1.1.1.1", "8.8.8.8"],
    "queryStrategy": "UseIPv4"
  },
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "network": "tcp,udp",
        "outboundTag": "direct"
      }
    ]
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 80,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "0663b071-0e5f-40aa-9b2d-5654a5dbff21",
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
      "tag": "inbound-80-xhttp-cdn"
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

### Шаг 5: Открой порт 80

```bash
# Если используется ufw:
ufw allow 80/tcp

# Если используется iptables:
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```

### Шаг 6: Запусти Xray

```bash
systemctl enable xray
systemctl start xray
systemctl status xray
```

Убедись что статус **active (running)**.

### Шаг 7: Обнови DNS в Cloudflare

1. Открой https://dash.cloudflare.com
2. Выбери домен (cookglobe.ru или какой используешь)
3. **DNS** → **Records**
4. Найди запись `cdn` (тип A)
5. Нажми **Edit**
6. Замени IP на **НОВЫЙ_IP** сервера
7. Убедись что облачко **оранжевое** (Proxied)
8. **Save**

### Шаг 8: Подожди 1-2 минуты и проверь

VPN-ссылка осталась ТА ЖЕ САМАЯ (домен не менялся, изменился только IP за ним):

```
vless://0663b071-0e5f-40aa-9b2d-5654a5dbff21@cdn.cookglobe.ru:443?encryption=none&security=tls&type=splithttp&path=/&sni=cdn.cookglobe.ru&host=cdn.cookglobe.ru#CookGlobe-CDN-XHTTP
```

В v2rayNG ничего менять не нужно!

---

<a id="сценарий-5"></a>
## СЦЕНАРИЙ 5: Сервер 93 умер — приложения на другой сервер

Это самый сложный сценарий. Сервер 93 содержит:
- Нави (веб-приложение, ИИ-чат)
- Нави Чат (мессенджер)
- Telegram-бот
- PostgreSQL базы данных
- Redis

### Краткий план:

1. **Купи новый VPS** (минимум 2 CPU, 4GB RAM, 40GB диск, Ubuntu 22.04)

2. **Установи зависимости:**
```bash
apt update && apt install -y python3 python3-pip python3-venv nginx certbot python3-certbot-nginx postgresql redis-server git
```

3. **Клонируй репозитории с GitHub:**
```bash
cd /opt
git clone https://Donavinat:<GITHUB_TOKEN>@github.com/Donavinat/navi-web.git
git clone https://Donavinat:<GITHUB_TOKEN>@github.com/Donavinat/navi-chat.git
```

4. **Восстанови базу данных** (если есть бэкап):
```bash
# Бэкапы делаются скриптом /shared/supabase_auto_backup_new.py на сервере 150
# Последний бэкап: /shared/supabase_new_backup_20260228.dump
# Скачай с сервера 150 и восстанови:
scp root@150.241.107.204:/shared/supabase_new_backup_20260228.dump /tmp/
sudo -u postgres pg_restore -d navi_memory /tmp/supabase_new_backup_20260228.dump
```

5. **Настрой nginx** — скопируй конфиги из этого документа (см. раздел "Контакты серверов" для текущих конфигов).

6. **Получи SSL-сертификаты:**
```bash
certbot --nginx -d ДОМЕН -d chat.ДОМЕН
```

7. **Обнови DNS** — направь домены на новый IP.

8. **Запусти приложения** как systemd-сервисы.

Это задача для технического специалиста. Если нужна помощь — подключи Claude Code (сервер 150) или попроси помощь в Telegram-канале t.me/deployladeploy.

---

<a id="сценарий-6"></a>
## СЦЕНАРИЙ 6: Telegram-бот не работает

Бот @donavi_clawd_bot работает на сервере 93 и общается с API Telegram напрямую из Германии.

### Если Telegram заблокирован в России:
- Включи VPN (Сценарий 1) и пользуйся Telegram через VPN
- Бот сам находится в Германии — ему блокировки не мешают

### Если сервер 93 не отвечает:
Нужно развернуть бота на другом сервере:

1. На любом VPS:
```bash
apt install -y python3 python3-pip git
git clone https://Donavinat:<GITHUB_TOKEN>@github.com/Donavinat/claude-telegram.git
cd claude-telegram
pip3 install -r requirements.txt
```

2. Настрой переменные окружения (токены API) — они есть в `.env` файле на сервере 93
3. Запусти бота:
```bash
python3 bot.py
```

### Если бот не отвечает, но сервер работает:

Подключись к серверу 93 и проверь:

```bash
ssh root@93.185.157.128
# Пароль: yEwL5jvAu92N

# Проверь статус:
systemctl status claude-telegram

# Перезапусти:
systemctl restart claude-telegram

# Посмотри логи:
journalctl -u claude-telegram -n 50 --no-pager
```

---

<a id="что-не-нужно-делать"></a>
## ЧТО НЕ НУЖНО ДЕЛАТЬ (ВАЖНО!)

| Действие | Почему нельзя |
|----------|--------------|
| Ставить SSL режим **Full** на cookglobe.ru в CF | VPN перестанет работать! Должен быть **Flexible** |
| Менять порт 80 на Xray для CDN inbound | CF CDN ходит именно на порт 80 при режиме Flexible |
| Удалять DNS-записи donavi-cors.fun | Все приложения перестанут работать |
| Использовать WebSocket транспорт для VPN | DPI провайдера блокирует WebSocket VPN-трафик |
| Использовать CF Worker для XHTTP | Worker убивает стриминг, CDN работает лучше |
| Называть домены/субдомены vpn, proxy, tunnel | CF может заблокировать, провайдер может заметить |
| Включать **Bot Fight Mode** в CF для VPN-домена | Блокирует v2rayNG как бота |
| Включать **Under Attack Mode** в CF | Показывает капчу, VPN не сможет подключиться |
| Ставить SSL **Full (Strict)** для CDN VPN | Нет валидного сертификата на порту 80 |

---

<a id="все-рабочие-ссылки-vpn"></a>
## ВСЕ РАБОЧИЕ ССЫЛКИ VPN

### CDN через Cloudflare (основной, работает всегда)

```
vless://0663b071-0e5f-40aa-9b2d-5654a5dbff21@cdn.cookglobe.ru:443?encryption=none&security=tls&type=splithttp&path=/&sni=cdn.cookglobe.ru&host=cdn.cookglobe.ru#CookGlobe-CDN-XHTTP
```

### Через мост (81.90.31.187) — если мост работает

```
vless://8b6fe505-bbc4-4205-ab37-c89bd5ee4f2f@81.90.31.187:443?encryption=none&security=reality&sni=ozone.ru&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#МОСТ-Ozone
```

```
vless://cc24c507-5dcd-4586-a746-7a78752a5f9a@81.90.31.187:443?encryption=none&security=reality&sni=www.google.com&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#МОСТ-Google
```

```
vless://02047ceb-0a41-4101-a544-f637c69b3f6f@81.90.31.187:443?encryption=none&security=reality&sni=wildberries.ru&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#МОСТ-Wildberries
```

```
vless://68946aae-1e30-48fe-a7a6-8d736c2836d1@81.90.31.187:443?encryption=none&security=reality&sni=yandex.ru&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=tcp#МОСТ-Yandex
```

```
vless://ab502201-b8bd-489f-ad37-c9a6d8443518@81.90.31.187:443?encryption=none&security=reality&sni=www.kinopoisk.ru&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=tcp#МОСТ-Kinopoisk
```

### Напрямую на сервер 64 (Reality)

```
vless://bb7610d9-d042-4729-a329-d29a36e01c45@64.188.118.135:2053?encryption=none&security=reality&sni=ozone.ru&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=tcp#S3-Ozone
```

```
vless://c7ef1edd-8cb4-49db-9c27-15216e67e408@64.188.118.135:3443?encryption=none&security=reality&sni=www.google.com&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=tcp#S3-Google
```

```
vless://68946aae-1e30-48fe-a7a6-8d736c2836d1@64.188.118.135:4443?encryption=none&security=reality&sni=yandex.ru&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=tcp#S3-Yandex
```

```
vless://35937d84-32dc-4740-baa4-a8a4930f82d8@64.188.118.135:5443?encryption=none&security=reality&sni=wildberries.ru&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=tcp#S3-Wildberries
```

```
vless://ab502201-b8bd-489f-ad37-c9a6d8443518@64.188.118.135:6443?encryption=none&security=reality&sni=www.kinopoisk.ru&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=tcp#S3-Kinopoisk
```

### Напрямую на сервер 64 (XHTTP + Reality)

```
vless://5783a735-22f5-4658-af56-e4f9cb7f56ca@64.188.118.135:10443?encryption=none&security=reality&sni=www.google.com&fp=chrome&pbk=fb1Xfw3BA-mt2xbLLuoc9dxKH7zebn8RnyuNsTLqO2k&type=xhttp&path=/xhttp&host=www.google.com#S3-XHTTP-Google
```

### Напрямую на сервер 93 (Reality)

```
vless://8b6fe505-bbc4-4205-ab37-c89bd5ee4f2f@93.185.157.128:2053?encryption=none&security=reality&sni=ozone.ru&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#S2-Ozone
```

```
vless://cc24c507-5dcd-4586-a746-7a78752a5f9a@93.185.157.128:3443?encryption=none&security=reality&sni=www.google.com&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#S2-Google
```

```
vless://84bd110f-0e00-4627-8aea-a7f468ac0ae1@93.185.157.128:4443?encryption=none&security=reality&sni=yandex.ru&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#S2-Yandex
```

```
vless://02047ceb-0a41-4101-a544-f637c69b3f6f@93.185.157.128:5443?encryption=none&security=reality&sni=wildberries.ru&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#S2-Wildberries
```

```
vless://2bedd827-dd1f-48d4-b23e-ae4ba325e2fb@93.185.157.128:6443?encryption=none&security=reality&sni=www.kinopoisk.ru&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=tcp#S2-Kinopoisk
```

### Напрямую на сервер 93 (XHTTP + Reality)

```
vless://cef90384-ac9c-41e7-89f3-6abece57a428@93.185.157.128:10443?encryption=none&security=reality&sni=www.google.com&fp=chrome&pbk=TvqXahk6cTXZY0kT2BY2R0Q2auMra-hxG3-y44emRho&type=xhttp&path=/xhttp&host=www.google.com#S2-XHTTP-Google
```

### Hysteria2

```
hysteria2://FuYjVF6BpKQHMbHc+ECIgLPD@64.188.118.135:443?insecure=1#S3-Hysteria2
```

```
hysteria2://GQesKuh7t28jc+2NUn5BHFZu@93.185.157.128:443?insecure=1#S2-Hysteria2
```

---

<a id="контакты-серверов"></a>
## КОНТАКТЫ СЕРВЕРОВ

| Сервер | IP | Пароль SSH | Роль | Порты |
|--------|-----|-----------|------|-------|
| Сервер 93 | 93.185.157.128 | yEwL5jvAu92N | Нави, Нави Чат, Telegram-бот, Xray VPN | 80, 443, 2053, 3443, 4443, 5443, 6443, 10443 |
| Сервер 64 | 64.188.118.135 | yt44m0vOSvlQ | VPN (Xray, Hysteria2), CDN XHTTP | 80, 443, 2053, 3443, 4443, 5443, 6443, 8443, 8444, 8445, 10443 |
| Сервер 150 | 150.241.107.204 | aUs9CZ0824aJ | Claude Code, WS VPN | 443, 8443 |
| Мост | 81.90.31.187 | (через другие серверы) | SNI-маршрутизация, проксирование | 443 |

### Cloudflare

- Панель: https://dash.cloudflare.com
- Домены: donavi-cors.fun, cookglobe.ru, dona-vi.ru, dolgorukova-nat.workers.dev

### GitHub

- Аккаунт: **Donavinat**
- Токен: `<GITHUB_TOKEN>`
- Репозитории:
  - https://github.com/Donavinat/navi-web — Нави (веб)
  - https://github.com/Donavinat/navi-chat — Нави Чат (он же DonaviChat)
  - https://github.com/Donavinat/donavi-vpn — VPN
  - https://github.com/Donavinat/claude-telegram — Telegram-бот

### Приложения

- Нави (веб): https://donavi-cors.fun
- Нави Чат (веб): https://chat.donavi-cors.fun или https://m.donavi-cors.fun
- Нави (резерв): https://dona-vi.ru:8888
- Telegram-бот: @donavi_clawd_bot

---

<a id="быстрая-проверка"></a>
## БЫСТРАЯ ПРОВЕРКА ЧТО ВСЁ РАБОТАЕТ

### С телефона (без VPN, через мобильные данные):

1. Открой в браузере https://donavi-cors.fun — должна открыться Нави
2. Открой https://chat.donavi-cors.fun — должен открыться Нави Чат
3. Напиши @donavi_clawd_bot в Telegram — бот должен ответить

### С телефона (VPN через CDN):

1. Подключись к **CookGlobe-CDN-XHTTP** в v2rayNG
2. Открой https://google.com — должен открыться
3. Открой https://donavi-cors.fun — должна работать Нави

### С компьютера (SSH к серверам):

```bash
# Проверить сервер 93:
ssh root@93.185.157.128 "systemctl status nginx --no-pager; echo '---'; systemctl status navi-web --no-pager; echo '---'; systemctl status navi-chat --no-pager; echo '---'; systemctl status claude-telegram --no-pager"
# Пароль: yEwL5jvAu92N

# Проверить сервер 64:
ssh root@64.188.118.135 "systemctl status xray --no-pager; echo '---'; systemctl status hysteria-server --no-pager"
# Пароль: yt44m0vOSvlQ

# Проверить сервер 150:
ssh root@150.241.107.204 "systemctl status shadowtunnel-ws --no-pager; echo '---'; systemctl status nginx --no-pager"
# Пароль: aUs9CZ0824aJ
```

### Проверить CDN VPN (с компьютера):

```bash
# Должен ответить — значит CF CDN проксирует к серверу:
curl -v https://cdn.cookglobe.ru/ 2>&1 | head -20
```

### Проверить что Xray работает на сервере 64:

```bash
ssh root@64.188.118.135 "curl -s http://localhost:80/ ; echo; ss -tlnp | grep -E '(:80|:443|:2053|:8443)'"
# Пароль: yt44m0vOSvlQ
```

---

## ПОРЯДОК ДЕЙСТВИЙ ПРИ ПРОБЛЕМАХ (ШПАРГАЛКА)

| Что сломалось | Что делать |
|--------------|-----------|
| VPN через мост не работает | Переключись на **CookGlobe-CDN-XHTTP** в v2rayNG |
| VPN через CDN не работает | Проверь сервер 64 (SSH), проверь DNS в CF |
| Нави/Чат не открываются | Проверь сервер 93 (SSH), перезапусти nginx |
| Telegram-бот молчит | SSH на 93, `systemctl restart claude-telegram` |
| Домен заблокирован | Купи новый домен (Сценарий 2 или 3) |
| Сервер 64 умер | Новый VPS + Xray (Сценарий 4) |
| Сервер 93 умер | Новый VPS + деплой из GitHub (Сценарий 5) |
| Всё умерло | Новый VPS + Сценарий 4 + Сценарий 5 |

---

*Документ создан 13.03.2026. Хранить в надёжном месте. Если потеряется — копия на сервере 150 в /shared/*
