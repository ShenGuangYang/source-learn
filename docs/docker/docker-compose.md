# docker compose

## docker compose 简介

## docker compose 实战

### 使用docker compose 搭建 pg14

`docker-compose.yml` 

```yaml
services:
  db:
    image: postgres:14
    container_name: postgres-dev
    restart: always
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      PGDATA: /var/lib/postgresql/data/
      TZ: Asia/Shanghai
      POSTGRES_HOST_AUTH_METHOD: "md5"
    command: -c 'config_file=/etc/postgresql/postgresql.conf'
    ports:
      - 1921:5432
    volumes:
      - /data/postgres/pg_data:/var/lib/postgresql/data
      - /etc/localtime:/etc/localtime:ro
      - /data/postgres/postgresql.conf:/etc/postgresql/postgresql.conf
      - /data/postgres/pg_hba.conf:/etc/postgresql/pg_hba.conf
```

启动成功后进入容器执行 

```
echo "host all all all $POSTGRES_HOST_AUTH_METHOD" >> /var/lib/postgresql/data/pg_hba.conf
```
