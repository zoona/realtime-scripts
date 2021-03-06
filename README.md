
# Flume Practice

## 1. Flume 설치

### 다운로드

```bash
wget http://apache.tt.co.kr/flume/1.6.0/apache-flume-1.6.0-bin.tar.gz
```

### 압축해제

```bash
tar xvfz apache-flume-1.6.0-bin.tar.gz
```

## 2. Agent Configuration

### 예제 1 (`spoolDir source` - `memoryChannel` - `file_roll sink`)

- spoolDir Source로 /tmp/spool 디렉토리 감시 후 생성 파일 수집
- memory channel에 저장
- fileRoll Sink로 /tmp/fileroll에 저장

```
# case01.conf
agent.sources = spoolDirSource
agent.channels = memoryChannel
agent.sinks = fileRollSink

agent.channels.memoryChannel.type = memory
agent.channels.memoryChannel.capacity = 1000

agent.sources.spoolDirSource.type = spoolDir
agent.sources.spoolDirSource.spoolDir = /tmp/spool
agent.sources.spoolDirSource.channels = memoryChannel

agent.sinks.fileRollSink.type = file_roll
agent.sinks.fileRollSink.sink.directory = /tmp/fileroll
agent.sinks.fileRollSink.channel = memoryChannel
```

디렉토리 생성

```bash
mkdir /tmp/fileroll
mkdir /tmp/spool
```

agent 실행

```bash
bin/flume-ng agent \
  --conf-file=case01.conf \
  --name agent
```

data 생성

```bash
ls -al /tmp/fileroll
echo hello > /tmp/spool/hello.txt
echo bye > /tmp/spool/bye.txt
```

결과 확인

```bash
ls -al /tmp/spool
```

```bash
ls -al /tmp/fileroll
```

```bash
cat /tmp/fileroll/`filename`
```

### 예제 2 (`exec source` - `fileChannel` - `hdfs sink`)

- tail로 buffer 파일의 내용을 수집
- file channel에 저장
- hdfs에 저장

```
# case02.conf
agent.sources = execSource
agent.channels = fileChannel
agent.sinks = hdfsSink

agent.sources.execSource.type = exec
agent.sources.execSource.command = tail -f /tmp/buffer
agent.sources.execSource.batchSize = 5
agent.sources.execSource.channels = fileChannel
agent.sources.execSource.interceptors = timestampInterceptor
agent.sources.execSource.interceptors.timestampInterceptor.type = timestamp

agent.sinks.hdfsSink.type = hdfs
agent.sinks.hdfsSink.hdfs.path = hdfs://Tom:9000/flume/%Y%m%d-%H%M%S
agent.sinks.hdfsSink.hdfs.fileType = DataStream
agent.sinks.hdfsSink.hdfs.writeFormat = Text
agent.sinks.hdfsSink.channel = fileChannel

agent.channels.fileChannel.type = file
agent.channels.fileChannel.checkpointDir = /tmp/flume/checkpoint
agent.channels.fileChannel.dataDirs = /tmp/flume/data
```

버퍼 파일 생성

```bash
touch /tmp/buffer
```

hdfs 디렉토리 생성

```bash
su - hdfs
hadoop fs -mkdir /flume
hadoop fs -chmod 777 /flume
exit
```
agent 실행

```bash
bin/flume-ng agent \
  --conf-file=case02.conf \
  --name agent
```

데이터 입력

```bash
echo hello >> /tmp/buffer
```

결과 확인

```bash
hadoop fs -ls /flume
hadoop fs -ls /flume/'directory name'
hadoop fs -cat /flume/'directory name'/'file name'
```
----

# Kafka Practice

## 1. Kafka 설치

### 다운로드
```bash
wget http://apache.tt.co.kr/kafka/0.8.2.2/kafka_2.10-0.8.2.2.tgz
```

### 압축해제
```bash
tar -xvzf kafka_2.10-0.8.2.2.tgz
cd kafka_2.10-0.8.2.2
```

### zookeeper 실행

```bash
bin/zookeeper-server-start.sh config/zookeeper.properties
```

### kafka server 실행
```bash
bin/kafka-server-start.sh config/server.properties
```

## 2. Topic

### create

```bash
bin/kafka-topics.sh \
  --create \
  --zookeeper localhost:2181 \
  --replication-factor 1 \
  --partitions 1 \
  --topic test
```

### list

```bash
bin/kafka-topics.sh \
  --list \
  --zookeeper localhost:2181
```

### describe

```bash
bin/kafka-topics.sh \
  --describe \
  --topic test \
  --zookeeper localhost:2181
```

## 3. Producer

### send message

```bash
bin/kafka-console-consumer.sh \
  --zookeeper localhost:2181 \
  --topic test \
  --from-beginning
```

## 4. Consumer

### consume

```bash
bin/kafka-console-producer.sh \
  --broker-list localhost:9092 \
  --topic test
```

### create with replication num 2

```bash

```

### create with partition num 3

```bash

```

## 5. Multi Broker

### config 수정

```bash
cp config/server.properties config/server-1.properties
cp config/server.properties config/server-2.properties
```

```
config/server-1.properties:
    broker.id=1
    port=9093
    log.dir=/tmp/kafka-logs-1
    zookeeper.connect=localhost:2181

config/server-2.properties:
    broker.id=2
    port=9094
    log.dir=/tmp/kafka-logs-2
    zookeeper.connect=localhost:2181
```

### start node

```
bin/kafka-server-start.sh config/server-1.properties
bin/kafka-server-start.sh config/server-2.properties
```

### create topic

```bash
bin/kafka-topics.sh \
  --create \
  --zookeeper localhost:2181 \
  --replication-factor 3 \
  --partitions 1 \
  --topic my-replicated-topic
```

### describe topic

```bash
bin/kafka-topics.sh \
  --describe \
  --zookeeper localhost:2181 \
  --topic my-replicated-topic
```

## 5. fault-tolerance

### kill process

```bash
ps -ef | grep server-1

kill `process id`
```

### describe topic

```bash
bin/kafka-topics.sh \
  --describe \
  --zookeeper localhost:2181 \
  --topic my-replicated-topic
```

### consume

```bash
bin/kafka-console-consumer.sh \
  --zookeeper localhost:2181 \
  --topic my-replicated-topic \
  --from-beginning
```

## 6. Monitoring

### Kafka Tools

ConsumerOffsetChecker

```bash
bin/kafka-run-class.sh \
  kafka.tools.ConsumerOffsetChecker \
  --broker-info localhost:9092 \
  --zookeeper localhost:2181 \
  --group `group name`
```

GetOffsetShell

```bash
bin/kafka-run-class.sh \
  kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic test \
  --time -2
```

SimpleConsumerShell

```bash
bin/kafka-run-class.sh \
  kafka.tools.SimpleConsumerShell \
  --broker-list localhost:9092 \
  --topic test \
  --partition 0
```

### quantifind/KafkaOffsetMonitor

[https://github.com/quantifind/KafkaOffsetMonitor](https://github.com/quantifind/KafkaOffsetMonitor)

### 다운로드

```bash
wget https://github.com/quantifind/KafkaOffsetMonitor/releases/download/v0.2.1/KafkaOffsetMonitor-assembly-0.2.1.jar
```

### 실행

```bash
java -cp KafkaOffsetMonitor-assembly-0.2.1.jar \
  com.quantifind.kafka.offsetapp.OffsetGetterWeb \
  --zk localhost:2181 \
  --port 9991 \
  --refresh 10.seconds \
  --retain 6.hours
```
----

# Storm

## Storm 설치

### 다운로드
```bash
wget http://apache.tt.co.kr/storm/apache-storm-0.10.0-beta1/apache-storm-0.10.0-beta1.tar.gz
```

### 압축해제
```bash
tar -xvzf apache-storm-0.10.0-beta1.tar.gz
cd apache-storm-0.10.0-beta1
```

### yaml 수정

```bash
cd conf
vi storm.yaml
```

```yaml
########### These MUST be filled in for a storm configuration
storm.zookeeper.servers:
    - "localhost"
#     - "server2"
#
nimbus.host: "localhost"
#
#
```

### Nimbus, UI 실행

```bash
# on master
bin/storm nimbus
bin/storm ui
```

### Supervisor 실행

```bash
# on slave
bin/storm Supervisor
```

### WebUI

```bash
# on browser
http://localhost:8080
```

----

# Redis

## Redis 설치

### 다운로드

```bash
wget http://download.redis.io/releases/redis-3.0.3.tar.gz
```

### 압축해제

```bash
tar xvfz redis-3.0.3.tar.gz
```

### 컴파일

```bash
cd redis-3.0.3
```

```bash
make
```
### conf 수정

```bash
################################ GENERAL  #####################################

# By default Redis does not run as a daemon. Use 'yes' if you need it.
# Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
daemonize yes
```

### server 실행

```bash
src/redis-server redis.conf
```

### client 실행

```bash
src/redis-cli
```
