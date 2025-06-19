version: '3'
services:
  tabix:
    image: spoonest/clickhouse-tabix-web-client
    ports:
      - "8080:80"
    environment:
      - CH_HOST=clickhouse-server  # Имя сервиса ClickHouse в Docker
      - CH_PORT=8123
    depends_on:
      - clickhouse-server

  clickhouse-server:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
    environment:
      - CLICKHOUSE_USER=admin
      - CLICKHOUSE_PASSWORD=123456



1. Экспорт через clickhouse-client (наиболее надежный)
bash
# Подключаемся к контейнеру и делаем дамп
docker exec -it clickhouse-server bash -c \
"clickhouse-client --query='SHOW TABLES' | xargs -I{} clickhouse-client --query='SELECT * FROM {} FORMAT CSV' > /tmp/all_data.csv"

# Копируем файл на хост
docker cp clickhouse-server:/tmp/all_data.csv ./clickhouse_dump_$(date +%Y%m%d).csv
2. Прямое подключение с хоста (если открыты порты)
bash
clickhouse-client --host 127.0.0.1 --port 9000 \
--query="SELECT * FROM database.table FORMAT CSV" > dump.csv
3. Использование clickhouse-backup (для полного бэкапа)
bash
# В контейнере
docker exec -it clickhouse-server bash -c \
"clickhouse-backup create backup_name"

# Копируем бэкап
docker cp clickhouse-server:/var/lib/clickhouse/backup/backup_name ./clickhouse_backup
