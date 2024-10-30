# Kết hợp Fluentd -> Kafka -> Clickhouse:

## I. Quy hoạch:
- Trong lab này, ta sẽ sử dụng 3 máy ảo:
- 171.254.95.137 để chạy Fluentd
- 171.254.95.193 để làm Kafka broker
- 171.254.93.218 để chạy Clickhouse

## II. Cài đặt:
### 1. Fluentd:
- Cài đặt fluentd thông qua fluent-package:

```
curl -fsSL https://toolbelt.treasuredata.com/sh/install-ubuntu-noble-fluent-package5-lts.sh | sh
```
- Kiểm tra cài đặt:
```
sudo systemctl start fluentd.service
sudo systemctl status fluentd.service
```
### 2. Kafka:
- Đầu tiên, cài đặt JDK:
```
apt update
apt install openjdk-11-jdk
```
- Tải và giải nén kafka về:
```
wget https://archive.apache.org/dist/kafka/3.4.0/kafka_2.12-3.4.0.tgz
tar -xvzf ~/kafka_2.12-3.4.0.tgz
mv kafka_2.12-3.4.0/ kafka/ # Đổi tên thư mục thành kafka.
```
-  Khởi động ZooKeeper và Kafka Broker:
```
~/kafka/bin/zookeeper-server-start.sh ~/kafka/config/zookeeper.properties &
~/kafka/bin/kafka-server-start.sh ~/kafka/config/server.properties &
# Chạy 2 lệnh trên trong nền hoặc trong 1 màn hình mới
```
- Kiểm tra cài đặt thông qua việc tạo 1 topic, 1 producer và 1 consumer:
```
~/kafka/bin/kafka-topics.sh --bootstrap-server 171.254.95.193:9092 --create --topic topic-1

#Tạo producer: 
 ~/kafka/bin/kafka-console-producer.sh --bootstrap-server 171.254.95.193:9092 --topic topic-1

#Tạo consumer:
~/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic topic-1 --from-beginning
```
### 3. Clickhouse:

- Cài đặt clickhouse client và server:
```
apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | sudo gpg --dearmor -o
/usr/share/keyrings/clickhouse-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb
stable main" | sudo tee \ /etc/apt/sources.list.d/clickhouse.list

apt-get update
apt-get install -y clickhouse-server clickhouse-client
```
- Khởi tạo 1 clickhouse server:
```
clickhouse start
```
- Khởi tạo 1 clickhouse-client:
```
clickhouse-client --password
```
## III. Kết nối:
- Trước khi cài đặt kết nối đến kafka cho fluentd, ta cần cài đặt plugin dưới đây:
```
fluent-gem install fluent-plugin-kafka  
```
- Tại node chứa fluentd, ta sẽ tạo 1 log giả và đẩy vào trong `/tmp/test.log`:
```
echo "Testing" >> /tmp/test.log
```
- Để cài đặt việc nhận log này, vào trong `/etc/fluent/fluentd.conf `, ta cài đặt cấu hình như sau:
```
<source>
  @type tail
  path /tmp/*.log
  pos_file /var/log/fluent/test.log.pos
  tag test.log
  format none
</source>

<filter test.log>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    tag ${tag}
    time ${time}
  </record>
</filter>

<match test.log>
  @type kafka2
  brokers 171.254.95.193:9092
  use_event_time true
  <format>
    @type json
  </format>
  <buffer>
     flush_at_shutdown true
     flush_mode immediate
     overflow_action throw_exception
   </buffer>
  topic_key test-logs
  default_topic test-logs
</match>
```
- Config trên sẽ nhận các thay đổi trong các file có đuôi `.log` truyền vào từ trong `/tmp/`, thêm các trường `hostname`, `tag`, `time` vào trong log, sau đó gửi đến broker của kafka.
- Ta để `flush_mode immediate` để có thể gửi log trực tiếp mà không cần chờ `buffer`, điều này làm tăng tốc độ gửi log tới kafka.
- Ví dụ về 1 log được tạo:
```
{"message":"Testing","hostname":"ceph-node-2","tag":"test.log","time":"2024-10-30 14:45:42 +0700"}
```
- Tại node chứa kafka, khởi động zookeeper và kafka broker và tạo 1 topic là `test-logs`:
```
 ~/kafka/bin/kafka-topics.sh --create --topic test-logs --bootstrap-server 171.254.95.193:9092
```
- Sau đó, ta có thể mở 1 consumer để kiểm tra logs đã được gửi thành công hay không:
```
~/kafka/bin/kafka-console-consumer.sh --topic test-logs --from-beginning --bootstrap-server 171.254.95.193:9092
```
- Cuối cùng, tại Clickhouse, ta cần tạo 1 bảng mới với engine kafka như sau:
```
CREATE TABLE kafka_logs (
    message String,
    hostname String,
    tag String,
    time String,
    received_time DateTime
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = '171.254.95.193:9092',
    kafka_topic_list = 'test-logs',
    kafka_group_name = 'my_group',
    kafka_format = 'JSONEachRow';
```
- Đây sẽ là bảng nhận dữ liệu từ Kafka, do sử dụng Kafka engine nên bảng này không có chức năng lưu trữ lâu dài, còn bảng chính để lưu logs, ta cần tạo thêm bảng sau:
```
CREATE TABLE logs (
    message String,
    hostname String,
    tag String,
    time String,
    received_time DateTime
) ENGINE = MergeTree()
ORDER BY time;
```
- Sau đó, ta cần tạo 1 MATERALIZED VIEW để tự động lấy dữ liệu từ bảng kafka_logs và chèn vào bảng logs mỗi khi có dữ liệu mới.
```
CREATE MATERIALIZED VIEW mv_kafka_logs TO logs AS
SELECT
    message,
    hostname,
    tag,
    time,
    now() AS received_time
FROM kafka_logs;
```
- Khi tạo các bảng xong, ta sẽ có thể gửi log trực tiếp từ node chứa fluentd đến clickhouse:

```
# Giả lập tạo log ở nút có fluentd:

$ echo "Đây là 1 log mới" >> /tmp/test.log

# Kiểm tra ở bên kafka consumer:

$ /kafka/bin/kafka-console-consumer.sh --topic test-logs --from-beginning --bootstrap-server 171.254.95.193:9092
{"message":"Đây là 1 log mới","hostname":"ceph-node-2","tag":"test.log","time":"2024-10-30 15:20:03 +0700"}

# Kiểm tra ở bên clickhouse thông qua lệnh SELECT:
SELECT *
FROM logs
LIMIT 10

Query id: 1d66f863-f59a-406d-bce1-7d155e1571be
    ┌─message──────────┬─hostname────┬─tag──────┬─time──────────────────────┬───────received_time─┐
1.  │ Đây là 1 log mới │ ceph-node-2 │ test.log │ 2024-10-30 15:20:03 +0700 │ 2024-10-30 15:20:05 │
    └──────────────────┴─────────────┴──────────┴───────────────────────────┴─────────────────────┘
```