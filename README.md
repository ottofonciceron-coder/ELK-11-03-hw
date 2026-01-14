# Домашнее задание к занятию "ELK" - Марчук Кирилл



### Задание 1.Установите и запустите Elasticsearch, после чего поменяйте параметр cluster_name на случайный.



1. `Устанавливаем и Elasticsearch изапускаем с нестандартным cluster.name`
2. `Проверяем что всё работает через команду curl -X GET "http://localhost:9200/_cluster/health?pretty"`


```

docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "cluster.name=ubuntoo-random-cluster" \
  -e "xpack.security.enabled=false" \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.4

```


---

![zadanie1](https://github.com/ottofonciceron-coder/ELK-11-03-hw/blob/main/Elasticsearch.png)`

---

### Задание 2.Установите и запустите Kibana.

1. `Подключаем Elasticsearch к сети`
2. `Запускаем Kibana`
3. `Заходим в браузере на сервер`
4. `Переходим в Dev Tools - Console и выполняем запрос GET /_cluster/health?pretty.`

```

docker run -d \
  --name kibana \
  --network elastic \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:8.11.4

```

---

![zadanie1](https://github.com/ottofonciceron-coder/ELK-11-03-hw/blob/main/Kibana.png)`

---


### Задание 3.Установите и запустите Logstash и Nginx. С помощью Logstash отправьте access-лог Nginx в Elasticsearch.

1. `Создаём docker-compose.yml`
2. `Создаём Конфиг Logstash`
3. `Запускаем весь стек и генерируем логи Nginx`
4. `В Kibana создаём Data View и Проверяем логи в Discover`


```
#docker-compose.yml

version: "3.8"

services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.4
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - cluster.name=ubuntoo-random-cluster
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.4
    container_name: kibana
    depends_on:
      - elasticsearch
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    networks:
      - elk

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "8080:80"
    volumes:
      - ./nginx/logs:/var/log/nginx
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.4
    container_name: logstash
    depends_on:
      - elasticsearch
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./nginx/logs:/var/log/nginx:ro
    ports:
      - "5044:5044"
    networks:
      - elk

volumes:
  esdata:

networks:
  elk:
    driver: bridge

```


```

#logstash pipeline nginx

input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  grok {
    match => {
      "message" => "%{IPORHOST:client_ip} - - \\[%{HTTPDATE:timestamp}\\] \"%{WORD:method} %{URIPATH:request} HTTP/%{NUMBER:http_version}\" %{NUMBER:status} %{NUMBER:bytes}"
    }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}

```

---

![zadanie1](https://github.com/ottofonciceron-coder/ELK-11-03-hw/blob/main/Logstash.png)`

---


### Задание 4.Установите и запустите Filebeat. Переключите поставку логов Nginx с Logstash на Filebeat.

1. `отключаем отправку логов Nginx через Logstash`
2. `включаем отправку логов Nginx через Filebeat напрямую в Elasticsearch`
3. `Создаем файл filebeat.yml и Добавляем Filebeat в docker-compose.yml`
4. `показываем логи в Kibana`

```
#filebeat.yml

filebeat.inputs:
  - type: filestream
    id: nginx-access
    enabled: true
    paths:
      - /var/log/nginx/access.log

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]

setup.template.enabled: true
setup.ilm.enabled: false

logging.level: info


```

---

![zadanie1](https://github.com/ottofonciceron-coder/ELK-11-03-hw/blob/main/filebeat.png)`

---






