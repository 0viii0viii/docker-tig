# docker-tig

# 개요

로컬에서 실행되고 있는 Spring boot server에서 발생하는 로그를 Docker환경의 Telegraf를 통해 수집하여 influxdb에 저장하고, 저장된 로그를 grafana를 통해 시각화하고 조회하는 테스트를 진행하였습니다.  

## Influxdb

- 많은 쓰기 작업과 쿼리 부하를 처리하기 위해 2013년에 Go 언어로 개발된 오픈소스 TSDB
- Tick Stack의 필수 컴포넌트 중 하나
- 가장 유명하고, 많이 사용되는 TSDB
- Restful API를 제공하고 있어 API 통신이 가능

### 핵심 기능

**Continous Query (연속적인 쿼리) - Task**

- 일정 주기마다 데이터를 처리하여 새롭게 저장하는 기능

**Retention Policy (보존 정책) - Retention Period**

- 데이터를 자동으로 삭제해주는 보존 정책
- 별도의 설정이 없다면 autogen이라는 보존 기간이 무제한인 기본 정책으로 설정

## Telegraf란

- 메트릭을 수집하고 기록하는 plugin-driven 서버 agent
- Tick stack의  필수 컴포넌트 중 하나
- 다양한 input (200 여 가지) 및 output plugin을 지원
- 최소한의 메모리를 차지

## 테스트 환경

**Target** : log.log → Spring boot 서버의 로그를 쌓는 파일

**Docker**

- influxdb: 1.8 / TSDB
- telegraf: 1.8 // 수집기
- grafana: 8.0.2 // 시각화

**Input plugin -** tail

**Output plugin -** influxdb

## 테스트 환경 설정 (Docker)

**docker-compose.yml**

```yaml
influxdb:
    image: influxdb:1.8
    container_name: influxdb
    restart: always
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_REPORTING_DISABLED=false
    volumes:
      - influxdb:/var/lib/influxdb
telegraf:
  image: telegraf:1.8
  container_name: telegraf
  privileged: true
  restart: always
  environment:
    - HOST_PROC=/rootfs/proc
    - HOST_SYS=/rootfs/sys
    - HOST_MOUNT_PREFIX=/rootfs
  net: host
  volumes:
    - C:/applog/log.log:/var/log/log.log  // log file location bind
    - /:/rootfs:ro
    - /proc:/rootfs/proc:ro
    - /sys:/rootfs/sys:ro
    - /var/run/docker.sock:/var/run/docker.sock
    - C:/dev/comeon/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf // your directory
grafana:
  image: grafana/grafana:8.0.2
  container_name: grafana
  env_file: configuration.env
  links:
    - influxdb
  ports:
    - '3000:3000'
  volumes:
    - grafana_data:/var/lib/grafana
    - ./grafana/provisioning/:/etc/grafana/provisioning/
    - ./grafana/dashboards/:/var/lib/grafana/dashboards/
```

```yaml
 - C:/applog/log.log:/var/log/log.log  # bind mount
```

설정에서 가장 중요한 것은 spring의 log가 쌓이는 파일을 telegraf의 volume에 bind시키는 것입니다. 이렇게 되면 로컬에서 쌓이는 log가 container에서도 동기화 됩니다.  

### **Telegraf.conf**

```yaml
[global_tags]

# [agent]
#   interval = "10s" // 수집 주기 
#   round_interval = true
#   metric_batch_size = 1000
#   metric_buffer_limit = 10000
#   collection_jitter = "0s"
#   flush_interval = "10s"
#   flush_jitter = "0s"
```

**[agent]**는 telegraf agent의 설정을 담당합니다. default 설정으로 하면 10초 주기로 로그를 읽어옵니다.

- **interval:** input plugin에서 데이터를 수집하는 간격
- **round_interval:** 수집 간격을 반 올림 합니다. 10s 인 경우 항상 :00, :10, :20 등에서 수집한다는 의미
- **metric_batch_size:** 지표 최대 수집량에 도달하면 output으로 전송
- **collection_jitter:** 지표를 수집하는 input plugin이 여러개 일경우 동시에 수집하는 것을 방지
- **flush_inteval :** output의 작성 빈도 (influxdb에 작성되는 빈도)
- **flush_jitter :**  output plugin이 여러개 일 경우 동시에 influxdb에 write 하는 것을 방지, 5초의 지터와 flush_interval 10s는 10-15초마다 플러시가 발생한다는 의미

**agent configuration 참조:**

[https://runebook.dev/ko/docs/influxdata/telegraf/v1.3/administration/configuration/index](https://runebook.dev/ko/docs/influxdata/telegraf/v1.3/administration/configuration/index)

[**Output plugins]**

```yaml
[[outputs.influxdb]] 
    urls = ["http://localhost:8086"]
    database = "spring_log"

[[outputs.file]]       # docker log에 결과 확인
    files = ["stdout"]
    data_format = "influx"
```

- **[[outputs.influxdb]]**는 influxdb로 가는 output에 대한 설정입니다. 연결할 influxdb의 주소와 생성할 database명을 정의하고 추가적으로 retention policy 또한 정의할 수 있습니다. 더 세부적인 항목에 대해서는 아래의 링크에서 확인할 수 있습니다.
- **[[outputs.file]]**  telegraf에서 input에서 가져온 data를 로그로 출력해주는 output입니다.

 

***configuration 참조***

[https://github.com/influxdata/telegraf/blob/release-1.20/plugins/outputs/influxdb/README.md](https://github.com/influxdata/telegraf/blob/release-1.20/plugins/outputs/influxdb/README.md)

### **[[inputs.tail]]**

```yaml
[[inputs.tail]]
    files = ["/var/log/log.log"]
    from_beginning = false    # true = 파일의 처음부터 읽어옴 false = 현재 시점
    tagexclude = ["path","host"]
    watch_method = "poll" # poll file updates 감지 / inotify 감지 안함
    grok_patterns=["%{CUSTOM_LOG}"]
    name_override = "spring_log"
    grok_custom_patterns = '''
    CUSTOM_LOG %{LOGLEVEL:log_level}\s+%{DATESTAMP:timestamp}\[%{DATA:class}\[%{DATA:subclass}\]%{GREEDYDATA:message}
    '''
    grok_timezone = "Asia/Seoul"
    data_format = "grok"
```

- **files:** 읽어올 로그의 파일을 지정
- **from_beginning**: true = 파일의 처음부터 읽어옴, false = 현재 시점부터 읽어옴
- **tagexclude**: default로 생성된 tag중에 저장하지 않을 태그를 제외
- **watch_metod**: inotify = update 감지하지 않음 / poll = update 감지해서 읽어옴
- **name_override**: measurement = table의 개념, 생성할 measurement의 명칭
- **grok_patterns:** 규칙이 없는 텍스트를 파싱하고 구조를 잡음
- **grok_custom_patterns**: 패턴을 사용자 정의
- **grok_timezone**: timezone을 지정, tag에 time 기본으로 찍힘
- **data_format**: influx / grok / json

configuration 참조: [https://github.com/influxdata/telegraf/blob/release-1.20/plugins/inputs/tail/README.md](https://github.com/influxdata/telegraf/blob/release-1.20/plugins/inputs/tail/README.md)

### **Grok Pattern**

로그를 파싱하여 influxdb에서 구조적으로 필드에 할당하여 저장하기 위해서는, 

알맞은 Grok Pattern을 정의하는 것이 중요합니다. 

아래의 예시는 입력된 로그에 grok pattern을 적용하는 테스트를 해볼 수 있는 기능을 제공합니다. 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a28e4390-f4fa-4c81-825a-dbd947b982c8/Untitled.png)

[Grok filter plugin | Logstash Reference [7.16] | Elastic](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)

[Grok Debugger](https://grokdebug.herokuapp.com/)

## Influxdb 1.8

최신 버전인 influxdb v2.*와 다르게 influxdb 1.* 에서는 

influx db shell 을 통하여 influxdb에 저장된 데이터를 명령어와 쿼리를 통해 조회할 수 있습니다.

*(influxdb 2.* 부터는 influxdb web interface cli를 제공)*

show databases → 데이터베이스 리스트 조회 

show measurements → 테이블 리스트 조회

**spring**_**log measurement(테이블) 조회 예시)**

```bash
$ docker exec -itu 0 influxdb /bin/bash
$ influx
> use <databasename> 
> select * from spring_log
```

## **Grafana**

grafana를 통해 influxdb의 데이터를 조회합니다.

[localhost](http://localhost):3000번 포트로 접속하여 grafana에 접속합니다. 

**1) Configuration 메뉴의 Data sources를 클릭합니다.**

**2) Influxdb와 Grafana를 연결을 설정합니다.**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c5cca5b3-b95d-429c-8faa-e49774208e5e/Untitled.png)

**3) telegraf.conf의 influxdb output에서 정의한 database 명을 설정하고 저장합니다.**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/321c4fb8-a154-48a4-b771-3d4d122d3fc8/Untitled.png)

**4) Explore 메뉴를 통해서 query를 설정합니다.** 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7606815d-477a-4917-bf3e-ce83c28854c6/Untitled.png)

**5) query를 실행하면 influxdb에 저장된 테이블을 보여줍니다.**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/522126e9-ab9d-45e7-8eb1-07dd4a69fffd/Untitled.png)

## Further Information

- Influxdb v2 부터는 web interface cli를 제공
- 1 file per 1 input plugin
- 다중 input & output plugin들을 사용할 때 jitter를 통해 collecting and writing의 과부화를 방지
- Elasticsearch는 v5.* 이상부터 log retention policy 지원하지않고 Cloud Enterprise 구독 환경에서 지원하는 것으로 보여집니다.
- Elasticsearch vs Influxdb → 특정한 테스트환경에서 두 TSDB를 비교하였을 때
    
    특히나 데이터가 클수록, 시스템의 동작시간이 클수록  write performance, on-disk compression, query performance 부분 모두에서 influxdb가 명확하게 우위를 점했습니다.
    

[InfluxDB vs. Elasticsearch for Time Series Data & Metrics Benchmark](https://www.influxdata.com/blog/influxdb-markedly-elasticsearch-in-time-series-data-metrics-benchmark/)

References

- [https://docs.influxdata.com/influxdb/v2.0/reference/internals/shards/](https://docs.influxdata.com/influxdb/v2.0/reference/internals/shards/)
- [https://mangkyu.tistory.com/190](https://mangkyu.tistory.com/190)
- [https://opensource.com/article/17/8/influxdb-time-series-database-stack](https://opensource.com/article/17/8/influxdb-time-series-database-stack)
- [https://devconnected.com/the-definitive-guide-to-influxdb-in-2019/](https://devconnected.com/the-definitive-guide-to-influxdb-in-2019/)
- [https://medium.com/schkn/4-best-time-series-databases-to-watch-in-2019-ef1e89a72377](https://medium.com/schkn/4-best-time-series-databases-to-watch-in-2019-ef1e89a72377)
- [https://docs.influxdata.com/influxdb/v2.0/reference/glossary/](https://docs.influxdata.com/influxdb/v2.0/reference/glossary/)
- [https://docs.influxdata.com/telegraf/v1.20/](https://docs.influxdata.com/telegraf/v1.20/)
- [https://www.metricfire.com/blog/grafana-vs-chronograf-and-influxdb/](https://www.metricfire.com/blog/grafana-vs-chronograf-and-influxdb/)
- [https://docs.influxdata.com/telegraf/v1.20/plugins/](https://docs.influxdata.com/telegraf/v1.20/plugins/)

[Grafana vs. Chronograf and InfluxDB | MetricFire Blog](https://www.metricfire.com/blog/grafana-vs-chronograf-and-influxdb/)

[InfluxDB vs. Elasticsearch for Time Series Data & Metrics Benchmark](https://www.influxdata.com/blog/influxdb-markedly-elasticsearch-in-time-series-data-metrics-benchmark/)
