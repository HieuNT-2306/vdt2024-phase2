# MinIO & MinIO Client

## Cài đặt MinIO - MinIO Client:

- Cài đặt theo hướng dẫn của file [MOP-deploy-minio](../Guide/MOP-deploy-minio.md).

## Thực hành với MinIO:

- Đầu tiên, ta cần đặt 1 alias cho cluster MinIO:  
```
mc alias set <alias> <MinIO Cluster endpoint> <username> <password>
```
- Ví dụ:
```
mc alias set myminio http://192.168.176.7:9000 minioadmin minioadmin
Added `myminio` successfully.
```
- Kiểm tra thông qua lệnh `mc alias list`:
```
myminio
  URL       : http://192.168.176.7:9000
  AccessKey : minioadmin
  SecretKey : minioadmin
  API       : s3v4
  Path      : auto
```
- Hoặc ta có thể sử dụng lệnh `mc alias import`:
```
nano myminio.json

...
{
        "url":"http://192.168.176.7:9000",
        "accessKey":"minioadmin",
        "secretKey":"minioadmin",
        "api":"s3v4",
        "path":"auto"
}
...
mc alias import myminio myminio.json
```
- Kiểm tra thông qua lệnh 
- Sau đó, ta có thể sử dụng alias này cho các tác vụ khác, VD, khi ta muốn tạo 1 bucket vào trong cluster:
```
#Tạo 1 bucket tên là testbucket vào trong cluster minio.
mc mb myminio/testbucket 
```
- Sử dụng lệnh copy để di chuyển file giữa các bucket, cluster.

```
#Di chuyển file lên trên bucket:
mc cp hello.txt myminio/bucket1

#Copy file từ bucket về:
mc cp myminio/bucket1/hello.txt hello2.txt

#Di chuyển file trong các bucket đến bucket khác ở cluster khác (Đây là cluster public của minio, nhằm mục đích học tập và thử nghiệm)
mc cp myminio/bucket1 play/bucket1 --recursive 
```
- Tìm kiếm file thông qua `mc find`
```
mc find myminio/bucket1 --name "*.pptx" #Tìm kiếm thông qua đuôi file
mc find myminio/bucket --larger 100m #Tìm kiếm thông qua kích thước file
#Ta có thể thêm mc ls $(mc find ....) để có thể xem thông tin chi tiết về kích thước file, ngày thêm 
```

- Liệt kê file và bucket thông qua `mc ls`
```
mc ls (-r) myminio (--summarize) (--json)
#Flag -r là flag recursive, để liệt kê toàn bộ object trong 1 cluster, 1 bucket,....
#Flag --summarize để cho thêm thông tin về tổng số object và tổng dung lượng
#Flag --json để thông tin về dạng --json.
```
- Xem cấu trúc của một cluster hoặc bucket thông qua `mc tree`:
```
mc tree (--files) myminio
#Flag --files để liệt kê tất cả cấu trúc của 1 bucket, bao gồm cả các files
#Flag --depths để chỉ độ sâu liệt kê ra.
```
- `mc mv`, giống `mc cp`, để di chuyển file giữa các bucket, các cluster, và cả các host, nhưng mà file sẽ được di chuyển hoàn toàn lên vị trí mới.
```
mc mv -r myminio/bucket1/bai-tap-thuc-hanh/session-1 myminio/bucket2
```
- `mc mirror`, giống với `mc cp`, sẽ là sao chép hàng loạt các object trong 1 vùng, tuy nhiên, có các cờ khác nhau như sau:

- Cờ mà mc cp cung cấp nhưng mc mirror không có:
  - --rewind value: quay lại đối tượng đến phiên bản hiện tại tại thời điểm đã chỉ định
  - --version-id value, --vid value: chọn một phiên bản đối tượng để sao chép
  - --attr: thêm siêu dữ liệu tùy chỉnh cho đối tượng (định dạng: KeyName1=string;KeyName2=string)
  - --continue, -c: tạo hoặc tiếp tục phiên sao chép
  - --tags: áp dụng thẻ cho các đối tượng đã tải lên (ví dụ: key=value&key2=value2, v.v.)
  - --rewind value: quay lại đối tượng đến phiên bản hiện tại tại thời điểm đã chỉ định
- Cờ mà mc mirror cung cấp nhưng mc cp không có:
  - --exclude value: loại trừ các đối tượng phù hợp với mẫu tên object đã chỉ định
  - --fake: thực hiện một thao tác mirror giả
  - --overwrite: ghi đè lên các object trên đích nếu khác với nguồn
  - --region value: chỉ định khu vực khi tạo các bucket mới trên đích (mặc định: "us-east-1")
  - --watch, -w: theo dõi và đồng bộ hóa các thay đổi (điều này có thể rất quan trọng)


- Cuối cùng, sử dụng lệnh `mc rm` hoặc `mc rb` để xóa 1 object, 1 folder hoặc 1 bucket: 
```
mc rm --force myminio/bucket3/hello.txt
mc rb --force myminio/bucket3
```
