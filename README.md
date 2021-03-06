# Сервер мониторинга Prometheus
Настроить дашборд с 4-мя графиками
память;
процессор;
диск;
сеть. Настроить на одной из систем:
zabbix (использовать screen (комплексный экран);
prometheus - grafana.
использование систем, примеры которых не рассматривались на занятии. Список возможных систем был приведен в презентации. В качестве результата прислать скриншот экрана - дашборд должен содержать в названии имя приславшего.

## Подготовка окружения
Сервер CentOS Linux 7 (Core)
Docker и docker-compose (через него развернем Prometheus, Grafana, необходимые экспортеры)
Загружаем необходимые образы и запускаем контейнеры:

### Prometheus
Находим образ Prometheus, пуллим на сервер:
```
docker pull prom/prometheus
```
Подготавлием для него docker-compose:
```
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
            #  - ./prometheus:/prometheus
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
```
Поднимаем контейнер проверяем:
```
[root@yarkozloff monitoring]# docker-compose up -d
Creating network "monitoring_default" with the default driver
Creating prometheus    ... done
```
Открывается страница с Prometheus, успех:
![image](https://user-images.githubusercontent.com/69105791/174459420-8fd7d334-251a-4166-ac5d-866ae09d1bbf.png)

### Grafana
Находим образ Grafana, пуллим на сервер:
```
docker pull grafana/grafana
```
Подготавлием для нее docker-compose (в тот же):
```
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
```
Поднимаем контейнер проверяем:
```
[root@yarkozloff monitoring]# docker-compose up -d
Creating network "monitoring_default" with the default driver
Creating prometheus    ... done
Creating grafana       ... done
```
Открывается страница с Grafana, происходит авторизация через admin,успех:
![image](https://user-images.githubusercontent.com/69105791/174459503-b80f75fa-bc36-4f8b-9cfe-bd477a0f9087.png)

### Node exporter
Node exporter — небольшое приложение, собирающее метрики операционной системы. Устанавливается на сервер, откуда будут собираться метрики. В нашем случае этот же сервер.
Находим образ Node-exporter, пуллим на сервер:
```
docker pull  prom/node-exporter
```
Подготавлием для нее docker-compose (в тот же):
```
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100
```
Поднимаем контейнер проверяем:
```
[root@yarkozloff monitoring]# docker-compose up -d
Creating network "monitoring_default" with the default driver
Creating prometheus    ... done
Creating grafana       ... done
Creating node-exporter ... done
```
На странице наблюдаем как уже собираются метрики:
![image](https://user-images.githubusercontent.com/69105791/174459600-8a91c2e4-9050-470c-81ad-6594fda63c4f.png)

## Сбор метрик
Для того чтобы Prometheus начал собирать в себя метрики с node-exporter, необходимо в конфиг prometheus.yml (он замаплен для удоства в композе) добавить название сборщика метрик и адрес node-exporter. 
Т.к. работаем через докер то в targets нужно указать частный ip либо адрес сервиса в этой сети (в моем случае так и сделано)
```
scrape_configs:
  - job_name: 'yarkozloff_metrics'
    static_configs:
      - targets: ['node-exporter:9100']
```
Проверим, что prometheus собирает метрики (через вебморду это выглядит нагляднее):
![image](https://user-images.githubusercontent.com/69105791/174460130-9e639323-ca36-4abb-a45f-fe5efc8ad5ad.png)

## Настройка Grfana
Для того чтобы приступить к настройке графиков, необходимо подключить Datasource для графаны, выбрав необходимый нам Prometheus (также учитывая что работаем в одной сети достаточно указать название сервиса в адресе):
![image](https://user-images.githubusercontent.com/69105791/174460222-2035c019-b059-4e6e-bf24-979011d185b1.png)
Тут есть проблемка, если перезапустить композ то мой datasource слетит и его приходится добавлять заново, нужно как-то его примапить к контейнеру чтобы хранился на хосте.
Предварительно настроив datasource зайдем в контейнер и видим базу графаны:
```
[root@yarkozloff monitoring]# docker exec -it grafana bash
bash-5.1$ ls /var/lib/grafana
alerting    csv         grafana.db  plugins     png
```
Вынесем этот каталог на хост, замапим в композ и перезапустимся:
```
[root@yarkozloff monitoring]# docker cp grafana:/var/lib/grafana grafana_storage
[root@yarkozloff monitoring]# docker-compose up -d
Recreating grafana ...
node-exporter is up-to-date
Recreating grafana ... done
```
Мапинг для grafana выглядит так:
```
    volumes:
      - ./grafana_storage:/var/lib/grafana
```
Этим самым в будущем решится проблема с дашбордами и прочими настройками

## Настройка графиков
Создадим дашборд с необходимым названием (yarkozloff / otus_14_prometheus)
Далее создаем графики. Пример рассчета cpu usage:
![image](https://user-images.githubusercontent.com/69105791/174463838-b20b2cb2-e762-46a0-a5c5-0b77b81e7a2f.png)

Общий дашборд:
![image](https://user-images.githubusercontent.com/69105791/174463810-04aff99f-a4df-4469-86f4-32b0f60b96e5.png)

## postgres-exporter
Интересно попробовать экспортер для бд. Запросы, написанные в определенный конфиг будут выполняться на бд, а их результаты можно реализовать в метрики.
Для этого нужно поднять postgres. Сделать это на минималках можно добавив в текущий композ:
```
  postgres:
    image: docker.io/library/postgres
    container_name: postgres
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=mydb
    ports:
      - 5432:5432
```
Аналогично поднимаем через docker-compose up -d.
Теперь можно подключиться средствами менеджера бд - Dbeaver для выполнения каких-либо скриптов. Для тестирования экспортера будем выполнять обычные селекты к базе, а для этого нужна таблица с данными (база уже mydb создалась при запуске контейнера). Это сделано следующим образом:
![image](https://user-images.githubusercontent.com/69105791/174464470-e069881a-86bb-4243-a1d6-ed29cd42ebb1.png)
Теперь ставим postgres-exporter, добавив его в тот же композ:
```
  postgres-exporter:
    image: quay.io/prometheuscommunity/postgres-exporter
    container_name: pg-exporter
    environment:
      - DATA_SOURCE_NAME=postgresql://postgres:postgres@postgres:5432/mydb?sslmode=disable
      - PG_EXPORTER_EXTEND_QUERY_PATH=/etc/queries.yaml
      - PG_EXPORTER_DISABLE_SETTINGS_METRICS=true
      - PG_EXPORTER_DISABLE_DEFAULT_METRICS=false
      - PG_EXPORTER_INCLUDE_DATABASES=mydb
    ports:
      - 9187:9187
    volumes:
      - ./queries.yaml:/etc/queries.yaml
```
Конфиг с запросом в бд уже замаплен в контейнер и имеет следующий вид:
```
postgres_metrics:
    query: select "user_name", "order_id" from order_details where "order_name" = 'toy'
    metrics:
     - Alias:
        usage: "LABEL"
        description: "Покупатель"
     - Version:
        usage: "COUNTER"
        description: "Номер заказа"
```
Остается обновить конфиг prometheus чтобы собирались метрики по этому экспортеру:
```
scrape_configs:
  - job_name: 'yarkozloff_metrics'
    static_configs:
      - targets: ['node-exporter:9100']
  
      - job_name: 'postgres_metrics'
    static_configs:
      - targets: ['pg-exporter:9187']
```
После того как переподняли prometheus можем наблюдать новый таргет pg-exporter:
![image](https://user-images.githubusercontent.com/69105791/174465155-c94638cd-864f-463a-ad15-866e8142147c.png)
Идём в графану настраивать график:
![image](https://user-images.githubusercontent.com/69105791/174465933-23f6f819-b84f-4212-8e40-39207c9df9c9.png)
По мимо кастомных скриптов pg-exporter может собирать большое количство метрик связанных с бд, для примера был настроен график количества соединений к имеющимся бд
