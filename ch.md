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

================================================================================

Отличное решение! Вот готовый рабочий вариант с отдельным контейнером clickhouse-backup, который подключается к работающему ClickHouse по сети:

1. docker-compose.yml для clickhouse-backup
yaml
version: '3.8'

services:
  clickhouse-backup:
    image: altinity/clickhouse-backup:latest  # Официальный образ с утилитой
    container_name: ch-backup
    restart: unless-stopped
    volumes:
      - ./backups:/var/lib/clickhouse/backup  # Монтируем папку для бэкапов
      - ./config.yml:/etc/clickhouse-backup/config.yml  # Конфиг
    environment:
      CLICKHOUSE_HOST: "172.17.0.1"  # Ваш IP ClickHouse (см. пояснение ниже)
      CLICKHOUSE_PORT: "9000"
      CLICKHOUSE_USER: "admin"
      CLICKHOUSE_PASSWORD: "your_password"
    networks:
      - host_network  # Используем сеть хоста

networks:
  host_network:
    name: host  # Подключаемся к сети хоста
    external: true
2. Конфигурационный файл config.yml
yaml
general:
  remote_storage: none
  backups_to_keep_local: 3

clickhouse:
  username: "${CLICKHOUSE_USER}"
  password: "${CLICKHOUSE_PASSWORD}"
  host: "${CLICKHOUSE_HOST}"
  port: ${CLICKHOUSE_PORT}
  skip_tables:
    - "system.*"
    - "INFORMATION_SCHEMA.*"
3. Как определить IP ClickHouse:
Если ClickHouse работает на том же хосте:

bash
# Для Linux:
CLICKHOUSE_HOST="172.17.0.1"  # Docker bridge network

# Для Windows/Mac (Docker Desktop):
CLICKHOUSE_HOST="host.docker.internal"
Если ClickHouse в другом контейнере:

bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ваш_clickhouse_container
4. Запуск и использование
Создайте папку для бэкапов:

bash
mkdir -p ./backups
Запустите контейнер:

bash
docker-compose up -d
Создание бэкапа:

bash
docker exec ch-backup clickhouse-backup create my_backup
Бэкап появится в ./backups/my_backup/

Список бэкапов:

bash
docker exec ch-backup clickhouse-backup list
5. Восстановление на другом сервере
Скопируйте папку ./backups/my_backup на новый сервер

Запустите такой же контейнер clickhouse-backup

Выполните:

bash
docker exec ch-backup clickhouse-backup restore my_backup
Ключевые особенности
Автономность:

Не требует установки clickhouse-backup в основной контейнер

Не мешает работе ClickHouse

Доступ к бэкапам:

Файлы доступны на хосте в ./backups

Можно копировать через scp/rsync

Безопасность:

Пароль передается через переменные окружения

Доступ только по указанному IP

Дополнительные команды
Команда	Описание
docker exec ch-backup clickhouse-backup delete local my_backup	Удалить бэкап
docker exec ch-backup clickhouse-backup tables	Список таблиц
docker exec ch-backup clickhouse-backup restore --schema my_backup	Только структура
Если ClickHouse слушает нестандартный порт
Измените в config.yml:

yaml
clickhouse:
  port: 9100  # Ваш кастомный порт
Типичные проблемы
"Connection refused":

Проверьте netstat -tulnp | grep clickhouse

Убедитесь, что ClickHouse принимает подключения на 0.0.0.0

"Access denied":

Проверьте логин/пароль в docker-compose.yml

Убедитесь, что пользователь имеет права BACKUP

Нет сети host:

bash
docker network create host
Это решение работает для любой версии ClickHouse и не требует модификации основного контейнера.
