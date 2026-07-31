# 🚇 TrustTunnel — свой VPN-сервер за 15 минут (альтернатива VLESS / WireGuard / OpenVPN)

> Подробный гайд по поднятию своего VPN-сервера на базе протокола **TrustTunnel** (AdGuard) — устойчивой альтернативы VLESS, WireGuard и OpenVPN для обхода блокировок в России.
> Гайд проведёт вас от покупки домена до запуска рабочего туннеля через **RU-сервер → иностранный сервер**.
> **Дисклеймер:** данная инструкция не несёт ответственности за неработоспособность VPN. Особенно в условиях белых списков всё постоянно может меняться.

## 📋 Содержание

- [Что такое TrustTunnel?](#-что-такое-trusttunnel)
- [Что понадобится](#-что-понадобится)
- [Как это работает](#-как-это-работает)
- [Шаг 1. Покупка домена и серверов](#-шаг-1-покупка-домена-и-серверов)
- [Шаг 2. Привязка домена к RU-серверу](#-шаг-2-привязка-домена-к-ru-серверу)
- [Шаг 3. Привязка поддомена к иностранному серверу](#-шаг-3-привязка-поддомена-к-иностранному-серверу)
- [Шаг 4. Настройка RU-сервера (rinetd)](#-шаг-4-настройка-ru-сервера-rinetd)
- [Шаг 5. Настройка иностранного сервера (TrustTunnel)](#-шаг-5-настройка-иностранного-сервера-trusttunnel)
- [Шаг 6. Настройка клиента](#-шаг-6-настройка-клиента)
- [FAQ и типичные проблемы](#-faq-и-типичные-проблемы)
- [Связь и поддержка](#-связь-и-поддержка)
- [Лицензия](#-лицензия)

## 🧐 Что такое TrustTunnel?

**TrustTunnel** — это современный VPN-протокол с открытым исходным кодом, разработанный командой **AdGuard**. Он был создан как ответ на участившиеся блокировки популярных протоколов (WireGuard, OpenVPN, VLESS) и сочетает в себе высокую производительность и устойчивость к обнаружению.
Подробнее [тут](https://trusttunnel.org/index.html) и [тут](https://habr.com/ru/companies/femida_search/articles/988466/)

## 🧰 Что понадобится

| Компонент | Назначение |
|-----------|------------|
| 🌐 Домен | Точка входа, маскируется под обычный сайт |
| 🇷🇺 RU-сервер (VPS) | Проксирует трафик через rinetd |
| 🌍 Иностранный сервер (VPS) | Основной узел TrustTunnel |
| 💻 SSH-клиент | Для подключения к серверам |

## 🧭 Как это работает

```
[Клиент] → [RU-сервер :8080 (rinetd)] → [Иностранный сервер :8080 (TrustTunnel)] → [Интернет]
```

Клиент подключается к российскому IP, а RU-сервер прозрачно перекидывает трафик на иностранный узел.

## 🛒 Шаг 1. Покупка домена и серверов

### 1.1 Домен

Купить можно в любой зоне у любого регистратора. Можно купить на **Рег.ру** (скидка 5% по ссылке):
👉 [Рег.ру (скидка 5%)](https://www.reg.ru/domain/new/?rlink=reflink-31985921)

> ⚠️ **Не называйте домен** `myvpn.ru`, `freevpn.com`, `proxy123.net` и т.п.
> Он должен выглядеть как обычный сайт, например: `nailstudio.ru`, `coffee-blog.com`, `petrov-photo.ru`.

### 1.2 Российский сервер (VPS)

Подойдёт любой провайдер. Примеры:
- [FirstVDS](https://firstvds.ru/?from=1235565)
- [SprintBox](https://sprintbox.ru/s44536)

**Минимальные требования:** 1 CPU / 1 GB RAM / Ubuntu 22.04+.

### 1.3 Иностранный сервер (VPS)

Аналогично, любой провайдер. Примеры:
- [FirstVDS](https://firstvds.ru/?from=1235565)
- [SprintBox](https://sprintbox.ru/s44536)

📍 **Локация:** берём **Германию** или **Амстердам (Нидерланды)** — у них лучший баланс задержки из РФ и стабильности каналов до популярных сервисов.

**Минимальные требования:** 1 CPU / 1 GB RAM / Ubuntu 22.04+.

## 🔗 Шаг 2. Привязка домена к RU-серверу

В личном кабинете Рег.ру:

1. Заходим в **Мои домены** → выбираем домен
2. Переходим в **DNS-серверы / Управление зоной**
3. Добавляем **A-запись**:

| Поле | Значение |
|------|----------|
| Имя | `@` (корень домена) |
| Тип | `A` |
| Значение | IP вашего **российского** сервера |

4. Сохраняем и ждём 5–30 минут.

### Проверка

```bash
# Windows (cmd / PowerShell)
ping ваш-домен.ru

# macOS / Linux
dig ваш-домен.ru +short
```

Должен ответить IP российского сервера. Если не резолвится — DNS ещё не обновился, подождите 15–30 минут.

## 🔗 Шаг 3. Привязка поддомена к иностранному серверу

Нужна отдельная DNS-запись для иностранного узла (на неё будем выпускать SSL-сертификат).

1. Заходим в **Мои домены** → выбираем домен
2. Переходим в **DNS-серверы / Управление зоной**
3. Добавляем **A-запись**:

| Поле | Значение |
|------|----------|
| Имя | `another` (или `admin`, `connect` — любое слово) |
| Тип | `A` |
| Значение | IP вашего **иностранного** сервера |

4. Сохраняем и ждём 5–30 минут.

### Проверка

```bash
# Windows
ping another.ваш-домен.ru

# macOS / Linux
dig another.ваш-домен.ru +short
```

Должен ответить IP иностранного сервера.

---

## 🇷🇺 Шаг 4. Настройка RU-сервера (rinetd)

### 4.1 Подключение

```bash
ssh root@<IP-РУ-сервера>
```

Пароль выдаёт провайдер после покупки.

### 4.2 Обновление системы

```bash
apt update && apt upgrade -y
apt install -y curl ufw
```

### 4.3 Установка Docker

Скопируйте и выполните целиком:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
                    docker-buildx-plugin docker-compose-plugin
```

Проверка:

```bash
docker --version
```

Если версия отобразилась — установка прошла успешно ✅

### 4.4 Настройка файрвола

```bash
ufw allow ssh
ufw allow 8080/tcp    # порт для клиентских подключений (можно любой)
ufw enable
```

### 4.5 Конфиг rinetd

Создайте файл `rinetd.conf`:

```txt
0.0.0.0   8080/tcp   <ip иностранного сервера или another.ваш-домен.ru>   8080/tcp
0.0.0.0   8080/udp   <ip иностранного сервера или another.ваш-домен.ru>   8080/udp
```

> Подставьте IP иностранного сервера либо его домен `another.ваш-домен.ru`. Домен удобнее: при смене IP не придётся править конфиг.

### 4.6 docker-compose файл

Создайте `rinetd.yml`:

```yaml
services:
  rinetd:
    image: alga777/rinetd:0.73
    container_name: rinetd
    network_mode: host
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
    volumes:
      - ./rinetd.conf:/etc/rinetd/rinetd.conf:ro
    restart: always
```

### 4.7 Запуск

```bash
docker compose -f rinetd.yml up -d
```

Проверка:

```bash
docker logs rinetd
```

Должно быть:

```
rinetd: starting redirections...
rinetd: starting redirections...
```

---

## 🌍 Шаг 5. Настройка иностранного сервера (TrustTunnel)

### 5.1 Подключение

```bash
ssh root@<IP-иностранного-сервера>
```

### 5.2 Обновление системы

```bash
apt update && apt upgrade -y
apt install -y curl ufw
```

### 5.3 Установка Docker

Скопируйте и выполните целиком (аналогично RU-серверу):

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
                    docker-buildx-plugin docker-compose-plugin
```

Проверка:

```bash
docker --version
```

### 5.4 Настройка файрвола

```bash
ufw allow ssh
ufw allow 8080/tcp    # ⚠️ ВАЖНО: тот же порт, что и на RU-сервере
ufw allow 80/tcp      # нужно для выпуска SSL-сертификата
ufw enable
```

### 5.5 Выпуск SSL-сертификата (Let's Encrypt)

Создайте папку для данных:

```bash
mkdir -p ~/trusttunnel-data
```

Выпустите сертификат:

```bash
docker run --rm -it \
  -v ~/trusttunnel-data/cert:/etc/letsencrypt \
  -v ~/trusttunnel-data/cert-webroot:/var/www/certbot \
  -p 80:80 \
  certbot/certbot:v2.8.0 certonly \
  --standalone \
  -d <another.ваш-домен.ru> \
  --email <ваш-реальный-email> \
  --agree-tos \
  --no-eff-email
```

> Замените:
> - `<another.ваш-домен.ru>` — DNS-запись иностранного сервера из [Шага 3](#-шаг-3-привязка-поддомена-к-иностранному-серверу)
> - `<ваш-реальный-email>` — работающий email (на него придут уведомления при истечении сертификата)

При успехе увидите сообщение вида:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/another.ваш-домен.ru/fullchain.pem
```

#### Автопродление сертификата

Сертификат Let's Encrypt действителен **90 дней**. Без продления через 3 месяца VPN перестанет работать. Настройте `cron`:

```bash
crontab -e
```

Добавьте строку (замените путь на ваш полный путь к `trusttunnel-data`):

```
0 3 * * * /usr/bin/docker run --rm -v /root/trusttunnel-data/cert:/etc/letsencrypt -p 80:80 certbot/certbot:v2.8.0 renew --quiet && /usr/bin/docker restart trusttunnel
```

Проверка без реального выпуска:
```bash
docker run --rm -v ~/trusttunnel-data/cert:/etc/letsencrypt -p 80:80 certbot/certbot:v2.8.0 renew --dry-run
```

### 5.6 Настройка TrustTunnel через wizard

Мы будем использовать образ:
📦 [Docker Hub: ruslanakhatv/trusttunnel-server](https://hub.docker.com/r/ruslanakhatv/trusttunnel-server)

#### Запуск wizard

```bash
docker run --rm -it \
  -v ~/trusttunnel-data:/etc/trusttunnel \
  --entrypoint /usr/local/bin/setup_wizard \
  ruslanakhatv/trusttunnel-server:latest
```

Разбор флагов:
- `--rm` — контейнер удалится после выхода (нам от него нужны только сгенерированные `.toml` файлы)
- `-it` — интерактивный режим, wizard будет задавать вопросы
- `-v ~/trusttunnel-data:/etc/trusttunnel` — папка с хоста пробрасывается внутрь; wizard положит туда все конфиги, и они останутся на сервере после удаления контейнера
- `--entrypoint /usr/local/bin/setup_wizard` — запускаем мастер настройки, а не сервер

> ⚠️ Wizard создаёт файлы в **текущей рабочей директории контейнера** — это и есть `/etc/trusttunnel/`. Поэтому все относительные пути (`credentials.toml`, `rules.toml`, `vpn.toml`, `hosts.toml`) сохранятся в `~/trusttunnel-data/` на хосте.

#### Пошаговый диалог с wizard-ом

Вот реальный сценарий диалога и что вводить:

```
Welcome to the setup wizard
Let's build the library settings
```

| № | Вопрос | Что вводить | Комментарий |
|---|--------|-------------|-------------|
| 1 | `The address to listen on` | `0.0.0.0:8080` | Тот же порт, что открыт в ufw и проброшен через rinetd |
| 2 | `Path to the credentials file` | `credentials.toml` | Оставляем по умолчанию |
| 3 | `Username` | `testuser` (или любое) | Логин клиента |
| 4 | `Password` | `••••••••` | Придумайте надёжный пароль |
| 5 | `Add one more user?` | `no` | Пользователей еще добавим позже, добавьте пока одного |
| 6 | `Path to the rules file` | `rules.toml` | По умолчанию |
| 7 | `Do you want to configure connection filtering rules?` | `no` | Разрешить все соединения |
| 8 | `Path to a file to store the library settings` | `vpn.toml` | По умолчанию |
| 9 | `How would you like to create a certificate?` | `Provide path to existing certificate` | Мы уже выпустили сертификат certbot-ом |
| 10 | `Path to certificate file(s)` | см. ниже 👇 | Два пути через пробел |
| 11 | `Do you want to configure alternative SNIs?` | `no` | Для базового сетапа не нужно |
| 12 | `Path to a file to store the TLS hosts settings` | `hosts.toml` | По умолчанию |

**Для пункта 10 (пути к сертификатам) — вводим одной строкой через пробел:**

```
/etc/trusttunnel/cert/live/another.ваш-домен.ru/fullchain.pem /etc/trusttunnel/cert/live/another.ваш-домен.ru/privkey.pem
```

> Замените `another.ваш-домен.ru` на ваш реальный поддомен, на который выпускали сертификат в шаге 5.5.
> Пути начинаются с `/etc/trusttunnel/`, потому что мы пробросили `~/trusttunnel-data → /etc/trusttunnel`.

#### После завершения

Wizard выведет итог:

```
═══════════════════════════════════════════════════════════════
                    Setup Complete!
═══════════════════════════════════════════════════════════════

Configuration files created:
  • vpn.toml          - Main endpoint settings
  • hosts.toml        - TLS host configuration
  • credentials.toml  - User credentials
  • .../fullchain.pem - TLS certificate
  • .../privkey.pem   - TLS private key
```

Проверяем, что файлы действительно на хосте:

```bash
ls -la ~/trusttunnel-data/
```

Должны быть: `vpn.toml`, `hosts.toml`, `credentials.toml`, `rules.toml` и папка `cert/`.

---

### 5.7 Запуск сервера

#### Запуск контейнера

```bash
docker run -d \
  --name trusttunnel \
  --restart unless-stopped \
  -p 8080:8080/tcp \
  -p 8080:8080/udp \
  -v ~/trusttunnel-data:/etc/trusttunnel \
  ruslanakhatv/trusttunnel-server:latest
```

Разбор флагов:
- `-d` — запуск в фоне
- `--name trusttunnel` — имя контейнера (по нему будем управлять)
- `--restart unless-stopped` — автоподъём при ребуте (пока сами не остановим)
- `-p 8080:8080/tcp` и `-p 8080:8080/udp` — пробрасываем порт наружу и по TCP, и по UDP (UDP нужен для QUIC)
- `-v ~/trusttunnel-data:/etc/trusttunnel` — та же папка с конфигами и сертификатами, что использовал wizard

> ⚠️ Порт `8080` должен совпадать с тем, что вы указали в wizard-е, открыли в `ufw` и прописали в `rinetd.conf` на RU-сервере.

#### Проверка

```bash
docker ps | grep trusttunnel
docker logs -f trusttunnel
```

Логи должны показать, что сервер запустился и слушает `0.0.0.0:8080`. Убедиться, что порт занят:

```bash
ss -tulpn | grep 8080
```

#### Автозапуск

Флаг `--restart unless-stopped` уже обеспечивает автостарт после ребута. Дополнительно убедитесь, что сам Docker в автозагрузке:

```bash
systemctl enable docker
```

#### 👥 Добавление нового пользователя

Пользователи хранятся в файле `~/trusttunnel-data/credentials.toml`. Смотрим текущее содержимое:

```bash
cat ~/trusttunnel-data/credentials.toml
```

```toml
[[client]]
username = "testuser"
password = "testpassword"
```

Открываем файл в редакторе:

```bash
nano ~/trusttunnel-data/credentials.toml
```

И **дописываем ещё одного клиента блоком `[[client]]`**:

```toml
[[client]]
username = "testuser"
password = "testpassword"

[[client]]
username = "testuser2"
password = "testpassword2"
```

Сохраняем (`Ctrl+O`, `Enter`, `Ctrl+X`) и перезапускаем сервер, чтобы он подхватил изменения:

```bash
docker restart trusttunnel
```

Готово — теперь оба пользователя могут подключаться.

> 💡 Хорошая практика: у каждого клиента свой **уникальный** пароль. Сгенерировать надёжный можно так:
> ```bash
> openssl rand -base64 24
> ```

## 💻 Шаг 6. Настройка клиента

### 6.1 Установка приложения TrustTunnel

| Платформа | Как поставить |
|-----------|---------------|
| 🪟 **Windows** | Скачать [TrustTunnel](https://github.com/TrustTunnel/TrustTunnel/blob/master/README.md#windows) по инструкции |
| 🐧 **Linux** | Скачать [TrustTunnel](https://github.com/TrustTunnel/TrustTunnel/blob/master/README.md#linux--macos-1) по инструкции |
| 🤖 **Android** | Скачать с [Google Play](https://play.google.com/store/apps/details?id=com.adguard.trusttunnel&pli=1) |
| 🍎 **iOS** | [App Store](https://apps.apple.com/ca/app/trusttunnel/id6755807890) — **приложение заблокировано в РФ** |
| 🍏 **macOS** | App Store — **приложение заблокировано в РФ**, либо CLI версия [ссылка](https://github.com/TrustTunnel/TrustTunnel/blob/master/README.md#linux--macos-1) |

#### 🍎 Смена региона для iOS / macOS

Приложение недоступно в российском App Store, поэтому нужно временно сменить регион аккаунта Apple ID.

📖 Пошаговая инструкция по смене региона:
👉 [change-appstore-country.com](https://change-appstore-country.com/ru)

После смены региона (например, на Казахстан или Турцию):
1. Откройте App Store
2. Найдите **TrustTunnel**
3. Установите приложение
4. Регион аккаунта после установки можно вернуть обратно — приложение продолжит работать

#### Использование CLI версии

> Различайте машины: **сервер** (иностранный VPS из Шага 5) и **клиент** (компьютер, на котором пользуетесь VPN). Команды ниже выполняются на разных машинах — это указано в каждом шаге.

1. **На сервере.** Получите deep-link URI для пользователя:
```bash
docker exec trusttunnel trusttunnel_endpoint   /etc/trusttunnel/vpn.toml   /etc/trusttunnel/hosts.toml   -c testuser  -a <another.ваш-домен.ru>:8080
```
Скопируйте полученный deep-link URI (вида `tt://...`).

2. **На клиенте.** Скачайте бинарные файлы `trusttunnel_client` и `setup_wizard` по [инструкции проекта](https://github.com/TrustTunnel/TrustTunnel/blob/master/README.md#linux--macos-1) и сделайте их исполняемыми:
```bash
chmod +x trusttunnel_client setup_wizard
```

3. **На клиенте.** Запустите интерактивный мастер настройки:
```bash
sudo ./setup_wizard
```

4. Последовательно пройдите шаги конфигурации, отвечая на подсказки мастера:
   - **Existing settings**: Если вы настраиваете клиент впервые, выберите `Make new from scratch`.
   - **VPN mode**: Выберите политику маршрутизации трафика:
     - `general`: весь трафик идет через VPN (кроме адресов из списка исключений).
     - `selective`: через VPN идет только трафик, адреса которого указаны в списке исключений.
   - **Endpoint configuration**: Выберите способ подключения — `Deep-link URI (tt://...)`.
   - **Paste deep-link URI**: Вставьте полученную deep-link ссылку (вида `tt://...`). Мастер автоматически извлечет из неё адрес сервера, логин, пароль, протокол и другие параметры.
   - **Accept this configuration?**: Проверьте выведенные на экран параметры (Hostname, Username, Protocol и т.д.) и подтвердите их (`yes`).
   - **Listener type**: Выберите тип локального слушателя:
     - `socks`: SOCKS5 прокси с поддержкой UDP.
     - `tun`: TUN-устройство (создает полноценный сетевой туннель на уровне ОС). **Выберите `tun`**.
   - **Interface name** *(опционально для `tun`)*: Укажите имя сетевого интерфейса или оставьте поле пустым для автоматического определения.
   - **Allow changing system DNS servers**: Разрешите (`yes`) или запретите (`no`) клиенту изменять системные DNS-серверы. **Выберите `yes`**.
   - **Path to a file to store the settings**: Укажите путь и имя файла, в который нужно сохранить конфигурацию (например, `trusttunnel_client.toml`).

5. После успешного завершения работы мастера запустите клиент, указав путь к созданному конфигурационному файлу:
```bash
sudo ./trusttunnel_client -c trusttunnel_client.toml
```

> **Дополнительные настройки:**
> Для более тонкой конфигурации (добавление исключений, настройка DNS, включение kill switch) вы можете вручную отредактировать созданный `.toml` файл в любом текстовом редакторе.
> Полный список доступных опций можно посмотреть, выполнив команду `./trusttunnel_client -h`.

### 6.2 Добавление сервера в приложении

1. Откройте приложение **TrustTunnel**.
2. Нажмите **«Add server»** (или иконку **«+»** в правом верхнем углу).
3. Заполните поля:

| Поле | Что вводить |
|------|-------------|
| **Server name** | Любое название на ваш вкус (например, `My VPN` или `Netherlands`) |
| **Address** | IP-адрес или домен вашего **RU-сервера** (например, `ваш-домен.ru` или прямой IP) |
| **Domain name** | Домен из сертификата — `another.ваш-домен.ru` (тот, что указывали в wizard-е) |
| **Username** | Логин из `credentials.toml` (например, `testuser`) |
| **Password** | Пароль из `credentials.toml` |
| **Protocol** | `http/2` **или** `quic` (рекомендуется QUIC — обычно быстрее) |
| **Allow IPv6 connections via the server** | ❌ **Отключено** |

> 💡 Если получаете данные через Telegram-бота — просто **тапайте по значениям в сообщении**, они автоматически скопируются в буфер, и их останется только вставить в поля.

4. Нажмите **«Add»**.

### 6.3 Подключение

1. В списке серверов выберите только что добавленный.
2. Нажмите кнопку подключения (обычно большая круглая кнопка в центре или переключатель).
3. На **iOS/macOS** система запросит разрешение на установку **VPN-профиля** — согласитесь и введите пароль от устройства.
4. Когда индикатор станет зелёным / активным — соединение установлено.

## ❓ FAQ и типичные проблемы

### Не резолвится домен после настройки DNS
DNS-записи могут обновляться до 24 часов. Проверьте:
```bash
dig +short another.ваш-домен.ru
nslookup another.ваш-домен.ru 8.8.8.8
```
Если через публичный DNS (`8.8.8.8` / `1.1.1.1`) IP виден, а у вас нет — почистите локальный кеш DNS или подождите.

### `rinetd` не стартует: `address already in use`
Порт 8080 занят другим процессом. Найдите и остановите его:
```bash
ss -tlnp | grep :8080
docker ps | grep 8080
```
Либо смените порт в `rinetd.conf` и в `rinetd.yml` (с обеих сторон — RU и иностранный сервер).

### Сертификат Let's Encrypt не выпускается
Частые причины:
- **Порт 80 закрыт файрволом** — откройте на время выпуска: `ufw allow 80/tcp`.
- **DNS-запись ещё не обновилась** — подождите или проверьте через `dig`.
- **Порт 80 занят другим процессом** — `certbot --standalone` требует свободный 80-й порт. Найдите, кто его держит: `ss -tlnp | grep :80`, и остановите этот процесс перед выпуском.

### Подключение есть, но трафик не идёт
Проверьте **IPv6**. На мобильных операторах и некоторых провайдерах IPv6 ломает маршрутизацию TrustTunnel:
- В приложении: **Allow IPv6 Connections via the server** → ❌ отключено.
- В CLI `.toml`: убедитесь, что нет `ipv6` маршрутов, либо отключите IPv6 на машине.

### QUIC не работает — откат на http/2
QUIC чувствителен к блокировкам и потере пакетов. Если соединение рвётся или не поднимается:
- В приложении: **Protocol** → `http/2` вместо `quic`.
- В CLI `.toml`: поменяйте протокол на `http/2`.

### Как проверить, что VPN реально работает
После подключения:
```bash
curl https://ifconfig.me
```
Должен вернуться IP иностранного сервера, а не ваш обычный.

## 💬 Связь и поддержка

👉 Telegram: [@miamistack](https://t.me/miamistack)

## 📜 Лицензия

MIT — используйте на своё усмотрение.

⭐ **Если гайд помог — поставьте звезду репозиторию!**

<!-- keywords: vpn, trusttunnel, adguard, свой vpn сервер, обход блокировок, vless альтернатива, wireguard, openvpn, vpn сервер, vpn для россии, проброс портов -->
