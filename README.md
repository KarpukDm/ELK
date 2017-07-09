#### For log analysis used the logstash-elasticsearch-kibana (ELK) stack.  

* `Elasticsearch` is used as a database.
* `Logstash` is necessary for receiving and parsing the incoming logs and sending it to elasticsearch.
* `Kibana` selects data from the elasticsearch and display it. Also allows you to present the result in the form of graphs and diagrams.

`ELK` stack is installed on a single machine that accepts logs from many machines.
To start the ELK stack with the docker, create a file named "docker-compose.yml" with the following content:  

```markdown
version: '3.0'
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:5.4.0
    volumes:
      - path/to/logstash.conf/:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - 5044:5044
    depends_on:
      - elasticsearch
    networks:
      - elastic-stack
 
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:5.4.0
    ports:
      - 9200:9200
    networks:
      - elastic-stack
 
  kibana:
    image: docker.elastic.co/kibana/kibana:5.4.0
    ports: 5601:5601
    depends_on:
      - elasticsearch
    networks:
      - elastic-stack
 
networks:
  elastic-stack:
    driver: bridge
```

#### In the logstash section, specify the path to logstash.conf  
```markdown
input {
  beats {
    port => 5044
  }
}
 
filter {
  #Grokking Spring Boot's default log format
  grok {
    match => [ "message",
               "(?<timestamp>%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME})  %{LOGLEVEL:level} %{NUMBER:pid} --- \[(?<thread>[A-Za-z0-9-]+)\] [A-Za-z0-9.]*\.(?<class>[A-Za-z0-9#_]+)\s*:\s+(?<logmessage>.*)",
               "message",
               "(?<timestamp>%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME})  %{LOGLEVEL:level} %{NUMBER:pid} --- .+? :\s+(?<logmessage>.*)"
             ]
  }
 
  #Parsing out timestamps which are in timestamp field thanks to previous grok section
  date {
    match => [ "timestamp" , "yyyy-MM-dd HH:mm:ss.SSS" ]
  }
}
 
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    user => elastic
    password => changeme
  }
  stdout { codec => rubydebug }
}
```
To send logs to the logstash, you need a `filebeat`. It is installed on the each machine, to which logs are stored.
The easiest way to install the filebeat is to install it as a service.
To installing with help the docker you need to create this files:  
* docker-compose.yml
```markdown
filebeat:
   build: path/to/Dockerfile
   volumes:
   - ./filebeat.yml:/filebeat.yml
   - /path/to/application.log:/application.log
```  
* Dockerfile  
```markdown
# Dockerfile for Filebeat
 
# Build with:
# docker build -t "filebeat" .
 
# Alpine OS 3.4
# http://dl-cdn.alpinelinux.org/alpine/v3.4/community/x86_64/
FROM alpine:3.4
 
MAINTAINER Tuan Vo <vohungtuan@gmail.com>
 
###############################################################################
#                                INSTALLATION
###############################################################################
 
ENV FILEBEAT_VERSION=5.0.0
 
RUN set -x \
 && apk add --update bash \
                     curl \
                     tar \
                openssl \
 && curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz \
 && tar xzvf filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz -C / --strip-components=1 \
 && rm -rf filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz \
 && apk --no-cache add ca-certificates \
 && wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://raw.githubusercontent.com/sgerrand/alpine-pkg-glibc/master/sgerrand.rsa.pub \
 && wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.23-r3/glibc-2.23-r3.apk \
 && apk add glibc-2.23-r3.apk \
 && apk del curl \
            tar \
         openssl \
 && rm -rf /var/cache/apk/*
 
###############################################################################
#                                   START
###############################################################################
 
COPY docker-entrypoint.sh /
RUN chmod +x docker-entrypoint.sh filebeat \
 && ln -s /filebeat /bin/filebeat
 
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD []
```  
* docker-entrypoint.sh  
```markdown

#!/bin/bash
 
set -xe
 
# If user don't provide any command
# Run filebeat
if [[ "$1" == "" ]]; then
     exec filebeat -e -c /filebeat.yml -d "publish"
else
    # Else allow the user to run arbitrarily commands like bash
    exec "$@"
fi
```
#### For the first and second cases, you must create a configuration file  
* filebeat.yml  
```markdown
filebeat.prospectors:
- input_type: log
  paths:
    - /application.log
output.logstash:
  hosts: ["serverIp:5044"]
```  
Run the command "docker-compose up" for the ELK and "docker-compose up" for the filebeat  
#### NOTE  

When starting ELK, the following problem may occur:  
```markdown
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```  
To solve the problem, run the command:  
```markdown
sudo sysctl -w vm.max_map_count=262144
```