# Monitoring stack

[![CI](https://github.com/RealPycuk/monitoring-stack/actions/workflows/ci.yml/badge.svg)](https://github.com/RealPycuk/monitoring-stack/actions/workflows/ci.yml)

Стек мониторинга для небольшого сервера на Docker Compose: Prometheus, Alertmanager, Grafana, node_exporter и blackbox_exporter, алерты в Telegram. Этот же стек работает на моём сервере и следит за Telegram-ботом и самим хостом. На сервер его ставит Ansible-роль из [ansible-server-setup](https://github.com/RealPycuk/ansible-server-setup).

## Что внутри

```
                    ┌──────────────── host network, только 127.0.0.1 ────────────────┐
  node_exporter ──► │                                                                  │
  blackbox ───────► │  Prometheus ──► alerts.yml ──► Alertmanager ──► Telegram          │
  приложение ─────► │      │                                                           │
  (/metrics)        │      └──────► Grafana (дашборд из provisioning)                   │
                    └──────────────────────────────────────────────────────────────────┘
                     Grafana открывается через SSH-туннель: ssh -L 3000:127.0.0.1:3000 user@server
```

| Сервис | Порт | Зачем |
|---|---|---|
| Prometheus 3.2 | 9090 | сбор метрик, правила алертов, хранение 30 дней / 1 ГБ |
| Alertmanager 0.28 | 9093 | группировка, подавление дублей, отправка в Telegram |
| Grafana 11.5 | 3000 | дашборд «Сервис: обзор» — приложение, сервер, доступность |
| node_exporter 1.9 | 9100 | CPU, память, диск, сеть, OOM |
| blackbox_exporter 0.26 | 9115 | TCP-проверки портов и HTTPS со сроком сертификата |

Все версии образов закреплены. Контейнеры работают с `read_only`, `no-new-privileges` и лимитами памяти, наружу ничего не публикуется.

## Алерты

| Алерт | Условие | Важность |
|---|---|---|
| BotDown | приложение не отдаёт `/metrics` больше 2 минут | critical |
| BotHandlerErrors | больше 5 необработанных исключений за 15 минут | warning |
| PanelApiErrors | больше 20 ошибок внешнего API за 15 минут | warning |
| PanelApiSlow | p95 ответа внешнего API больше 5 секунд | warning |
| BackupMissing | последний успешный бэкап старше 26 часов | critical |
| HostMemoryLow | свободной памяти меньше 10% | warning |
| HostDiskWillFillIn24h | по тренду за 6 часов диск заполнится за сутки | warning |
| HostHighLoad | load15 на ядро больше 2 | warning |
| HostOomKill | сработал OOM killer | critical |
| TargetDown | любая цель мониторинга, кроме приложения, недоступна 5 минут | warning |

Если упало само приложение, Alertmanager не шлёт вдогонку алерты о его ошибках (`inhibit_rules`). Правила покрыты юнит-тестами в `prometheus/alerts_test.yml`: проверяется и срабатывание, и то, что алерта нет раньше времени.

## Запуск

```bash
cp .env.example .env                                         # пароль администратора Grafana
cp alertmanager/alertmanager.example.yml alertmanager/alertmanager.yml   # chat_id
echo "<токен бота>" > alertmanager/bot_token && chmod 600 alertmanager/bot_token
cp prometheus/targets/tcp.example.yml prometheus/targets/tcp.yml         # что проверять blackbox'ом
cp prometheus/targets/https.example.yml prometheus/targets/https.yml
docker compose up -d --wait
```

Цели blackbox лежат в `prometheus/targets/*.yml` (file_sd). Prometheus перечитывает их сам, перезапуск не нужен.

## CI

На каждый push:

1. `promtool check config` и `promtool test rules` — конфиг и юнит-тесты алертов;
2. `amtool check-config`, проверка конфига blackbox, валидность JSON дашбордов, `docker compose config`;
3. **smoke-тест**: весь стек поднимается на раннере. Проверяется, что все сервисы готовы, Prometheus опрашивает свои цели и загрузил все 10 правил, blackbox видит открытый порт, а в Grafana появились источник данных и дашборд.
