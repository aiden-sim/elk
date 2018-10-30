## 1) 전체 흐름도
![logstach-archi2](https://user-images.githubusercontent.com/7076334/47731885-63190280-dca8-11e8-953b-943d393c1f12.png)

- 앞단에 redis를 사용하는 ELK 구성으로 정리해 보았다.
- 예제는 windows로.. (귀찮아서)
- 그리고 logbak 환경은 spring으로..

## 2) logback => redis 연동
![image2018-10-26_18-43-10](https://user-images.githubusercontent.com/7076334/47732312-4af5b300-dca9-11e8-9437-279be0d1de81.png)

### 2-1) redis 설치 및 실행
- Windows 버전은 별다르게 설치가 필요 없다. 압축을 풀고 실행시키면 된다
```
$REDIS_HOME$/redis-server
```
- 해당 명령어로 실행 시키면 기본 설정으로 redis를 실행 시킬 수 있다.
![image2018-10-26_16-17-25](https://user-images.githubusercontent.com/7076334/47732699-16362b80-dcaa-11e8-93fb-2642937ca16d.png)

- 기타 다른 명령어들
```
> redis-server redis.windows.conf	redis 설정파일을 이용해서 실행
> redis-server --service-start	redis 서비스 시작
> redis-server --service-stop	redis 서비스 종료
```

### 2-2) logback과 연동
- 프로젝트의 logback을 통해서 Redis에 데이터를 전송할 수 있다.


### web.xml
- 주의 점은 web.xml에 logback-redis-appender 라이브러리가 꼭 추가되어 있어야 한다.. 해당 라이브러리가 정상적으로 추가되지 않으면 redis로 전송이 되지 않을 뿐만 아니라 별도의 에러가 나지 않아서 원인을 찾기도 쉽지 않다.
```
web.xml
<!-- logstash -->
<dependency>
    <groupId>com.cwbase</groupId>
    <artifactId>logback-redis-appender</artifactId>
    <version>1.1.5</version>
</dependency>
```

### logback.xml
- logback-redis-appender 라이브러리 추가 후 redis관련 설정을 logback에 추가해준다.

```
logback.xml
<appender name="redis" class="com.cwbase.logback.RedisAppender">
    <host>127.0.01</host>
    <port>6379</port>
    <key>tomcat</key>
    <type>simjunbo</type>
    <source>local</source>
</appender>
<appender name="redis-logstash" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="redis"/>
</appender>
<logger name="access_log" level="INFO" additivity="false">
    <appender-ref ref="console"/>
    <appender-ref ref="FILE"/>
    <appender-ref ref="redis-logstash"/>
</logger>
<logger name="com.sjb" level="DEBUG" additivity="false">
    <appender-ref ref="console"/>
    <appender-ref ref="FILE"/>
    <appender-ref ref="redis-logstash"/>
</logger>
<root level="WARN">
    <appender-ref ref="console"/>
    <appender-ref ref="FILE"/>
    <appender-ref ref="redis-logstash"/>
</root>
```

- 주요 정보로는 다음과 같다.
```
host,port   | 정보를 전송할 redis 주소          
key         | redis 사용하는 key               
type,source | logstash 설정파일에서 field로 사용 
```

### 2-3) Redis 데이터 확인
- 실제 request를 발생 시켜서 logback을 통해 redis에 정상적으로 저장되었는지 확인해 본다.
- redis-cli를 이용해서 접속해서 확인가능합니다. 또는 FastoRedis 같은 GUI 툴을 사용 ( 나는 FastoRedis 이용 )
```
$REDIS_HOME$/redis-cli
```

- redis에 데이터가 List 형태로 쌓이기 때문에 [GET KEY] 형태로 조회 하면 출력이 되지 않는다.
- [LRANGE KEY START STOP] 형태로 범위로 조회 한다.
- 나는 logback에 key를 tomcat으로 설정했기 때문에 LRANGE tomcat 0 3 으로 조회를 했고 정상적으로 데이터가 들어간 것을 확인했다.

![image2018-10-26_17-42-7](https://user-images.githubusercontent.com/7076334/47733458-9c9f3d00-dcab-11e8-97f1-ef0ff546edd3.png)

- redis에 저장된 데이터 형태는 대략 다음과 같고 source, type 같은 컬럼은 logstash에서 사용된다.

```
{
  "source": "local",
  "host": "SIMJUNBO",
  "path": null,
  "type": "simjunbo",
  "tags": [],
  "message": "[200][GET][http://localhost:8080/api/test][0:0:0:0:0:0:0:1][message={test}][accept-language={ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7}&host={localhost:8080}&upgrade-insecure-requests={1}&connection={keep-alive}&accept-encoding={gzip, deflate, br}&user-agent={Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36}&accept={text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8}][][4][test2]",
  "@timestamp": "2018-10-26T15:08:25.536+0900",
  "logger": "access_log",
  "level": "INFO",
  "thread": "http-nio-8080-exec-1"
}
```

## 3) redis => logstash 연동
![111](https://user-images.githubusercontent.com/7076334/47736246-a166ef80-dcb1-11e8-89f7-cb53bc60b268.png)

### 3-1) logstash 설치
- logstash도 Windows 버전은 별다르게 설치가 필요 없다. 압축만 풀어주면 된다.
- input(redis)와 ouput(elasticsearch) 에 대한 설정이 필요하기 때문에 해당 위치에 logstash-tomcat.conf 라고 파일을 하나 생성했다.
```
$LOGSTASH_HOME$/bin/logstash-tomcat.conf
```

### 3-2) 환경파일 구성
- 우리가 logback에서 설정했던 type, source등이 여기서 사용됩니다.
```
logstash-tomcat.conf
input {
     redis {
      codec => json
      host => "127.0.0.1"
      port => 6379
      key => "tomcat"
      data_type => "list"
     }
     stdin {
        codec => json
     }  
}
filter {
    ruby {
        code => "event['path'] = 'remove'"
        remove_field => [ "@version","path"]
    }
    if [logger] == "access_log" {
        if [type] == "simjunbo" {
            grok{
                match => {"message" => "\[%{NUMBER:status}\]\[%{WORD:method}\]\[%{URI:request}\]\[%{IP:ip}\]\[%{DATA:userId}\]\[%{DATA:authorities}\]\[%{DATA:params}\]\[%{DATA:browser}\]\[%{DATA:os}\]"}
            }
        }
    }
}
output {
    elasticsearch {
        hosts => ["127.0.0.1:9200"]
        index => "log-tomcat-simjunbo-%{+YYYY.MM}"
        template => "C:\elk\logstash\template\template-tomcat.json"
        template_name => "tomcat"
        template_overwrite => true
        }
        file {
            path => "C:\elk\logstash\log\log-tomcat-simjunbo-%{source}-%{+YYYY.MM.dd}"
        }
        stdout {
            codec => rubydebug { }
        }
    }      
}
```

- input은 실제 데이터를 받아오는 부분이다. 예제는 redis와 연동되어 있기 때문에 redis 정보가 입력되어 있다.
- filter는 위에서 저장 된, redis 정보를 받아와서 elasticsearch에 적재 하기 전에 정규표현식을 이용해서 데이터를 재 가공한다. 우리가 kibana에서 보는 method, params, request 등 항목들도 logstash의 정규화를 통해서 생성된다.
- output은 filter에서 가공한 데이터를 보내는 부분이다. 예제는 elasticsearch에 연동되어 있기 때문에 elasticsearch 정보가 입력되어 있다.
- 그리고 output에서 가장 중요한 부분이 index 인데 서버 수집로그는 log-tomcat-simjunbo-2018.10 형태로 되어 있다.
- stdin, stdout 은 로컬에서 연동테스트 하면서 추가한 것이다. stdin은 실제로 redis가 아닌 콘솔창에 데이터를 입력해볼 수 있으며 stdout은 입력된 데이터에 대해 정상처리 시, 콘솔에 출력된다.


- output에서 설정되어 있는 template 부분이다. template를 사용하면 사전에 type mapping을 미리 선언하지 않고 패턴이나 특성에 맞춰서 구성할 수 있다.
```
template-tomcat.json
{
    "template" : "log-tomcat*" ,
    "settings" : {
        "number_of_shards" : "1",
        "number_of_replicas" : "0"
    },
    "mappings" : {
            "simjunbo" : {
                 "properties" : {
                   "responseTime" : {"type" : "number"}
                 }
            }
    }
}
```

### 3-3) logstash 실행
- 해당 명령으로 설정파일을 이용해서 기동하면 된다.
```
$LOGSTASH_HOME$/bin/logstash.bat -f logstash-tomcat.conf
```

- 정상적으로 기동 되면 다음과 같은 화면이 출력된다.

![image2018-10-26_18-42-16](https://user-images.githubusercontent.com/7076334/47734499-c48fa000-dcad-11e8-9c8f-98cb7cb4c1b6.png)

※ 참고로 redis와 elasticsearch가 먼저 기동되어 있어야 한다.

## 4) logstash => elasticsearch 연동
![222](https://user-images.githubusercontent.com/7076334/47736244-a0ce5900-dcb1-11e8-8fa6-512bd5f3360e.png)

### 4-1) elasticsearch 설치
- elasticsearch Windows 버전도 별다르게 설치가 필요 없다. 압축을 풀고 실행시키면 된다.

### 4-2) 환경파일 구성
- 분산 환경 설정으로 하면 좀더 복잡한 설정이 필요하지만 일단 단일 환경으로 구성했다.
- elasticsearch 파일에서 간단하게 cluser.name과 node.name만 설정 후 바로 기동했다.
```
$ELASTICSEARCH$/config/elasticsearch.yml
```

### elasticsearch.yml

![image2018-10-26_19-0-8](https://user-images.githubusercontent.com/7076334/47734704-31a33580-dcae-11e8-81df-3149ac83db5d.png)


### 4-3) elasticsearch 실행

해당 명령으로 기동하면 된다.
```
$ELASTIC_SEARCH$/bin/elasticsearch
```

- 정상적으로 기동 되면 다음과 같은 화면이 출력된다.
![image2018-10-26_19-7-15](https://user-images.githubusercontent.com/7076334/47734758-4f709a80-dcae-11e8-8130-b51c7d7891ee.png)


### 4-4) elasticsearch 데이터 확인

- curl 프로그램을 이용해서 logstash에 정상적으로 index의 생성여부를 확인할 수 있다.
```
curl -XGET 127.0.0.1:9200/[indexkey]?pretty
```

- 내 로컬에서는 index를 log-tomcat-simjunbo-2018.10 으로 생성했기 때문에 curl -XGET 127.0.0.1:9200/log-tomcat-simjunbo-2018.10?pretty 로 조회 했다..
- 뒤에 pretty 명령어를 붙여주면 json 형태로 정렬되서 보여줍니다.

```
조회결과
{
  "log-tomcat-simjunbo-2018.10" : {
    "aliases" : { },
    "mappings" : {
      "simjunbo" : {
        "properties" : {
          "@timestamp" : {
            "type" : "date",
            "format" : "strict_date_optional_time||epoch_millis"
          },
          "headers" : {
            "type" : "string"
          },
          "host" : {
            "type" : "string"
          },
          "ip" : {
            "type" : "string"
          },
          "level" : {
            "type" : "string"
          },
          "logger" : {
            "type" : "string"
          },
          "message" : {
            "type" : "string"
          },
          "method" : {
            "type" : "string"
          },
          "params" : {
            "type" : "string"
          },
          "port" : {
            "type" : "string"
          },
          "request" : {
            "type" : "string"
          },
          "response" : {
            "type" : "string"
          },
          "responseTime" : {
            "type" : "long"
          },
          "source" : {
            "type" : "string"
          },
          "status" : {
            "type" : "long"
          },
          "tags" : {
            "type" : "string"
          },
          "thread" : {
            "type" : "string"
          },
          "type" : {
            "type" : "string"
          }
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1540523115581",
        "number_of_shards" : "1",
        "number_of_replicas" : "0",
        "uuid" : "_ABhy15OQre15ETTOqmdQQ",
        "version" : {
          "created" : "2020099"
        }
      }
    },
    "warmers" : { }
  }
}
```

### 5) elasticsearch => kibana 
![333](https://user-images.githubusercontent.com/7076334/47736245-a0ce5900-dcb1-11e8-8e84-bf31973dbf1a.png)

## 5-1) kibana 설치 
- Windows 버전은 별다르게 설치가 필요 없다. 압축을 풀고 실행시키면 된다.

## 5-2) 환경파일 구성
- elasticsearch에서 수집한 데이터를 시각화 해야 되기 때문에 elasticsearch 접근 url만 설정해 주었다.
```
$KIBANA_HOME$/bin/kibana/config/kibana.yml
```

### kibana.yml
![image2018-10-29_11-19-11](https://user-images.githubusercontent.com/7076334/47735506-fb66b580-dcaf-11e8-975b-094a4bb6fd36.png)


## 5-3) kibana 실행
- 해당 명령으로 기동하면 된다.
```
$KIBANA_HOME$/bin/kibana
```

- 정상적으로 기동 되면 다음과 같은 화면이 출력된다.

![image2018-10-29_11-3-24](https://user-images.githubusercontent.com/7076334/47735007-e63d5700-dcae-11e8-8b54-f90afc13c57a.png)

- 기동 후, kibana.yml 파일에 설정한 port로 접속하면 우리가 익숙하게 사용하고 있는 kibana 화면이 나온다.
- 하지만 현재는 아무런 index를 등록하지 않았기 때문에 데이터가 노출되지 않는다.

![image2018-10-29_11-24-0](https://user-images.githubusercontent.com/7076334/47735068-0ec55100-dcaf-11e8-9c1d-ee1ca6ace1dc.png)

## 5-4) index pattern 설정

- kibana > settings에서 elasticsearch에 등록된 index pattern을 설정해 준다.
- 나는 테스트 index로 log-tomcat-simjunbo-2018.10 를 사용했기 때문에 log-tomcat-* 으로 등록 시켰다.

![image2018-10-29_11-27-36](https://user-images.githubusercontent.com/7076334/47735095-2270b780-dcaf-11e8-9edb-cea35bf8b3e0.png)

## 5-5) kibana 데이터 확인
- 실제 로컬에서 test request 발생 후, kibana에서 정상적으로 조회까지 확인해 보았다.

![image2018-10-29_11-31-38](https://user-images.githubusercontent.com/7076334/47735171-521fbf80-dcaf-11e8-9e38-01fd33628cc9.png)


## 기동순서
- redis
- elasticsearch
- logstash
- kibana

