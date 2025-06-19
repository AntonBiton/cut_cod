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
