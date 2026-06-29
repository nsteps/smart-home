# Raspberry Pi 5 Home Server

Инфраструктура домашнего сервера на Raspberry Pi 5, разворачиваемая Ansible. Один прогон плейбука превращает чистую Raspberry Pi OS Lite в хост умного дома: устанавливает Docker, рендерит конфигурацию и поднимает стек сервисов (автоматизация, MQTT, мониторинг) в контейнерах Docker Compose.

Проект описателен и воспроизводим: версии образов зафиксированы, секреты хранятся в Ansible Vault, а корректность ролей проверяется линтерами и Molecule в CI.

## Архитектура

Управляющая нода (ноутбук с Ansible) по SSH конфигурирует одну Raspberry Pi. На Pi разворачивается единый Docker Compose-проект в `/opt/home-server`; все сервисы общаются по внутренней bridge-сети Compose, наружу проброшены только нужные порты.

```
control node (Ansible)  ──ssh──▶  Raspberry Pi 5
                                     ├─ Docker Engine + Compose
                                     └─ /opt/home-server (docker-compose.yml + конфиги)
                                          ├─ Home Assistant ──┐
                                          ├─ Mosquitto (MQTT) ◀┘ (127.0.0.1:1883)
                                          ├─ Zigbee2MQTT ──▶ Mosquitto   (опц., нужен донгл)
                                          ├─ Speedtest Tracker
                                          └─ Prometheus ──▶ scrape ──▶ Node Exporter / cAdvisor
                                                  ▲
                                                  └── Grafana (provisioned datasource + dashboard)
```

Поток мониторинга: Node Exporter отдаёт метрики хоста, cAdvisor — метрики контейнеров, Prometheus их собирает и хранит, Grafana визуализирует через автоматически провиженный datasource и дашборд «Home Server».

## Компоненты

| Сервис | Образ | Порт | Назначение |
| --- | --- | --- | --- |
| Home Assistant | `ghcr.io/home-assistant/home-assistant:2026.6` | 8123 | Хаб автоматизации умного дома. Работает в `host` network mode; к MQTT обращается по `127.0.0.1:1883`. |
| Mosquitto | `eclipse-mosquitto:2.0.22` | 1883 | MQTT-брокер. По умолчанию анонимный доступ в LAN. |
| Zigbee2MQTT | `ghcr.io/koenkk/zigbee2mqtt:2.12.0` | 8080 | Мост Zigbee-координатора в MQTT. Опционален, требует USB-донгл; по умолчанию выключен. |
| Speedtest Tracker | `lscr.io/linuxserver/speedtest-tracker:1.14.4` | 8081 | Периодические замеры скорости интернета и история. |
| Prometheus | `prom/prometheus:v3.12.0` | 9090 | Сбор и хранение метрик (retention 30 дней). |
| Grafana | `grafana/grafana-oss:13.0.2` | 3000 | Дашборды. Datasource Prometheus и дашборд «Home Server» провиженятся автоматически. |
| cAdvisor | `ghcr.io/google/cadvisor:v0.53.0` | 8088 | Метрики ресурсов по каждому контейнеру. |
| Node Exporter | `quay.io/prometheus/node-exporter:v1.11.1` | 9100 | Метрики хоста (CPU, RAM, диск, сеть). |
| Uptime Kuma | `louislam/uptime-kuma:1` | 3001 | Мониторинг доступности. Опционален, по умолчанию выключен. |
| Tailscale | `tailscale/tailscale:stable` | — | Mesh-VPN для удалённого доступа. Опционален, по умолчанию выключен. |

Каждый сервис включается/выключается флагом `*_enabled` в `group_vars/all.yml`; выключенные не попадают ни в Compose-файл, ни в конфиг Prometheus, ни в правила фаервола.

## Структура репозитория

```
.
├── ansible.cfg                  # настройки Ansible (inventory, vault password file)
├── site.yml                     # плейбук: роли base → docker → home_server
├── requirements.yml             # Galaxy-коллекции (community.docker/general, ansible.posix)
├── inventory/
│   ├── hosts.yml.example        # шаблон инвентори (реальный hosts.yml в .gitignore)
├── group_vars/
│   └── all.yml                  # вся конфигурация: сервисы, порты, образы, тюнинг
├── vault.yml                    # секреты (Ansible Vault, зашифрован)
├── vault.example.yml            # шаблон секретов
├── roles/
│   ├── base/                    # ОС: пакеты, hostname, timezone, avahi, UFW
│   ├── docker/                  # Docker Engine, Compose plugin, daemon-тюнинг
│   └── home_server/             # директории, рендер конфигов, деплой стека
├── molecule/default/            # сценарий тестов (converge/idempotence/verify)
└── .github/workflows/ci.yml     # CI: lint + molecule
```

## Роли

**base** — приводит ОС в рабочее состояние: проверяет, что это Debian-семейство, обновляет систему, ставит базовые пакеты, задаёт hostname и timezone, поднимает avahi-daemon, опционально настраивает UFW (список разрешённых портов вычисляется из включённых сервисов).

**docker** — устанавливает Docker Engine и Compose plugin из официального репозитория Docker, включает сервис, добавляет пользователя в группу `docker` и, если задан `docker_daemon_options`, пишет `/etc/docker/daemon.json` (например, MTU) с перезапуском демона.

**home_server** — определяет путь к Zigbee-донглу (при включённом Zigbee2MQTT), создаёт дерево каталогов в `/opt/home-server`, рендерит все конфиги (Mosquitto, Zigbee2MQTT, Prometheus, провижининг Grafana, `docker-compose.yml`), разворачивает стек через `community.docker.docker_compose_v2` и дожидается готовности сервисов на их портах (для Home Assistant — дополнительно HTTP-проверка готовности).

## Конфигурация

Вся пользовательская конфигурация — в `group_vars/all.yml`:

- **Хост:** `pi_hostname`, `server_timezone`, `server_user`, `base_dir`.
- **Сервисы:** флаги `*_enabled` и порты `*_port`.
- **Образы:** теги зафиксированы (никаких `:latest`) для воспроизводимости.
- **Docker:** `docker_daemon_options` — содержимое `daemon.json` (по умолчанию `mtu: 1280` и `max-concurrent-downloads: 1`, см. раздел «Сеть»).
- **Сервисные параметры:** retention Prometheus, расписание Speedtest, тип Zigbee-адаптера и т.д.

Инвентори (`inventory/hosts.yml`) содержит реальные адрес и пользователя Pi и намеренно не коммитится — в репозитории лежит только `hosts.yml.example`.

## Развёртывание

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml -K        # -K запрашивает sudo-пароль на Pi
```

Плейбук идемпотентен: повторный прогон не вносит изменений, если конфигурация не менялась. Сервисы развёрнуты в `/opt/home-server`; управлять стеком можно и напрямую (`docker compose ps|logs|up -d|down`) из этого каталога.

Предпосылки: на Pi установлена Raspberry Pi OS Lite 64-bit, включён SSH с доступом по ключу, для Zigbee2MQTT воткнут USB-донгл.

## Секреты

Все секреты (пароль администратора Grafana, ключ приложения Speedtest, при использовании — токен Tailscale) хранятся в зашифрованном `vault.yml` (Ansible Vault). `group_vars/all.yml` ссылается на них как на `vault_*`-переменные. Файл с паролем хранилища (`.vault_pass`) в репозиторий не попадает; `ansible.cfg` подхватывает его автоматически. Шаблон ключей — в `vault.example.yml`.

## Тестирование и CI

- **yamllint** и **ansible-lint** (профиль `basic`) — статический анализ.
- **Molecule** (`molecule/default`) — `converge → idempotence → verify` ролей `base` и `docker` в systemd-контейнере Debian 12. Полный сервисный стек на реальном железе не тестируется в CI (нужен Zigbee-донгл и многогигабайтные образы).
- **GitHub Actions** (`.github/workflows/ci.yml`) — джобы `lint` и `molecule` на push в `main` и на PR. Для расшифровки vault в CI задаётся секрет `ANSIBLE_VAULT_PASSWORD`.

Задачи, несовместимые с контейнерным окружением (правка `/etc/hostname`, `/etc/hosts`), помечены тегом `molecule-notest` и пропускаются только в Molecule, на реальном хосте выполняясь штатно.

## Модель безопасности

Конфигурация по умолчанию рассчитана на доверенный LAN:

- Mosquitto разрешает анонимные подключения (`mosquitto_allow_anonymous: true`).
- UFW выключен; при включении список открытых TCP-портов формируется из активных сервисов.
- Внешний доступ (Tailscale) выключен.

Для ужесточения: включить UFW, настроить аутентификацию/ACL в Mosquitto, поднять Tailscale вместо проброса портов наружу, а Prometheus-эндпоинт Speedtest защитить токеном (`speedtest_metrics_enabled` + `vault_speedtest_metrics_bearer_token`).

## Сеть

Скачивание образов с некоторых CDN может зависать на каналах с двойным NAT/PPPoE из-за чёрной дыры Path MTU Discovery: TLS-рукопожатие проходит, а крупные слои встают. Лечится уменьшением MTU Docker (`docker_daemon_options.mtu`, по умолчанию 1280) — роль `docker` пишет это в `daemon.json` до деплоя. Радикальное решение на уровне всей сети — MSS clamping на WAN-интерфейсе роутера. Если IPv6-маршрут до registry нерабочий, Docker стоит ограничить IPv4.
