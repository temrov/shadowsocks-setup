# Мой Shadowsocks VPN: установка и настройка

Инструкция описывает конфигурацию из статьи [Set Up Shadowsocks Proxy Service on Linux Server and Mac Client](https://sequoia18.medium.com/set-up-shadowsocks-server-linux-and-client-mac-a72ce2aeabd3), с учётом моих изменений для macOS.

> **Моя схема**
>
> - Сервер: AWS Lightsail, Ubuntu, статический IPv4, регион Stockholm.
> - Серверная реализация: `shadowsocks-libev` (`ss-server`).
> - Обфускация: `v2ray-plugin`, WebSocket.
> - Клиент: Mac с Apple Silicon, `shadowsocks-libev` из Homebrew.
> - Локальный SOCKS5: `127.0.0.1:1080`.
> - Конфигурация клиента: `~/.config/ss-client/config.json`.
> - `proxybridge` запускается отдельно от `ss-local` через собственный LaunchAgent.
> - SOCKS5 задаётся в системных настройках macOS, а не в браузерном расширении.

В примерах ниже `<SERVER_IP>` и `<PASSWORD>` — заполнители. Подставляй свои значения. Не публикуй конфигурацию с паролем.

---

## 1. Сервер AWS Lightsail (Ubuntu)

### 1.1. Создать или подготовить инстанс

1. Создать Linux/Unix instance в AWS Lightsail с Ubuntu.
2. Назначить инстансу **Static IPv4**, чтобы адрес не менялся после перезапуска.
3. Создать отдельного пользователя с правами `sudo` и настроить вход по SSH-ключу (как в исходной статье).

Подключение:

```bash
ssh <USERNAME>@<SERVER_IP>
```

### 1.2. Установить Shadowsocks-libev

```bash
sudo apt-get update
sudo apt install shadowsocks-libev
```

### 1.3. Установить серверный v2ray-plugin

На сервере должен быть установлен `v2ray-plugin`, совместимый с архитектурой Ubuntu-инстанса.

В исходной статье приведён пример для Linux x86 (386) и версии 1.3.2:

```bash
wget https://github.com/shadowsocks/v2ray-plugin/releases/download/v1.3.2/v2ray-plugin-linux-386-v1.3.2.tar.gz
tar xvf v2ray-plugin-linux-386-v1.3.2.tar.gz
sudo mv v2ray-plugin_linux_386 /usr/bin/v2ray-plugin
sudo setcap cap_net_bind_service+ep /usr/bin/v2ray-plugin
```

**Важно:** имя `linux-386` в команде означает 32-битную x86-сборку. Перед повторной установкой проверь архитектуру сервера:

```bash
uname -m
```

Для `x86_64` нужна соответствующая Linux amd64-сборка плагина, а для ARM — сборка соответствующей ARM-архитектуры. Не устанавливай `linux-386` вслепую на сервер другой архитектуры. Если плагин уже установлен и сервер работает, повторно устанавливать его не требуется.

### 1.4. Настроить сервер

Открыть конфигурацию:

```bash
sudo nano /etc/shadowsocks-libev/config.json
```

Конфигурация с обфускацией по схеме статьи:

```json
{
    "server": "0.0.0.0",
    "mode": "tcp_and_udp",
    "server_port": 80,
    "password": "<PASSWORD>",
    "timeout": 60,
    "method": "aes-256-gcm",
    "fast_open": true,
    "plugin": "v2ray-plugin",
    "plugin_opts": "server"
}
```

`<PASSWORD>` должен совпадать с паролем в конфигурации Mac.

### 1.5. Открыть порт в Lightsail

В AWS Lightsail открой **Networking → IPv4 Firewall** и добавь входящее правило:

- Protocol: TCP
- Port: `80`

В этой схеме клиент подключается к серверу через `v2ray-plugin` по WebSocket поверх TCP.

### 1.6. Перезапустить и проверить сервер

```bash
sudo systemctl restart shadowsocks-libev
sudo systemctl status shadowsocks-libev --no-pager
```

Если служба не запустилась, посмотреть журнал:

```bash
sudo journalctl -u shadowsocks-libev -n 100 --no-pager
```

Проверить, что сервер слушает порт:

```bash
sudo ss -lntp | grep ':80'
```

### 1.7. Проверить энтропию и при необходимости установить rng-tools

В исходной статье этот шаг отмечен как optional. `rng-tools` помогает Linux получать качественные случайные числа, когда системе не хватает энтропии. Это относится к серверу Ubuntu и не требует установки чего-либо на Mac.

Сначала проверь журнал Shadowsocks на предупреждение о недостатке энтропии:

```bash
sudo journalctl -u shadowsocks-libev -n 100 --no-pager
```

Ищи сообщение примерно такого вида:

```text
This system doesn't provide enough entropy to quickly generate high-quality random numbers.
Installing the rng-utils/rng-tools, jitterentropy or haveged packages may help.
```

**Если такого предупреждения нет, устанавливать `rng-tools` специально не требуется.** Если предупреждение есть, установи пакет:

```bash
sudo apt update
sudo apt install rng-tools
```

Проверь состояние службы:

```bash
sudo systemctl status rng-tools --no-pager
```

Затем перезапусти Shadowsocks и проверь журнал ещё раз:

```bash
sudo systemctl restart shadowsocks-libev
sudo journalctl -u shadowsocks-libev -n 50 --no-pager
```

Предупреждение о недостатке энтропии должно исчезнуть. Если в конкретной версии Ubuntu служба называется иначе или пакет недоступен, ориентируйся на фактическое имя пакета и службы в этой системе.

---

## 2. Клиент macOS

### 2.1. Установить shadowsocks-libev

На Mac с Apple Silicon:

```bash
brew install shadowsocks-libev
```

Путь к исполняемому файлу в этой конфигурации:

```text
/opt/homebrew/bin/ss-local
```

### 2.2. Установить v2ray-plugin вручную

В моём случае установка через Homebrew не подошла. Поэтому плагин устанавливается напрямую из GitHub Releases — именно ARM64-сборка для macOS и версия `1.5.0`.

Скачать архив:

```bash
curl -L \
  -o ~/Downloads/v2ray-plugin-darwin-arm64-v1.5.0.tar.gz \
  https://github.com/shadowsocks/v2ray-plugin/releases/download/v1.5.0/v2ray-plugin-darwin-arm64-v1.5.0.tar.gz
```

Распаковать:

```bash
tar -xzf ~/Downloads/v2ray-plugin-darwin-arm64-v1.5.0.tar.gz \
  -C ~/Downloads
```

Скопировать бинарный файл в `/usr/local/bin`:

```bash
sudo cp ~/Downloads/v2ray-plugin_darwin_arm64 /usr/local/bin/v2ray-plugin
```

Проверить наличие файла:

```bash
ls -l /usr/local/bin/v2ray-plugin
```

#### Разрешить запуск в macOS

Поскольку скачанный бинарный файл не подписан Apple-разработчиком, macOS может заблокировать его первый запуск.

1. Запусти плагин из Terminal:

   ```bash
   /usr/local/bin/v2ray-plugin
   ```

   Если macOS заблокирует запуск, это ожидаемо для неподписанной утилиты.

2. Открой **System Settings → Privacy & Security**.
3. В нижней части раздела безопасности найди сообщение о блокировке `v2ray-plugin` и нажми **Open Anyway / Всё равно открыть** (название может отличаться в зависимости от версии macOS).
4. Подтверди разрешение и повтори запуск из Terminal, если потребуется.

Не нужно отключать Gatekeeper целиком.

### 2.3. Создать конфигурацию ss-local

Создать каталог и файл:

```bash
mkdir -p ~/.config/ss-client
nano ~/.config/ss-client/config.json
```

Содержимое:

```json
{
    "server": "<SERVER_IP>",
    "server_port": 80,
    "password": "<PASSWORD>",
    "timeout": 60,
    "method": "aes-256-gcm",
    "fast_open": true,
    "local_address": "127.0.0.1",
    "local_port": 1080,
    "plugin": "v2ray-plugin"
}
```

Здесь:
- `server` — статический IPv4 Lightsail-инстанса;
- `server_port` — `80`, поскольку используется серверная конфигурация с `v2ray-plugin`;
- `password` и `method` должны совпадать с сервером;
- `local_address` и `local_port` задают локальный SOCKS5 `127.0.0.1:1080`;
- `plugin` указывает на установленный `v2ray-plugin`.

Ограничить доступ к конфигурации, так как в ней хранится пароль:

```bash
chmod 600 ~/.config/ss-client/config.json
```

### 2.4. Запустить локальный Shadowsocks-клиент

```bash
/opt/homebrew/bin/ss-local \
  -c /Users/<USERNAME>/.config/ss-client/config.json
```

`ss-local` должен оставаться запущенным: он предоставляет локальный SOCKS5 на `127.0.0.1:1080`.

Проверить, что порт слушается:

```bash
lsof -nP -iTCP:1080 -sTCP:LISTEN
```

Проверить прокси запросом:

```bash
curl --socks5-hostname 127.0.0.1:1080 https://1.1.1.1/cdn-cgi/trace
```

Если запрос проходит, в ответе Cloudflare будет текст с параметрами соединения. Опция `--socks5-hostname` передаёт разрешение DNS через SOCKS5.

---

## 3. Промежуточная проверка через системный SOCKS5 macOS

Этот этап нужен **только для проверки работоспособности Shadowsocks до установки и настройки ProxyBridge**. Так можно отдельно убедиться, что сервер, `ss-local` и локальный SOCKS5 работают, не добавляя пока ещё один компонент в цепочку.

Я не использую браузерное расширение SwitchyOmega. Для временной проверки SOCKS5 можно включить непосредственно в системных настройках macOS:

1. Открой **System Settings**.
2. Перейди в **Wi-Fi**.
3. Нажми **Details** у подключённой сети.
4. Открой **Proxies**.
5. Включи **SOCKS Proxy**.
6. Укажи:
   - SOCKS Proxy Server: `127.0.0.1`
   - Port: `1080`
7. Сохрани настройки.

Убедись, что `ss-local` запущен, затем открой сайты или приложения и проверь, что соединение работает через VPN. Дополнительно можно проверить сам SOCKS5 из Terminal:

```bash
curl --socks5-hostname 127.0.0.1:1080 https://1.1.1.1/cdn-cgi/trace
```

Если проверка успешна, базовая цепочка Shadowsocks работает: Mac → локальный SOCKS5 (`ss-local`) → сервер Shadowsocks.

**После установки и настройки ProxyBridge отключи SOCKS Proxy в системных настройках macOS.** Дальше перенаправлением трафика будет заниматься ProxyBridge, а не глобальная настройка SOCKS5 macOS.

---

## 4. Установка и автозапуск ProxyBridge

### Зачем нужен ProxyBridge

`ss-local` сам по себе предоставляет локальный SOCKS5-прокси. Если указать этот SOCKS5 в системных настройках macOS, приложения, учитывающие системный прокси, будут использовать его глобально. Это удобно для первичной проверки, но не даёт гибко выбирать, какой трафик отправлять через VPN.

**ProxyBridge решает задачу раздельной маршрутизации (split tunneling).** Он подключается к локальному SOCKS5 `127.0.0.1:1080` как к upstream-прокси и применяет собственные правила. Например, можно:

- направлять через Shadowsocks только выбранные приложения или адреса;
- оставлять остальные приложения и назначения с прямым подключением (`DIRECT`);
- при необходимости блокировать трафик, соответствующий отдельным правилам (`BLOCK`).

Таким образом, не требуется включать SOCKS5 в системных настройках macOS для всего трафика. Системный SOCKS5 из предыдущего раздела — временный диагностический этап; в рабочей схеме маршрутизацией управляет ProxyBridge.

ProxyBridge устанавливается как macOS-приложение с системным расширением Network Extension. В моей конфигурации дополнительно используется LaunchAgent, который запускает CLI-команду `proxybridge` с профилем `Default`.

### 4.1. Установить ProxyBridge

Использовать официальную инструкцию проекта:

[ProxyBridge for macOS — Installation](https://github.com/InterceptSuite/ProxyBridge/blob/master/MacOS/README.md#installation)

Порядок установки из README проекта:

1. Открыть [страницу Releases](https://github.com/InterceptSuite/ProxyBridge/releases) и скачать последний релиз.
2. Открыть скачанный файл `.pkg`.
3. Пройти шаги установщика.
4. Разрешить System Extension, когда macOS запросит разрешение.
5. Запустить ProxyBridge из папки **Applications**.

При первом запуске также проверь разрешение расширения:

**System Settings → General → Login Items & Extensions → Network Extension**

Разреши ProxyBridge в этом разделе, если macOS запросит подтверждение.

### 4.2. Настроить SOCKS5 в ProxyBridge

В меню приложения открой:

**Menu Bar → Proxy → Proxy Settings**

Выбери тип прокси **SOCKS5** и укажи локальный SOCKS5, который предоставляет `ss-local`:

- Host: `127.0.0.1`
- Port: `1080`
- Username / Password: не заполнять, если локальный SOCKS5 не требует аутентификации.

Правила маршрутизации настраиваются отдельно:

**Menu Bar → Proxy → Proxy Rules**

В ProxyBridge можно направлять выбранные приложения или назначения через прокси, пропускать напрямую (`DIRECT`) либо блокировать (`BLOCK`). Правила оцениваются по порядку. Именно здесь задаётся, какой трафик пойдёт через Shadowsocks, а какой останется прямым.

### 4.3. LaunchAgent для Shadowsocks

Файл:

```text
~/Library/LaunchAgents/com.local.shadowsocks.plist
```

Мой plist запускает shell-скрипт, а не `ss-local` напрямую. Содержимое:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.local.shadowsocks</string>

    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/ss-local</string>
        <string>-c</string>
        <string>/Users/USERNAME/.config/ss-local/config.json</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>LimitLoadToSessionType</key>
    <string>Aqua</string>

    <key>ProcessType</key>
    <string>Interactive</string>

    <key>StandardOutPath</key>
    <string>/tmp/shadowsocks_output.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/shadowsocks_error.log</string>
</dict>
</plist>
```

Перед установкой замени `<USERNAME>` на короткое имя своего пользователя macOS. Например, если домашняя папка пользователя `/Users/alex`, путь должен быть `/Users/alex/launch_localshadowsocks.sh`.

Сам скрипт должен находиться по этому пути и быть исполняемым:

```bash
chmod +x ~/launch_localshadowsocks.sh
```

### 4.4. LaunchAgent для ProxyBridge

Файл:

```text
~/Library/LaunchAgents/com.local.proxybridge.plist
```

Это содержимое моего загруженного plist — параметры запуска, профиль, задержка и отдельные логи сохранены:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.local.proxybridge</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>sleep 15; exec /usr/local/bin/proxybridge --profile Default --verbose 2</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>LimitLoadToSessionType</key>
    <string>Aqua</string>

    <key>StandardOutPath</key>
    <string>/tmp/proxybridge-output.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/proxybridge-error.log</string>
</dict>
</plist>
```

### 4.5. Установить plist и загрузить LaunchAgent

Скопируй оба plist в `~/Library/LaunchAgents/`. Если скачал их из этой инструкции, сначала переименуй файлы в исходные имена:

```text
com.local.shadowsocks.plist
com.local.proxybridge.plist
```

Проверь их синтаксис:

```bash
plutil -lint ~/Library/LaunchAgents/com.local.shadowsocks.plist
plutil -lint ~/Library/LaunchAgents/com.local.proxybridge.plist
```

Загрузить оба агента:

```bash
launchctl bootstrap gui/$(id -u) \
  ~/Library/LaunchAgents/com.local.shadowsocks.plist

launchctl bootstrap gui/$(id -u) \
  ~/Library/LaunchAgents/com.local.proxybridge.plist
```

Проверить состояние:

```bash
launchctl print gui/$(id -u)/com.local.shadowsocks
launchctl print gui/$(id -u)/com.local.proxybridge
```

Перезапустить отдельный агент:

```bash
launchctl kickstart -k gui/$(id -u)/com.local.shadowsocks
launchctl kickstart -k gui/$(id -u)/com.local.proxybridge
```

Остановить и выгрузить отдельный агент:

```bash
launchctl bootout gui/$(id -u)/com.local.shadowsocks
launchctl bootout gui/$(id -u)/com.local.proxybridge
```

После изменения plist сначала выгрузи соответствующий агент через `bootout`, затем снова выполни `bootstrap`.

### 4.6. Логи

Shadowsocks:

```bash
tail -f /tmp/shadowsocks_output.log
tail -f /tmp/shadowsocks_error.log
```

ProxyBridge:

```bash
tail -f /tmp/proxybridge-output.log
tail -f /tmp/proxybridge-error.log
```

У каждого процесса собственные файлы логов.

**Важно:** в моём plist для ProxyBridge `KeepAlive` включён, а для Shadowsocks — нет. Это намеренно сохранено из предоставленных файлов. Также оба агента запускаются независимо друг от друга; задержка 15 секунд задана только для ProxyBridge.

После того как ProxyBridge настроен и проверен, вернись в **System Settings → Wi-Fi → Details → Proxies** и отключи **SOCKS Proxy**. Не оставляй одновременно глобальный системный SOCKS5 и маршрутизацию ProxyBridge включёнными без необходимости: это может привести к тому, что системный прокси будет вмешиваться в выбранную схему маршрутов.

## 5. Проверка всей цепочки

Проверять последовательно:

1. **Сервер:** служба Shadowsocks активна:

   ```bash
   sudo systemctl status shadowsocks-libev --no-pager
   ```

2. **Клиент:** `ss-local` запущен и слушает `127.0.0.1:1080`:

   ```bash
   lsof -nP -iTCP:1080 -sTCP:LISTEN
   ```

3. **SOCKS5:** запрос через локальный прокси проходит:

   ```bash
   curl --socks5-hostname 127.0.0.1:1080 https://1.1.1.1/cdn-cgi/trace
   ```

4. **ProxyBridge:** LaunchAgent загружен:

   ```bash
   launchctl print gui/$(id -u)/com.local.proxybridge
   ```

5. **ProxyBridge:** проверь, что профиль `Default` активен и правила маршрутизации применяются ожидаемым образом.
6. **macOS:** системный SOCKS Proxy выключен; выборочная маршрутизация выполняется через ProxyBridge.

---

## 6. Полезные пути и команды

| Назначение | Значение |
|---|---|
| Серверный конфиг | `/etc/shadowsocks-libev/config.json` |
| Клиентский конфиг | `~/.config/ss-client/config.json` |
| Клиентский бинарник | `/opt/homebrew/bin/ss-local` |
| v2ray-plugin на Mac | `/usr/local/bin/v2ray-plugin` |
| ProxyBridge | `/usr/local/bin/proxybridge` |
| LaunchAgent Shadowsocks | `~/Library/LaunchAgents/com.local.shadowsocks.plist` |
| LaunchAgent ProxyBridge | `~/Library/LaunchAgents/com.local.proxybridge.plist` |
| stdout ProxyBridge | `/tmp/proxybridge-output.log` |
| stderr ProxyBridge | `/tmp/proxybridge-error.log` |
| Локальный SOCKS5 | `127.0.0.1:1080` |
| Серверный порт с obfuscation | `80/TCP` |

---

## 7. Что важно помнить

- На сервере и клиенте должны совпадать пароль и метод шифрования (`aes-256-gcm`).
- При использовании `v2ray-plugin` сервер и клиент должны использовать совместимые настройки плагина.
- На Mac установлен именно `v2ray-plugin` версии `1.5.0`, сборка `darwin-arm64`, вручную из GitHub Releases.
- `ss-local` и `proxybridge` — разные процессы. LaunchAgent из этой инструкции запускает только `proxybridge`.
- SOCKS5 включён на уровне системных настроек macOS, а не в браузере.
- Не отправляй другим людям `config.json`: в нём находится пароль Shadowsocks.
- Shadowsocks — это прокси с шифрованием между клиентом и сервером, а не полноценный VPN-туннель для всего сетевого трафика. Поведение зависит от того, использует ли приложение системный прокси.

## Источник

- [Исходная статья: Set Up Shadowsocks Proxy Service on Linux Server and Mac Client](https://sequoia18.medium.com/set-up-shadowsocks-server-linux-and-client-mac-a72ce2aeabd3)
- [v2ray-plugin 1.5.0 — macOS ARM64 release](https://github.com/shadowsocks/v2ray-plugin/releases/download/v1.5.0/v2ray-plugin-darwin-arm64-v1.5.0.tar.gz)


## Дополнительный источник: ProxyBridge

- [Официальный README ProxyBridge for macOS — Installation и Configuration](https://github.com/InterceptSuite/ProxyBridge/blob/master/MacOS/README.md#installation)
- [Релизы ProxyBridge](https://github.com/InterceptSuite/ProxyBridge/releases)
