여러 CSV 파일을 Kafka 토픽으로 저장하기 Kafka Connect CSV 소스 커넥터와 Logstash를 활용할 수 있는데, CSV 소스 커넥터의 경우 토픽 수만큼의 커넥터가 필요한 반면 Logstash의 경우 복수 파이프라인 구성으로 가능

1. Kafka Connect Connector 사용
- CSV 소스 커넥터 사용 (kafka-connect-spooldir @ https://github.com/jcustenborder/kafka-connect-spooldir)
- Zookeeper와 Kafka를 각각 3개의 서버로 클러스터 구성
- Kafka Connect를 Kafka 브로커 서버에서 실행
- Schema-registry와 ksqlDB 서버를 도커 기반 컨테이너로 실행
- 여러 csv 파일을 별도의 토픽으로 저장하기 위해서는 복수의 소스 커넥터 실행 필요
- Kakfa 토픽으로 저장할 파일 - 헤더 정보가 없는 txt 파일, item1.txt
=========================================================
1,SHOP001,929,2018-10-01
2,SHOP001,480,2018-10-02
3,SHOP001,25,2018-10-03
=========================================================

- item1.txt 파일을 Kafka 토픽으로 저장하기 위해 Kafka Connect 소스 커넥터 실행
curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://localhost:8083/connectors -d @xxx.json
  : 헤더 미존재 및 카프카에서 스키마 자동 생성하는 경우
	- item1.json
=========================================================
{
  "name": "item1-csv",
  "config": {
    "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector",
    "topic": "item1_csv",			# 토픽명
    "input.path": "/data/unprocessed",		# item1.txt 파일이 존재하는 디렉토리
    "finished.path": "/data/processed",		# 처리 후 item1.txt 파일이 이동할 디렉토리
    "error.path": "/data/error",
    "input.file.pattern": "item1.txt",
    "schema.generation.enabled":"true"		# 스키마 자동 생성
  }
}
=========================================================

	- ksqlDB에서 확인, column01 ~ column04로 컬럼 생성
=========================================================
ksql> print item1_csv from beginning;
Key format: KAFKA_BIGINT or KAFKA_DOUBLE or KAFKA_STRING
Value format: AVRO or KAFKA_STRING
rowtime: 2022/01/17 03:56:43.745 Z, key: 6013557250951773053, value: {"column01": "1", "column02": "SHOP001", "column03": "929", "column04": "2018-10-01"}, partition: 0
rowtime: 2022/01/17 03:56:43.746 Z, key: 6013557250951773053, value: {"column01": "2", "column02": "SHOP001", "column03": "480", "column04": "2018-10-02"}, partition: 0
rowtime: 2022/01/17 03:56:43.746 Z, key: 6013557250951773053, value: {"column01": "3", "column02": "SHOP001", "column03": "25", "column04": "2018-10-03"}, partition: 0
=========================================================

스키마 확인
curl -X GET http://localhost:8081/subjects/item1_csv-value/versions/
curl -X GET http://localhost:8081/subjects/item1_csv-value/versions/1

  : 헤더 미존재 및 파일 스키마 설정하는 경우
	- 스키마에 설정 가능한 값 : [STRING, INT16, STRUCT, BOOLEAN, ARRAY, FLOAT64, BYTES, MAP, INT64, INT32, INT8, FLOAT32]
	- item1-sch.json
=========================================================
{
  "name": "item1-csv-sch "
  "config": {
    "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector",
    "topic": "item1_csv",
    "input.path": "/data/unprocessed",
    "finished.path": "/data/processed",
    "error.path": "/data/error",
    "input.file.pattern": "item1.txt",
    "csv.first.row.as.header": "false",
    "schema.generation.enabled":"true",
    "schema.generation.key.fields": "item_id",
    "schema.generation.key.name": "item_id",
    "key.schema": "{\"name\":\"com.github.jcustenborder.kafka.connect.model.Key\",\"type\":\"STRUCT\",\"isOptional\": false,\"fieldSchemas\":{\"item_id\":{\"type\":\"INT64\",\"isOptional\":false}}}",
    "value.schema": "{\"name\":\"com.github.jcustenborder.kafka.connect.model.Value\",\"type\":\"STRUCT\",\"isOptional\": false,\"fieldSchemas\":{\"item_id\": {\"type\": \"INT64\",\"isOptional\": true},\"shop_id\": {\"type\": \"STRING\",\"isOptional\": true},\"item_total\": {\"type\":\"INT64\",\"isOptional\": true},\"item_date\": {\"type\":\"STRING\",\"isOptional\": true}}}"
  }
}
=========================================================

	- ksqlDB에서 확인, 스키마에 맞게 컬럼 생성
=========================================================
ksql> print item1_csv_sch from beginning;
Key format: SESSION(KAFKA_STRING) or HOPPING(KAFKA_STRING) or TUMBLING(KAFKA_STRING) or KAFKA_STRING
Value format: AVRO
rowtime: 2022/01/17 04:09:05.161 Z, key: [S@7308602676550119805/8390898125761112436], value: {"item_id": 1, "shop_id": "SHOP001", "item_total": 929, "item_date": "2018-10-01"}, partition: 0
rowtime: 2022/01/17 04:09:05.163 Z, key: [S@7308602676550120061/8390898125761112436], value: {"item_id": 2, "shop_id": "SHOP001", "item_total": 480, "item_date": "2018-10-02"}, partition: 0
rowtime: 2022/01/17 04:09:05.163 Z, key: [S@7308602676550120317/8390898125761112436], value: {"item_id": 3, "shop_id": "SHOP001", "item_total": 25, "item_date": "2018-10-03"}, partition: 0
=========================================================

스키마 확인
curl -X GET http://localhost:8081/subjects/item1_csv_sch-value/versions/
curl -X GET http://localhost:8081/subjects/item1_csv_sch-value/versions/1

2. logstash 수집기 사용
- logstash를 도커 기반 컨테이너로 실행

docker run -it --net=host \
  -v /home/hadoop/data/csv-data/:/csv-data \
  -v /home/hadoop/data/logstash-settings:/usr/share/logstash/config \
  -v /home/hadoop/data/logstash-pipeline:/usr/share/logstash/pipeline \
  -e LS_JAVA_OPTS="-Xms256m -Xmx512m" \
  logstash:7.3.1 logstash

- 각 파일 처리를 위해 복수의 파이프라인을 구성하여 별도의 Kafka 토픽으로 저장
- 파일 처리 후 액션은은 file_completed_action 참조
@ https://www.elastic.co/guide/en/logstash/current/plugins-inputs-file.html#plugins-inputs-file-file_completed_action
- Kakfa 토픽으로 저장할 파일 - 헤더 정보가 없는 txt 파일, item1.txt
=========================================================
1,SHOP001,929,2018-10-01
2,SHOP001,480,2018-10-02
3,SHOP001,25,2018-10-03
=========================================================
- item2.txt 파일의 경우 dummy 값 추가

- 복수 파이프 라인 구성
  : /usr/share/logstash/config/pipelines.yml 설정
=========================================================
- pipeline.id: pipe1
  path.config: "/usr/share/logstash/pipeline/pipe1.conf"
- pipeline.id: pipe2
  path.config: "/usr/share/logstash/pipeline/pipe2.conf"
=========================================================

  : /usr/share/logstash/config/logstash.yml 기본 설정
=========================================================
http.host: "0.0.0.0"
path.config: /usr/share/logstash/pipeline
path.logs: /var/log/logstash
config.reload.automatic: true 
log.level: debug
xpack.monitoring.enabled: false
=========================================================

  : /usr/share/logstash/pipeline/pipe1.conf, 파일명에 따라 type 필터링 조건 부여
=========================================================
input {
  file {
    path => "/csv-data/test1.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    type => "test1"
    codec => plain {
      charset => "ISO-8859-1"
    }
  }
}
filter {
  if [type] == "test1" {
  csv {
    separator => ","
    columns => ["id","name","nums","mdate"]
    convert => {
      "id" => "integer"
      "name" => "integer"
      "nums" => "float"
      "mdate" => "date"
    }
  }
    mutate {
    remove_field => ["@version","@timestamp","path","host", "tags", "message"] # type 제거해야함
  }
  } 
}
output {
  if [type] == "test1" {
  kafka {
        bootstrap_servers => "master:9092,node1:9092,node2:9092"
        codec => json
        topic_id => "test1_topic"
      }
  }
  stdout {}
}
=========================================================

ksqlDB로 확인
=========================================================
ksql> print test1_topic from beginning;
Key format: ¯\_(ツ)_/¯ - no data processed
Value format: JSON or KAFKA_STRING
rowtime: 2022/01/17 07:27:07.799 Z, key: <null>, value: {"type":"test1","nums":929.1,"id":1,"mdate":"2021-10-01T00:00:00.000Z","name":1}, partition: 0
rowtime: 2022/01/17 07:27:07.828 Z, key: <null>, value: {"type":"test1","nums":480.5,"id":2,"mdate":"2021-10-02T00:00:00.000Z","name":1}, partition: 0
rowtime: 2022/01/17 07:27:07.848 Z, key: <null>, value: {"type":"test1","nums":25.0,"id":3,"mdate":"2021-10-03T00:00:00.000Z","name":1}, partition: 0
=========================================================
  : pipe2.conf 파일의 경우 type => "test2", 컬럼 타입 지정 삭제 

ksqlDB로 확인
=========================================================
ksql> print test2_topic from beginning;
Key format: ¯\_(ツ)_/¯ - no data processed
Value format: JSON or KAFKA_STRING
rowtime: 2022/01/17 07:27:08.143 Z, key: <null>, value: {"mdate":"2021-10-01","type":"test2","dummy":"2","nums":"929.1","id":"1","name":"1"}, partition: 0
rowtime: 2022/01/17 07:27:08.144 Z, key: <null>, value: {"mdate":"2021-10-02","type":"test2","dummy":"2","nums":"480.5","id":"2","name":"1"}, partition: 0
rowtime: 2022/01/17 07:27:08.144 Z, key: <null>, value: {"mdate":"2021-10-03","type":"test2","dummy":"2","nums":"25.0","id":"3","name":"1"}, partition: 0
rowtime: 2022/01/17 07:27:08.144 Z, key: <null>, value: {"mdate":"2021-10-04","type":"test2","dummy":"2","nums":"35.7","id":"4","name":"4"}, partition: 0
=========================================================
