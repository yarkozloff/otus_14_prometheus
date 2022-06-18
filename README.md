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
Скачиваем необходимые образы:

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
Для того чтобы Prometheus начал собирать в себя метрики с node-exporter, необходимо в его конфиг добавить следующее:
```
scrape_configs:
  - job_name: 'yarkozloff_metrics'
    static_configs:
      - targets: ['93.185.166.183:9100']
```




