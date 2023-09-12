# elk
Docker ELK

https://techexpert.tips/elasticsearch/elasticsearch-password-recovery/

Nginx log -> logstash/ELK
Dockerlog -> filebeat/ELK

Dockerfile
```
version: '3'

services:
  # Elasticsearch Docker Images: https://www.docker.elastic.co/
  elasticsearch2:
    image: elasticsearch:7.17.9
    container_name: elasticsearch2
    environment:
      - xpack.security.enabled=false
      - discovery.type=single-node
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    cap_add:
      - IPC_LOCK
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300

  logstash:
    container_name: logstash
    image: logstash:8.8.1
    ports:
      - "9600:9600"
    volumes:
      - ./logstash.conf:/usr/share/logstash/config/logstash.conf
      - /tmp/nginx:/var/log/nginx
    command: logstash -f /usr/share/logstash/config/logstash.conf
    depends_on:
      - elasticsearch2

  kibana:
    container_name: kibana
    image: kibana:7.17.9
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch2

  filebeat:
    image: elastic/filebeat:7.17.9
    command: --strict.perms=false
    user: root
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /var/run/docker.sock:/var/run/docker.sock

  nginx:
    image: nginx:1.25
    ports:
      - 80:80
    volumes:
      - /tmp/nginx:/var/log/nginx
    restart: always

volumes:
  elasticsearch-data:
    driver: local
```
logstash.conf
```
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
  }
}

filter {
    grok {
        match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user_name}
\[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url}
HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes}
\"%{DATA:referrer}\" \"%{DATA:agent}\"" }
    }
    mutate {
        remove_field => [ "host" ]
    }
}

output {
    elasticsearch {
        hosts => "elasticsearch:9200"
        data_stream => "true"
    }
}
```
filebeat.yml
```
filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'

processors:
- add_docker_metadata:
    host: "unix:///var/run/docker.sock"

- decode_json_fields:
    fields: ["message"]
    target: "json"
    overwrite_keys: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  indices:
    - index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

logging.json: true
logging.metrics.enabled: false
```
