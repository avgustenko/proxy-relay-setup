# Прокси через релей (обход DPI оператора)

Прозрачный прокси для всего трафика ноутбука: `sing-box` (tun) → socks5 → SSH-туннель через свой VPS → прокси-сервер.

Работает и на Wi-Fi, и на мобильном интернете, где прямой доступ к прокси блокирует оператор.

---

## Зачем релей

Мобильные операторы (МТС и др.) блокируют прямой доступ к прокси-серверу через DPI:
TCP-рукопожатие проходит, а данные режутся. Порт/протокол при этом не важны —
блокируется сам IP прокси.

Решение — **релей**: трафик идёт на свой VPS (его оператор не блокирует),
а с VPS — уже на прокси-сервер.

```
приложения → sing-box tun → socks 127.0.0.1:1080 → SSH-туннель → VPS → socks прокси
```

---

## Что нужно

- Ubuntu 24.04+ / Debian, права `sudo`
- `sing-box` (пакет)
- свой VPS с SSH-доступом (к нему идёт туннель)
- данные прокси: `socks5` хост, порт, логин, пароль

---

## Данные, которые нужно взять у владельца

| Что | Переменная | Примечание |
|---|---|---|
| Прокси: хост | `ПРОКСИ_ХОСТ` | IP прокси-сервера |
| Прокси: порт | `ПРОКСИ_ПОРТ` | обычно 64333 |
| Прокси: логин | `ЛОГИН` | — |
| Прокси: пароль | `ПАРОЛЬ` | — |
| VPS: IP | `VPS_IP` | релей-сервер |
| VPS: пользователь | `VPS_USER` | обычно root |
| SSH-ключ к VPS | — | приватный ключ, добавить в `authorized_keys` на VPS |

> Секреты (логин, пароль, приватный SSH-ключ) в этот репозиторий **не коммитить**.

---

## Шаги

### 1. Установить sing-box

```bash
curl -L -o /tmp/sing-box.deb \
  https://github.com/SagerNet/sing-box/releases/latest/download/sing-box_1.14.4_linux_amd64.deb
sudo dpkg -i /tmp/sing-box.deb
sing-box version
```

(если архитектура не amd64 — взять подходящий пакет из
[releases](https://github.com/SagerNet/sing-box/releases)).

### 2. SSH-ключ до VPS

```bash
ssh-keygen -t ed25519 -f ~/.ssh/relay_ed25519 -N ''      # если ключа ещё нет
ssh-copy-id -i ~/.ssh/relay_ed25519 VPS_USER@VPS_IP       # добавить ключ на VPS
ssh -i ~/.ssh/relay_ed25519 VPS_USER@VPS_IP 'echo ok'     # проверить
```

### 3. Конфиг sing-box

Скопировать `config/sing-box-config.json` в `/etc/sing-box/config.json`,
заменив `ПРОКСИ_ХОСТ`, `ПРОКСИ_ПОРТ`, `ЛОГИН`, `ПАРОЛЬ`, `VPS_IP`:

```bash
sudo mkdir -p /etc/sing-box
sudo cp config/sing-box-config.json /etc/sing-box/config.json
sudo nano /etc/sing-box/config.json   # подставить свои значения
```

### 4. systemd user-юнит для SSH-туннеля

```bash
mkdir -p ~/.config/systemd/user
cp config/relay-tunnel.service ~/.config/systemd/user/
```

В файле подставить `VPS_USER`, `VPS_IP`, `ПРОКСИ_ХОСТ`, `ПРОКСИ_ПОРТ`
и правильный путь к ключу (`/home/ИМЯ/.ssh/relay_ed25519`).

```bash
systemctl --user daemon-reload
systemctl --user enable --now relay-tunnel.service
systemctl --user status relay-tunnel.service   # должно быть active
```

### 5. Включить linger (туннель переживает перезагрузку)

```bash
sudo loginctl enable-linger "$(whoami)"
```

### 6. (опционально) kill-switch — блокировка прямого выхода

Чтобы приложение не могло выйти в интернет мимо прокси с реальным IP:

```bash
sudo cp config/killswitch.nft /etc/nftables-proxy.nft
sudo nft -f /etc/nftables-proxy.nft
```

В файле подставить `VPS_IP` и локальную подсеть (см. комментарии в файле).

### 7. Команда `proxy`

```bash
cp bin/proxy ~/.local/bin/proxy
chmod +x ~/.local/bin/proxy
```

Проверить, что `~/.local/bin` в `PATH` (иначе добавить в `.bashrc`).

### 8. Проверка

```bash
proxy on
proxy ip           # внешний IP = IP прокси-сервера
curl -s https://ifconfig.me    # должен показать IP прокси, а не свой
curl -s -o /dev/null -w '%{http_code}\n' https://example.com   # 200
```

---

## Управление

```bash
proxy on        # включить
proxy off       # выключить
proxy status    # статус sing-box
proxy log       # логи
proxy ip        # текущий внешний IP
```

Туннель (`relay-tunnel.service`) живёт сам — `proxy on/off` управляет только sing-box.

---

## Файлы репозитория

```
README.md
config/
  sing-box-config.json    # шаблон конфига sing-box (tun → socks через туннель)
  relay-tunnel.service    # systemd user-юнит SSH-туннеля
  killswitch.nft          # nftables kill-switch (опционально)
bin/
  proxy                   # команда управления
```

---

## Как это работает (кратко)

1. `sing-box` поднимает виртуальный интерфейс `singtun0` и направляет весь трафик в него.
2. Внутри sing-box трафик уходит в socks5-выход на `127.0.0.1:1080`.
3. Порт `1080` — это SSH-туннель (`relay-tunnel.service`) до VPS.
4. VPS открывает соединение к прокси-серверу (`ПРОКСИ_ХОСТ:ПРОКСИ_ПОРТ`).
5. В конфиге `route_exclude_address` для `VPS_IP` — чтобы SSH к VPS не зациклился в tun.
