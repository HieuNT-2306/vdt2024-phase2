# Kế hoạch Benchmark, so sánh hiệu năng giữa CEPHADM và ROOK-CEPH trong môi trường nhỏ.

## A. Chuẩn bị môi trường:

- Quy hoạch của 2 cụm ceph được benchmark trong node:
  - Nằm cùng trên 1 máy ảo, 1 node.
  - 1 Monitor
  - 1 Manager
  - 3 Osd, nằm trong cùng 1 virtual disk với bộ nhớ 120GB ( Mỗi osd chứa 40 GB)

- Cấu hình của VM chạy thử nghiệm:
  - CPU: 24 thread CPU
  - RAM: 32GiB
  - Disk: HDD 500GB

## B. Kế hoạch benchmark:

### 1. Công cụ sử dụng:

- [FIO](https://github.com/axboe/fio): Một công cụ lâu đời, linh hoạt cho các thử nghiệm I/O, hỗ trợ nhiều loại workload như đọc, ghi, và các mô hình truy cập khác nhau, có thể được sử dụng cho cả ổ đĩa vật lý và ảo hóa.

- [CBT (Ceph Benchmarking Tool) ](https://github.com/ceph/cbt): Là công cụ được thiết kế đặc biệt cho việc kiểm tra hiệu suất của các cụm Ceph. Nó tích hợp nhiều module khác nhau như radosbench và rbdfio để thực hiện các bài kiểm tra hiệu suất. CBT cho phép thu thập và phân tích dữ liệu từ cụm Ceph một cách chi tiết và có hệ thống.

- [SIBENCH](https://github.com/SoftIron/sibench): Là một công cụ benchmarking mã nguồn mở nhỏ mới, được thiết kế để mô phỏng và đo lường hiệu suất của hệ thống lưu trữ. Giúp phân tích hiệu suất của Ceph trong các tình huống thực tế.

- [hsbench](https://github.com/ceph/hsbench.git): là một công cụ dùng để kiểm tra hiệu suất của 1 cụm Ceph. Nó mô phỏng các hoạt động đọc và ghi của nhiều client để đánh giá hiệu suất tổng thể của 1 cụm.

- [gosbench](https://github.com/ceph/gosbench.git): công cụ kiểm tra hiệu suất cho dịch vụ S3 và lưu trữ đối tượng trong Ceph, giúp đánh giá khả năng xử lý của các dịch vụ này khi có nhiều yêu cầu đồng thời gửi tới.

- RADOS Bench, RBD bench: Các công cụ kiểm tra hiệu suất tích hợp sẵn trong Ceph, dùng để đo hiệu suất của các dịch vụ rgw, rbd

- Ngoài ra, đối với việc đo hiệu suất của các bucket thông qua s3, ta sẽ sử dụng lệnh `time s3cmd` trong s3cmd để đo tốc độ tạo và xóa file lên trên bucket của các cụm ceph. 

### 2. Kịch bản benchmark:
- Tạo 1 pool mới lên trên 2 cụm cluster với:
```
ceph osd pool create testbench 100 100
```
#### 2.1 Benchmark hiệu năng của 1 cụm cluster:
- Input: 
    - fsid của 1 cụm cluster:
    - Tên của cụm cluster:
    - Thời gian thử nghiệm
    - Số lượng client
- Output:
    - Số lượng yêu cầu
    - Thời gian phản hồi
    - Thông lượng  
#### 2.2 Benchmark đọc/ghi 1 object lên 1 pool mới:
- Input:
    - Thời gian chạy thử nghiệm.
    - Kích thước của file đọc/ghi.
    - Số lượng luồng sử dụng.
- Output:
    - Tổng số lần đọc/ghi.
    - Băng thông tối thiểu, tối đa, trung bình được sử dụng.
    - IOPS (Input Output Per Seconds - số thao tác đọc - ghi trong 1 giây) tối thiểu, tối đa, trung bình.

#### 2.3 Benchmark đọc/ghi lên 1 image tạo bởi dịch vụ rbd:
- Input:
    - Kích thước của image.
    - Thời gian chạy thử nghiệm.
    - Số lượng luồng sử dụng.
- Output:
    - Số thao tác hoàn thành, số thao tác hoàn thành mỗi giây.
    - Tổng số byte được đọc/ghi mỗi giây.

#### 2.4 Benchmark tốc độ đẩy file, lấy file, liệt kê file qua dịch vụ s3api sử dụng s3cmd:
- Input:
    - Các thông tin về s3user như access_key, secret_key,....
    - Số lượng object đẩy lên.
    - Kích thước object.
- Output:
    - Thời gian hoàn thành hoạt động.
    - Thời gian phản hồi.

#### 3. Tài liệu tham khảo:
- https://docs.redhat.com/en/documentation/red_hat_ceph_storage/5/html/administration_guide/ceph-performance-benchmarking#ceph-performance-benchmarking