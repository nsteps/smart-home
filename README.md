# Raspberry Pi 5 Home Server — Ansible + Docker

Готовый проект для раскатки домашнего сервера на Raspberry Pi 5.

## Что поднимется

В базовой конфигурации:
- Home Assistant
- Mosquitto
- Zigbee2MQTT
- Speedtest Tracker
- Prometheus
- Grafana
- cAdvisor
- Node Exporter

Опционально:
- Uptime Kuma
- Tailscale

## Что нужно до запуска

На Raspberry Pi уже должна быть:
- установлена Raspberry Pi OS Lite 64-bit
- включён SSH
- настроен пользователь
- подключён Ethernet или Wi‑Fi
- желательно воткнут Zigbee-донгл, если `zigbee2mqtt_enabled: true`

## Что нужно поправить перед запуском

### 1) Inventory
Открой `inventory/hosts.yml` и укажи:
- `ansible_host`
- `ansible_user`

### 2) Переменные
Открой `group_vars/all.yml` и проверь минимум:
- `pi_hostname`
- `server_timezone`
- `grafana_admin_password`

### 3) Zigbee
Если автопоиск не сработает, пропиши вручную:
```yaml
zigbee_serial_port: /dev/serial/by-id/usb-...
zigbee_adapter: ember
```

Если донгл пока не подключён:
```yaml
zigbee2mqtt_enabled: false
```

## Первый запуск

Установи Ansible на Mac:
```bash
brew install ansible
```

Проверка доступа:
```bash
ansible all -i inventory/hosts.yml -m ping
```

Запуск:
```bash
ansible-playbook -i inventory/hosts.yml site.yml -K
```

Если вход по SSH-паролю:
```bash
ansible-playbook -i inventory/hosts.yml site.yml -kK
```

## Что делает playbook

- обновляет систему
- ставит базовые пакеты
- настраивает hostname и timezone
- ставит Docker Engine + Compose plugin
- создаёт директории в `/opt/home-server`
- рендерит docker-compose и все конфиги
- поднимает стек
- ждёт доступности основных сервисов

## Куда потом ходить

После раскатки:
- Home Assistant: `http://<IP_PI>:8123`
- Zigbee2MQTT: `http://<IP_PI>:8080`
- Speedtest Tracker: `http://<IP_PI>:8081`
- Prometheus: `http://<IP_PI>:9090`
- Grafana: `http://<IP_PI>:3000`
- Node Exporter: `http://<IP_PI>:9100/metrics`
- cAdvisor: `http://<IP_PI>:8088`
- Uptime Kuma: `http://<IP_PI>:3001` (если включишь)

## Полезные замечания

### Home Assistant
Home Assistant работает в `host` network mode. Для MQTT в Home Assistant используй:
- host: `127.0.0.1`
- port: `1883`

### Zigbee
Zigbee2MQTT использует путь `/dev/serial/by-id/...` и пробрасывает его как `/dev/ttyACM0`.

### Grafana
Datasource Prometheus и базовый dashboard провиженятся автоматически.

### Безопасность
Для быстрого старта:
- Mosquitto по умолчанию с `allow_anonymous true`
- UFW выключен
- Tailscale выключен

Потом стоит:
- включить Tailscale
- закрыть наружу доступ
- настроить пароль/ACL в MQTT
- спрятать секреты в Ansible Vault

## Полезные команды

Контейнеры:
```bash
ssh <user>@<ip> 'cd /opt/home-server && docker compose ps'
```

Логи:
```bash
ssh <user>@<ip> 'cd /opt/home-server && docker compose logs -f'
```

Перезапуск:
```bash
ssh <user>@<ip> 'cd /opt/home-server && docker compose up -d'
```

Остановка:
```bash
ssh <user>@<ip> 'cd /opt/home-server && docker compose down'
```

## Важно

Этот архив собран так, чтобы стартовать максимально быстро на свежей Raspberry Pi OS.
Я не прогонял его на твоей конкретной Pi и конкретном Zigbee-донгле, поэтому:
- путь к донглу может потребовать правки
- `zigbee_adapter` может потребовать правки
- если у тебя нестандартный пользователь/UID/GID, поправь `server_uid/server_gid`
