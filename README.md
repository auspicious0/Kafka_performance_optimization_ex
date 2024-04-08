# kafka 성능 최적화 적용

## 1.	default configuration의 producer, consumer의 성능을 테스트 한다.  
```
./kafka-topics.sh --create --bootstrap-server 192.168.1.93:9091,192.168.1.93:9092,192.168.1.93:9093 --replication-factor 3 --partitions 10 --topic test

./kafka-producer-perf-test.sh --topic test --num-records 200000 --record-size 1000 --throughput -1 --producer-props bootstrap.servers=192.168.1.93:9092,192.168.1.93:9091,192.168.1.93:9093

./kafka-consumer-perf-test.sh --topic test --broker-list 192.168.1.93:9092,192.168.1.93:9091,192.168.1.93:9093 --messages 200000
```
## 2.	producer, consumer, broker tuning을 진행한다. 
broker

num.replica.fetchers=3 브로커 메시지 복제 스레드 수를 브로커 수에 맞춰줌

auto.create.topics.enable=false 토픽 자동생성을 방지함-> default로 토픽이 자동으로 생성될 수 있음

min.insync.replica=2 최소 리플리케이션 펙터를 줄여줌 -> 서버 다운을 견딜 수 있음

unclean.leader.election.enable=true 데이터 유실이 발생하더라도 카프카 서버 중지를 막음

```
num.replica.fetchers=3
auto.create.topics.enable=false
min.insync.replicas=1
unclean.leader.election.enable=true
```

consumer

fetch.max.bytes=104857600 한번에 가져올 수 있는 최대 데이터 지정 default(52428800)

max.partition.fetch.bytes=52428800 각 파티션에 대해 반환되는 데이터의 최대 제한(default 1048576)


```
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# see org.apache.kafka.clients.consumer.ConsumerConfig for more details

# list of brokers used for bootstrapping knowledge about the rest of the cluster
# format: host1:port1,host2:port2 ...
bootstrap.servers=192.168.1.93:9091,192.168.1.93:9092,192.168.1.93:9093
topic.metadata.refresh.interval.ms=60000
fetch.max.bytes=104857600
max.partition.fetch.bytes=52428800

# consumer group id
group.id=test-consumer-group

# What to do when there is no initial offset in Kafka or if the current
# offset does not exist any more on the server: latest, earliest, none
#auto.offset.reset=
```

producer
linger.ms 전송 전 대기 시간을 높여줌 -> 서버 부하 감소 (default 0)

batch.size 한 번에 보내는 양을 늘려줌 -> 서버 부하 감소(default 16kb)

enable.idempotence=false 멱등성 동일한 데이터 1번만 저장 default -> 특성상 맞지 않음

max.in.flight.requests.per.connection=1000000 한 번에 몇 개의 요청을 전송할 것인가

acks=0 서버로부터 ack를 기다리지 않음->속도 향상 
 
```
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# see org.apache.kafka.clients.producer.ProducerConfig for more details

############################# Producer Basics #############################

# list of brokers used for bootstrapping knowledge about the rest of the cluster
# format: host1:port1,host2:port2 ...
bootstrap.servers=192.168.1.93:9091,192.168.1.93:9092,192.168.1.93:9093

# specify the compression codec for all data generated: none, gzip, snappy, lz4, zstd
compression.type=none

# name of the partitioner class for partitioning records;
# The default uses "sticky" partitioning logic which spreads the load evenly between partitions, but improves throughput by attempting to fill the batches sent to each partition.
#partitioner.class=

# the maximum amount of time the client will wait for the response of a request
#request.timeout.ms=

# how long `KafkaProducer.send` and `KafkaProducer.partitionsFor` will block for
#max.block.ms=1000

# the producer will wait for up to the given delay to allow other records to be sent so that the sends can be batched together
linger.ms=100

# the maximum size of a request in bytes
#max.request.size=

# the default batch size in bytes when batching multiple records sent to a partition
batch.size=16000000

# the total bytes of memory the producer can use to buffer records waiting to be sent to the server
topic.metadata.refresh.interval.ms=60000
enable.idempotence=false
max.in.flight.requests.per.connection=1000000
acks=0
```



## 3.	결과 비교
사전 테스트 결과

```
[kafca@localhost bin]$ ./kafka-producer-perf-test.sh --topic test --num-records 200000 --record-size 1000 --throughput -1 --producer-props bootstrap.servers=192.168.1.93:9092,192.168.1.93:9091,192.168.1.93:9093
27697 records sent, 5483.5 records/sec (5.23 MB/sec), 2462.0 ms avg latency, 3977.0 ms max latency.
58960 records sent, 11782.6 records/sec (11.24 MB/sec), 3284.4 ms avg latency, 4494.0 ms max latency.
66224 records sent, 13242.2 records/sec (12.63 MB/sec), 2362.6 ms avg latency, 3105.0 ms max latency.
200000 records sent, 10645.659232 records/sec (10.15 MB/sec), 2639.14 ms avg latency, 4494.00 ms max latency, 2565 ms 50th, 3836 ms 95th, 4197 ms 99th, 4388 ms 99.9th.

[kafca@localhost bin]$ ./kafka-consumer-perf-test.sh --topic test --broker-list 192.168.1.93:9092,192.168.1.93:9091,192.168.1.93:9093 --messages 200000 start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec, rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec WARNING: Exiting before consuming the expected number of messages: timeout (10000 ms) exceeded. You can use the --timeout option to increase the timeout. 2024-04-08 10:34:39:195, 2024-04-08 10:34:49:921, 13.5422, 1.2626, 14200, 1323.8859, 413, 10313, 1.3131, 1376.9029
```
변경 후 테스트 결과

```

[kafca@localhost bin]$ ./kafka-producer-perf-test.sh --topic test --num-records 200000 --record-size 1000 --throughput -1 --producer-props bootstrap.servers=192.168.1.93:9092,192.168.1.93:9091,192.168.1.93:9093
80070 records sent, 15969.3 records/sec (15.23 MB/sec), 1193.0 ms avg latency, 1885.0 ms max latency.
117312 records sent, 23462.4 records/sec (22.38 MB/sec), 1422.2 ms avg latency, 2059.0 ms max latency.
200000 records sent, 19508.388607 records/sec (18.60 MB/sec), 1328.63 ms avg latency, 2059.00 ms max latency, 1360 ms 50th, 1794 ms 95th, 1984 ms 99th, 2038 ms 99.9th.


[kafca@localhost bin]$ ./kafka-consumer-perf-test.sh --topic test --broker-list 192.168.1.93:9092,192.168.1.93:9091,192.168.1.93:9093 --messages 200000
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec, rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec
2024-04-08 14:40:52:315, 2024-04-08 14:40:54:135, 191.0563, 104.9760, 200337, 110075.2747, 550, 1270, 150.4380, 157745.6693
```


Producer:
```
총 200,000개의 레코드를 보냈습니다.
평균 throughput은 약 10,645.66 records/초 (약 10.15 MB/초)입니다.
평균 지연 시간은 약 2,639.14ms이며, 최대 지연 시간은 4,494.00ms입니다.
```
Consumer:
```
총 200,000개의 메시지를 소비했습니다.
데이터 소비 속도는 약 1.2626 MB/초이며, 메시지 소비 속도는 약 1,323.89개/초입니다.
리밸런스 시간은 413ms이고, 메시지 검색 시간은 10,313ms입니다.
두 번째 변경 후 테스트 결과:
```
Producer:
```
총 200,000개의 레코드를 보냈습니다.
평균 throughput은 약 19,508.39 records/초 (약 18.60 MB/초)입니다.
평균 지연 시간은 약 1,328.63ms이며, 최대 지연 시간은 2,059.00ms입니다.
```
Consumer:
```
총 200,337개의 메시지를 소비했습니다.
데이터 소비 속도는 약 104.9760 MB/초이며, 메시지 소비 속도는 약 110,075.27개/초입니다.
리밸런스 시간은 550ms이고, 메시지 검색 시간은 1,270ms입니다.
변경 전과 변경 후의 결과를 비교해 보겠습니다.
```

```
Producer Throughput: 변경 후에 생산자의 throughput이 대폭 향상되었습니다.

변경 전 throughput은 약 10,645.66 records/초에서 변경 후에는 약 19,508.39 records/초로 약 1.83배 향상되었습니다.

이는 처리량이 거의 두 배로 증가했음을 의미합니다.

Producer Latency: 변경 후에 평균 지연 시간이 2,639.14ms에서 1,328.63ms로 줄었습니다.

즉, 변경 후에 생산자는 메시지를 더 빠르게 처리했습니다.

Consumer Throughput: 변경 후에 데이터 소비 속도와 메시지 소비 속도가 대폭 향상되었습니다.

데이터 소비 속도는 약 1.2626 MB/초에서 약 104.9760 MB/초로 약 83배,

메시지 소비 속도는 약 1,323.89개/초에서 약 110,075.27개/초로 약 83배 향상되었습니다.

Consumer Latency: 소비자의 평균 지연 시간은 변경 전에 10,313ms에서 변경 후에 1,270ms로 크게 감소했습니다.
```

## 4.	log 용량에 따른 최적화
 ![image](https://github.com/auspicious0/Kafka_performance_optimization_ex/assets/108572025/de2f7faa-bf42-46d2-87dc-4f7ae70a279b)

 log용량에 따라 최적화 되는 것을 볼 수 있다.

 ![image](https://github.com/auspicious0/Kafka_performance_optimization_ex/assets/108572025/5c6e4802-7999-4785-8436-f72d9f8dff2c)
