Реализовано две версии (первая через установку node-exporter, вторая через ранее созданный руками node-exporter, разница только в месте указания пути до файла сохранения метрик про telegram)


```
sudo apt update
sudo apt install -y prometheus-node-exporter curl
sudo systemctl enable --now prometheus-node-exporter
```
После установки node-exporter на клиентов необходимо создать отдельную директорию для хранения данных и отредактировать путь в systemd


```
sudo mkdir -p /opt/node-exporter/textfile_collector
sudo chmod 755 /opt/node-exporter/textfile_collector
```
sudo systemctl edit --full node-exporter
В случае собранном руками node-exporter указываем директорию так

systemctl edit --full node-exporter

```
[Unit]
Description=Node Exporter

[Service]
ExecStart=/opt/node-exporter/node_exporter --collector.systemd --web.listen-address=0.0.0.0:9100 --collector.textfile.directory=/opt/node-exporter/textfile_collector
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```
Если установлен пакет prometheus-node-exporter выполняем команды ниже


```
sudo install -d -o prometheus \
  -g prometheus -m 0755 \
  /var/lib/node_exporter/textfile_collector
```
в /etc/default/prometheus-node-exporter сохраним текущие параметры и добавь к ним:

Не перезаписываем существующие ARGS, если в них уже указаны другие параметры. Нужно именно добавить новый параметр к текущему списку.


```
ARGS="--collector.textfile.directory=/var/lib/node_exporter/textfile_collector"
```
Создаем скрипт для мониторинга телеграм


```
sudo nano /usr/local/bin/check-telegram-api.sh
```
```
#!/usr/bin/env bash

set -u

METRICS_DIR="/opt/node-exporter/textfile_collector"
METRICS_FILE="${METRICS_DIR}/telegram_api.prom"

TMP_FILE=$(mktemp "${METRICS_DIR}/.telegram_api.XXXXXX") || exit 1
trap 'rm -f "$TMP_FILE"' EXIT

URL="https://api.telegram.org/bot"

# Проверяем HTTP-ответ
HTTP_CODE=$(
    curl \
        --silent \
        --show-error \
        --output /dev/null \
        --write-out '%{http_code}' \
        --connect-timeout 5 \
        --max-time 10 \
        "$URL" 2>/dev/null
)

CURL_EXIT=$?

# Любой HTTP-ответ 1xx-4xx подтверждает,
# что Telegram API доступен на сетевом уровне.
if [[ "$CURL_EXIT" -eq 0 && "$HTTP_CODE" =~ ^[1-4][0-9][0-9]$ ]]; then
    STATUS=1
else
    STATUS=0
fi

TIMESTAMP=$(date +%s)

cat > "$TMP_FILE" <<EOF
# HELP telegram_api_up Telegram API availability (1=up, 0=down).
# TYPE telegram_api_up gauge
telegram_api_up ${STATUS}

# HELP telegram_api_last_check_timestamp_seconds Unix timestamp of the last Telegram API check.
# TYPE telegram_api_last_check_timestamp_seconds gauge
telegram_api_last_check_timestamp_seconds ${TIMESTAMP}
EOF

chmod 644 "$TMP_FILE"
mv -f "$TMP_FILE" "$METRICS_FILE"

trap - EXIT
```
Назначаем права на выполнения и создаем corn на проверку каждую минуту


```
sudo chmod +x /usr/local/bin/check-telegram-api.sh

sudo nano /etc/cron.d/check-telegram-api
```
```
* * * * * root /usr/local/bin/check-telegram-api.sh
```
Выполняем команды запуска и проверки


```
sudo chown root:root /etc/cron.d/check-telegram-api
sudo chmod 644 /etc/cron.d/check-telegram-api
sudo chmod +x /usr/local/bin/check-telegram-api.sh


sudo systemctl enable --now cron
sudo systemctl status cron --no-pager

sudo /usr/local/bin/check-telegram-api.sh

cat /opt/node-exporter/textfile_collector/telegram_api.prom
```
