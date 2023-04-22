# Error handling of data in Connect

Options:  
```
#"errors.tolerance": "none", # default 
#"errors.tolerance": "all",
#"errors.deadletterqueue.topic.name":"dlq_file_sink_02_header_log_mix",
#"errors.deadletterqueue.topic.replication.factor": 1,
#"errors.deadletterqueue.context.headers.enable":true,
#"errors.log.enable":true,
#"errors.log.include.messages":true
```
Make sure you create file `/tmp/data/file_sink_01.txt`  


## errors.tolerance = none

Fail right away 

```
## valid 
for i in `seq 200`; do echo '{"id": "key'$i'"}';done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json

## fail the connector 
for i in `seq 200`; do echo asdas$i;done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json
```

```
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_01",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file":"/tmp/data/file_sink_01.txt"
                }
        }'

curl -X GET http://localhost:8083/connectors/file_sink_01/status      

...
...
Caused by: org.apache.kafka.common.errors.SerializationException: Error deserializing message to JSON in topic test_topic_json
```


## errors.tolerance = all
Do not fail , do not record.  

```
curl -X DELETE http://localhost:8083/connectors/file_sink_01

## valid 
for i in `seq 200`; do echo '{"id": "key'$i'"}';done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all
## fail the connector 
for i in `seq 200`; do echo asdas$i;done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all



curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_01",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json_t_all",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file":"/tmp/data/file_sink_01.txt",
                "errors.tolerance": "all"
                }
        }'

curl -X GET http://localhost:8083/connectors/file_sink_01/status | jq                  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   112  100   112    0     0  12985      0 --:--:-- --:--:-- --:--:-- 56000
{
  "name": "file_sink_01",
  "connector": {
    "state": "RUNNING",
    "worker_id": "192.168.1.19:8083"
  },
  "tasks": [],
  "type": "sink"
}
```


## errors.tolerance = all, with DLQ , no header 
Do not fail , do record the message.  


```
curl -X DELETE http://localhost:8083/connectors/file_sink_01

## valid 
for i in `seq 200`; do echo '{"id": "key'$i'"}';done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_withdlq
## fail the connector 
for i in `seq 200`; do echo asdas$i;done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_withdlq



curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_01",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json_t_all_withdlq",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file":"/tmp/data/file_sink_01.txt",
                "errors.tolerance": "all",
                "errors.deadletterqueue.topic.name":"dlq_file_sink_02",
                "errors.deadletterqueue.topic.replication.factor": 1
                }
        }'

curl -X GET http://localhost:8083/connectors/file_sink_01/status | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   170  100   170    0     0  17118      0 --:--:-- --:--:-- --:--:-- 56666
{
  "name": "file_sink_01",
  "connector": {
    "state": "RUNNING",
    "worker_id": "192.168.1.19:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "192.168.1.19:8083"
    }
  ],
  "type": "sink"
}
```

check DLQ topic  

```
kafka-console-consumer --from-beginning -bootstrap-server localhost:9092 --topic dlq_file_sink_02 
asdas1
..
asdas200
```

## errors.tolerance = all, with DLQ , with header 

Do not fail , do record the message, include error in header in DLQ topic.    

```
curl -X DELETE http://localhost:8083/connectors/file_sink_01

## valid 
for i in `seq 200`; do echo '{"id": "key'$i'"}';done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_withdlq_header
## fail the connector 
for i in `seq 200`; do echo asdas$i;done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_withdlq_header


curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_01",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json_t_all_withdlq_header",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file":"/tmp/data/file_sink_01.txt",
                "errors.tolerance": "all",
                "errors.deadletterqueue.topic.name":"dlq_file_sink_02_header",
                "errors.deadletterqueue.topic.replication.factor": 1,
                "errors.deadletterqueue.context.headers.enable":true
                }
        }'

curl -X GET http://localhost:8083/connectors/file_sink_01/status | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   170  100   170    0     0  24113      0 --:--:-- --:--:-- --:--:--  166k
{
  "name": "file_sink_01",
  "connector": {
    "state": "RUNNING",
    "worker_id": "192.168.1.19:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "192.168.1.19:8083"
    }
  ],
  "type": "sink"
}

## check DLQ topic 
kafka-console-consumer --from-beginning -bootstrap-server localhost:9092 --property print.headers=true --topic dlq_file_sink_02_header

__connect.errors.topic:test_topic_json_t_all_withdlq_header,__connect.errors.partition:0,__connect.errors.offset:200,__connect.errors.connector.name:file_sink_01,__connect.errors.task.id:0,__connect.errors.stage:VALUE_CONVERTER,__connect.errors.class.name:org.apache.kafka.connect.json.JsonConverter,__connect.errors.exception.class.name:org.apache.kafka.connect.errors.DataException,__connect.errors.exception.message:Converting byte[] to Kafka Connect data failed due to serialization error: ,__connect.errors.exception.stacktrace:org.apache.kafka.connect.errors.DataException: Converting byte[] to Kafka Connect data failed due to serialization error: 
        at org.apache.kafka.connect.json.JsonConverter.toConnectData(JsonConverter.java:324)
        at org.apache.kafka.connect.storage.Converter.toConnectData(Converter.java:88)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.lambda$convertAndTransformRecord$5(WorkerSinkTask.java:519)
        at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execAndRetry(RetryWithToleranceOperator.java:183)
        at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execAndHandleError(RetryWithToleranceOperator.java:217)
        at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execute(RetryWithToleranceOperator.java:159)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.convertAndTransformRecord(WorkerSinkTask.java:519)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.convertMessages(WorkerSinkTask.java:494)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.poll(WorkerSinkTask.java:333)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.iteration(WorkerSinkTask.java:235)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.execute(WorkerSinkTask.java:204)
        at org.apache.kafka.connect.runtime.WorkerTask.doRun(WorkerTask.java:200)
        at org.apache.kafka.connect.runtime.WorkerTask.run(WorkerTask.java:255)
        at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
        at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
        at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
        at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
        at java.base/java.lang.Thread.run(Thread.java:829)
Caused by: org.apache.kafka.common.errors.SerializationException: Error deserializing message to JSON in topic test_topic_json_t_all_withdlq_header
        at org.apache.kafka.connect.json.JsonDeserializer.deserialize(JsonDeserializer.java:66)
        at org.apache.kafka.connect.json.JsonConverter.toConnectData(JsonConverter.java:322)
        ... 17 more
        asdas1
```


## errors.tolerance = all, with log
Do not fail , do record the error in Connect log.

```
curl -X DELETE http://localhost:8083/connectors/file_sink_01

## valid 
for i in `seq 200`; do echo '{"id": "key'$i'"}';done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_log
## fail the connector 
for i in `seq 200`; do echo asdas$i;done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_log



curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_01",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json_t_all_log",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file":"/tmp/data/file_sink_01.txt",
                "errors.tolerance": "all",
                "errors.log.enable":true,
                "errors.log.include.messages":true
                }
        }'

curl -X GET http://localhost:8083/connectors/file_sink_01/status | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   170  100   170    0     0  20274      0 --:--:-- --:--:-- --:--:-- 85000
{
  "name": "file_sink_01",
  "connector": {
    "state": "RUNNING",
    "worker_id": "192.168.1.19:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "192.168.1.19:8083"
    }
  ],
  "type": "sink"
}

## check Connect log

[2023-04-22 16:36:10,639] ERROR [file_sink_01|task-0] Error encountered in task file_sink_01-0. Executing stage 'VALUE_CONVERTER' with class 'org.apache.kafka.connect.json.JsonConverter', where consumed record is {topic='test_topic_json_t_all_log', partition=0, offset=399, timestamp=1682170563857, timestampType=CreateTime}. (org.apache.kafka.connect.runtime.errors.LogReporter:66)
org.apache.kafka.connect.errors.DataException: Converting byte[] to Kafka Connect data failed due to serialization error: 
```

## errors.tolerance = all, with DLQ , with header and log
Do not fail , do record the error in Connect log, include error in header in DLQ topic.    


```
curl -X DELETE http://localhost:8083/connectors/file_sink_01

## valid 
for i in `seq 200`; do echo '{"id": "key'$i'"}';done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_withdlq_header_withlog
## fail the connector 
for i in `seq 200`; do echo asdas$i;done | kafka-console-producer --broker-list localhost:9092 --topic test_topic_json_t_all_withdlq_header_withlog



curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_01",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json_t_all_withdlq_header_withlog",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file":"/tmp/data/file_sink_01.txt",
                "errors.tolerance": "all",
                "errors.deadletterqueue.topic.name":"dlq_file_sink_02_header_log_mix",
                "errors.deadletterqueue.topic.replication.factor": 1,
                "errors.deadletterqueue.context.headers.enable":true,
                "errors.log.enable":true,
                "errors.log.include.messages":true
                }
        }'

curl -X GET http://localhost:8083/connectors/file_sink_01/status | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   170  100   170    0     0  24113      0 --:--:-- --:--:-- --:--:--  166k
{
  "name": "file_sink_01",
  "connector": {
    "state": "RUNNING",
    "worker_id": "192.168.1.19:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "192.168.1.19:8083"
    }
  ],
  "type": "sink"
}

## check DLQ topic 
kafka-console-consumer --from-beginning -bootstrap-server localhost:9092 --property print.headers=true --topic dlq_file_sink_02_header_log_mix


__connect.errors.topic:test_topic_json_t_all_withdlq_header_withlog,__connect.errors.partition:0,__connect.errors.offset:799,__connect.errors.connector.name:file_sink_01,__connect.errors.task.id:0,__connect.errors.stage:VALUE_CONVERTER,__connect.errors.class.name:org.apache.kafka.connect.json.JsonConverter,__connect.errors.exception.class.name:org.apache.kafka.connect.errors.DataException,__connect.errors.exception.message:Converting byte[] to Kafka Connect data failed due to serialization error: ,__connect.errors.exception.stacktrace:org.apache.kafka.connect.errors.DataException: Converting byte[] to Kafka Connect data failed due to serialization error: 
        at org.apache.kafka.connect.json.JsonConverter.toConnectData(JsonConverter.java:324)
        at org.apache.kafka.connect.storage.Converter.toConnectData(Converter.java:88)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.lambda$convertAndTransformRecord$5(WorkerSinkTask.java:519)
        at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execAndRetry(RetryWithToleranceOperator.java:183)
        at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execAndHandleError(RetryWithToleranceOperator.java:217)
        at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execute(RetryWithToleranceOperator.java:159)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.convertAndTransformRecord(WorkerSinkTask.java:519)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.convertMessages(WorkerSinkTask.java:494)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.poll(WorkerSinkTask.java:333)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.iteration(WorkerSinkTask.java:235)
        at org.apache.kafka.connect.runtime.WorkerSinkTask.execute(WorkerSinkTask.java:204)
        at org.apache.kafka.connect.runtime.WorkerTask.doRun(WorkerTask.java:200)
        at org.apache.kafka.connect.runtime.WorkerTask.run(WorkerTask.java:255)
        at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
        at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
        at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
        at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
        at java.base/java.lang.Thread.run(Thread.java:829)
Caused by: org.apache.kafka.common.errors.SerializationException: Error deserializing message to JSON in topic test_topic_json_t_all_withdlq_header_withlog
        at org.apache.kafka.connect.json.JsonDeserializer.deserialize(JsonDeserializer.java:66)
        at org.apache.kafka.connect.json.JsonConverter.toConnectData(JsonConverter.java:322)
        ... 17 more
        asdas200



## Connect log: 

kafka-console-consumer --from-beginning -bootstrap-server localhost:9092 --property print.headers=true --topic dlq_file_sink_02_header_log_mix


[2023-04-22 16:41:26,047] ERROR [file_sink_01|task-0] Error encountered in task file_sink_01-0. Executing stage 'VALUE_CONVERTER' with class 'org.apache.kafka.connect.json.JsonConverter', where consumed record is {topic='test_topic_json_t_all_withdlq_header_withlog', partition=0, offset=799, timestamp=1682170885998, timestampType=CreateTime}. (org.apache.kafka.connect.runtime.errors.LogReporter:66)
org.apache.kafka.connect.errors.DataException: Converting byte[] to Kafka Connect data failed due to serialization error: 
	at org.apache.kafka.connect.json.JsonConverter.toConnectData(JsonConverter.java:324)
	at org.apache.kafka.connect.storage.Converter.toConnectData(Converter.java:88)
	at org.apache.kafka.connect.runtime.WorkerSinkTask.lambda$convertAndTransformRecord$5(WorkerSinkTask.java:519)
	at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execAndRetry(RetryWithToleranceOperator.java:183)
	at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execAndHandleError(RetryWithToleranceOperator.java:217)
	at org.apache.kafka.connect.runtime.errors.RetryWithToleranceOperator.execute(RetryWithToleranceOperator.java:159)
	at org.apache.kafka.connect.runtime.WorkerSinkTask.convertAndTransformRecord(WorkerSinkTask.java:519)
	at org.apache.kafka.connect.runtime.WorkerSinkTask.convertMessages(WorkerSinkTask.java:494)
	at org.apache.kafka.connect.runtime.WorkerSinkTask.poll(WorkerSinkTask.java:333)
	at org.apache.kafka.connect.runtime.WorkerSinkTask.iteration(WorkerSinkTask.java:235)
	at org.apache.kafka.connect.runtime.WorkerSinkTask.execute(WorkerSinkTask.java:204)
	at org.apache.kafka.connect.runtime.WorkerTask.doRun(WorkerTask.java:200)
	at org.apache.kafka.connect.runtime.WorkerTask.run(WorkerTask.java:255)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
Caused by: org.apache.kafka.common.errors.SerializationException: Error deserializing message to JSON in topic test_topic_json_t_all_withdlq_header_withlog
	at org.apache.kafka.connect.json.JsonDeserializer.deserialize(JsonDeserializer.java:66)
	at org.apache.kafka.connect.json.JsonConverter.toConnectData(JsonConverter.java:322)
	... 17 more
```

