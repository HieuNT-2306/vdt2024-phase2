# Lab week 4, Benchmark giữa cephadm và rook-ceph.

# A. Chuẩn bị:

## Cài đặt go trên các vm:
```
wget https://golang.org/dl/go1.21.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
```
- Trong bash, cài đặt:
```
export PATH=$PATH:/usr/local/go/bin

source ~/.bashrc
```

## Cài đặt C, C++:
```
sudo apt update
sudo apt install build-essential
```

# B. Benchmark:

## 1. Sử dụng hsbench để kiểm tra hiệu suất của 1 cluster:
- Đầu tiên, cài đặt hsbench:
```
mkdir hsbench && cd hsbench
go mod init hsbench
go install github.com/markhpc/hsbench@latest
```
- Gán path của hsbench vào bin:
```
ls $HOME/go/bin/hsbench
export PATH=$PATH:$HOME/go/bin
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```
- Chạy hsbench để bắt đầu benchmark 1 cụm cluster:
```
hsbench -a <Access-key> -s <Secret-key> -u <Endpoint> -z <Object-size> -d <Duration> -t <Number-of-threads> -b <Number-of-buckets>
```

## 2. Sử dụng gosbench để kiểm tra hiệu xuất của dịch vụ object storage thông qua s3api:

- Cài đặt gosbench server lên 1 máy:
```
go install github.com/mulbc/gosbench/server@latest
```
- Viết 1 file .yaml config cho server:
```
---

s3_config:
  - access_key: <Access-key-1>
    secret_key: <Secret-key-1>
    region: <Region-1>
    endpoint: <Endpoint-for-service-rgw-in-cluster1>
    skipSSLverify: false

  - access_key: <Access-key-2>
    secret_key: <Secret-key-2>
    region: <Region-2>
    endpoint:  <Endpoint-for-service-rgw-in-cluster2>
    skipSSLverify: false
grafana_config:
  endpoint: http://<host-ip>:9090
  username: admin
  password: grafana

tests:
  - name: Test 1
    read_weight: 100
    existing_read_weight: 0
    write_weight: 100
    delete_weight: 100
    list_weight: 100
    objects:
      size_min: 20
      size_max: 100
      part_size: 0
      size_distribution: random
      unit: KB
      number_min: 100
      number_max: 2000
      number_distribution: constant
    buckets:
      number_min: 10
      number_max: 10
      number_distribution: constant
    bucket_prefix: 1255gosbench-
    object_prefix: obj
    stop_with_runtime:
    stop_with_ops: 10
    workers: 2
    parallel_clients: 2
    clean_after: True
...
```
- Tạo 1 server thông qua lệnh: 
```
server -c gosbench-config.yaml 
```
- Sau đó, trên cả 2 máy, tải gosbench worker về:
```
go install github.com/mulbc/gosbench/worker@latest
```
- Mở cổng 2000 trên 2 máy:
```
sudo iptables -A INPUT -p tcp --dport 2000 -j ACCEPT
```
- Chạy các lệnh sau trên 2 máy để tạo worker:
```
worker -s <host-server-ip>:2000  (-p 8878)
# Nếu port 8888 đã sử dụng, hoặc không muốn sử dụng port 8888, thêm cờ -p này.
```

## 3. Sử dụng fio để đánh giá hiệu xuất các dịch vụ i/o của object và images rbd:
- Cài đặt fio:
```
apt install fio
```
- Đối với cụm cephadm, các cấu trúc file `etc/ceph/` đã có sẵn `ceph.conf` và `keyring` nên ta không cần config gì, còn đối với cụm `ceph-rook`, ta cần phải export ceph.conf và keyring từ trong pods ceph-toolbox ra:
```
mkdir /etc/ceph/
kubectl exec -it -n rook-ceph rook-ceph-tools-767b99dbdd-t4tvv -- cat /etc/ceph/ceph.conf > /etc/ceph/ceph.conf 
kubectl exec -it -n rook-ceph <ceph-toolbox-pod-id> -- cat /etc/ceph/keyring > /etc/ceph/keyring
```
- Sau đó, viết các file kịch bản `.fio` để chạy thử:
```
[global]
ioengine=rados 
runtime=60 #Thời gian thử nghiệm: 60s
time_based #Chạy trong 1 khoảng thời gian xác định
rw=randrw #Kiểu mẫu i/o: đọc, ghi ngẫu nhiên
rwmixread=70 #Chỉ tỉ lệ đọc ghi: 70 đọc - 30 ghi
size=1G #Thực hiện trên 1 khối 1GB
numjobs=4 #Số lượng luồng sử dụng
blocksize=4k #Kích thước mỗi thao tác io
pool=testpool #Tên pool thực hiệnn
iodepth=4 #Số lượng thao tác i/o đồng thời gửi tới
loops=5 #Số lượng lặp lại các thao tác.
[randrw]
```
- Đối với dịch vụ rbd, ta sẽ viết như sau:
```
[global]
ioengine=rbd
runtime=60s
time_based
blocksize=4k
numjobs=1
group_reporting

[seq-read]
pool=rbd.pool
rbdname=image1 #Tên image
rw=read
size=2G

[seq-write]
rbdname=image1 #Tên image
pool=rbd.pool
rw=write
size=2G
```
- Sau đó, áp dụng thông qua lệnh:
```
fio mixed.fio --output=mixed_rook.txt
```
## 4. Benchmark hiệu năng của ceph cluster thông qua rados bench:
- Tạo 1 giống nhau pool ở cả 2 cụm ceph:
```
ceph osd pool create testpool 32 32 rep_hdd_osd 3 
```
- Sử dụng rados bench có sẵn trong cụm ceph để thực hiện benchmark các cụm thông qua lệnh:
```
rados bench -p testbench 20 write --no-cleanup
rados bench -p testbench 20 read --no-cleanup
```
- Chi tiết câu lệnh:
  - -p: cờ để xác định tên pool.
  - 20: Thời gian chạy test, tính theo giây.
  - write/read: Thao tác chạy
  - --no-cleanup: Cờ để ngắt việc remove object sau test, nhằm tăng tốc độ đọc ở các test sau.
- Sau khi kết thúc test, ta cần chạy lệnh:
```
rados -p testbench cleanup
```
# C. Kết quả chạy:

- Kết quả chạy và lệnh chạy sẽ có trong file excel sau [đây](../Lab-report/Benchmark%20CEPHROOK%20vs%20CEPHADM.xlsx)
- Tổng kết:
### 1. Kết quả khi test tốc độ IOPS của 1 pool thông qua fio:
-  Tốc độ đọc và ghi của cụm rookceph đều chậm hơn cụm cephadm từ 3-4%, điều này còn tệ hơn đối với tốc độ đọc ghi hỗn hợp, khi mà có thể chênh lệnh lên tới 5.8%
-  Tương tự với băng thông và độ trễ, đều là chênh lệch trong khoảng 3-5%
### 2. Kết quả khi test performance của cluster thông qua rados bench:
- Tốc độ ghi của rook-ceph chậm hơn cephadm từ 2.4% đến 2.5%, tuy nhiên, tốc độ đọc của rook-ceph lại nhanh hơn từ 12%-29%.
- Tương tự với băng thông và độ trễ, đều có chênh lệch lớn giữa kết quả của đọc so với kết quả của ghi, cho thấy rằng rook-ceph có lợi thế lớn hơn khi đọc dữ liệu.

###  3. Kết quả khi test hiệu xuất của api s3 thông qua hsbench:
- Tương tự, tốc độ ghi (PUT) và xóa (DELETE) của rook-ceph chậm hơn cephadm, nhưng tốc độ đọc của rook-ceph lại lớn hơn trông thấy so với tốc độ đọc của cephadm.

### 4. Giả định nguyên nhân:
#### Về fio:
- Rook-Ceph tích hợp chặt chẽ với Kubernetes, sử dụng cơ sở hạ tầng của k8s để triển khai và quản lý các dịch vụ Ceph. Điều này có thể dẫn đến một số overhead nhất định, đặc biệt là khi sử dụng fio để kiểm tra hiệu năng truy cập file hệ thống, do k8s cần quản lý các pod và dịch vụ.
- Do fio đo lường hiệu năng truy cập I/O ngẫu nhiên, phải thông qua 1 lớp ảo hóa nên nó có thể bị ảnh hưởng bởi cách mà các khối lưu trữ được quản lý bởi Kubernetes và cơ sở hạ tầng của Rook.
- Cephadm, ngược lại, quản lý các daemon của Ceph trực tiếp trên hệ thống mà không cần thông qua k8s, điều này có thể giúp tăng hiệu suất trong các tác vụ đòi hỏi nhiều tài nguyên như fio
#### Về hsbench và gosbench:
- Rook sử dụng k8s để quản lý các dịch vụ Ceph, điều này có thể làm tăng overhead trong một số tác vụ, nhưng lại mang lại lợi ích trong việc cân bằng tải và khả năng tự phục hồi khi gặp lỗi. Điều này có thể giải thích tại sao Rook lại cho kết quả tốt hơn trong các bài test như hsbench và rados bench cho các tác vụ liên quan đến đọc ngẫu nhiên và tuần tự.

### 5. Kết luận:
- Ta nên chọn Cephadm nếu:
  - Ta muốn một hệ thống Ceph không phụ thuộc vào k8s mà chạy bare metal (Sử dụng cho một khách hàng duy nhất).
  - Ta muốn tập trung vào các tác vụ I/O ngẫu nhiên hoặc truy cập file hệ thống (fio).
  - Ta cần một hệ thống đơn giản với hiệu năng ổn định mà không cần nhiều tích hợp với k8s.

- Ta nên chọn Rook-Ceph nếu:
  - Ta có dự định triển khai K8s và muốn tích hợp chặt chẽ với hệ sinh thái này.
  - Ta ưu tiên hiệu năng tốt hơn trong các tác vụ S3 hoặc các thao tác đọc/GET so với các thao tác ghi/PUT và xóa/DELETE trên Ceph.
  - Ta cần một giải pháp dễ dàng mở rộng và quản lý trong môi trường nhiều container.
  - Ta muốn triển khai 1 cụm ceph nhanh chóng với đầy đủ chức năng từ RESTapi đến telementry,....

### 6. Lưu ý:
- Các bài test trên đều được thử nghiệm trong 1 môi trường lab nhỏ, với các cụm ceph chỉ có 1 host duy nhất. Các kết quả này sẽ có thể khác với các cụm ceph to hơn, do ceph hoạt động hiệu quả hơn với các cụm lớn, có dung lượng tính từ TiB trở lên.
- Các con máy ảo trong lab này đều chưa được tuning, nên điều này có thể dẫn đến thay đổi kết quả.
- Cần thử nghiệm nhiều hơn ở các cụm ceph lab to hơn để rõ kết quả hơn. 