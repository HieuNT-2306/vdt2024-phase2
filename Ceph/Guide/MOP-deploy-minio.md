## A. Chuẩn bị:

### Quy hoạch:

- Một cụm k8s, 1 node với địa chỉ ip 171.254.95.224, được cài đặt calico CNI.

### B. Cài đặt MinIO server: 

- Cài đặt một cụm minio server cơ bản thông qua file .yaml sau:
```
apiVersion:v1
kind:Namespace
metadata:
  name: minio-dev
  labels:   
    name:minio-dev
  apiVersion: v1
kind: Pod
metadata:
  labels:
    app: minio
  name: minio
  namespace: minio-dev
spec:
  containers:
  - name: minio
    image: quay.io/minio/minio:latest
    command:
    - /bin/bash
    - -c
    args:
    - minio server /data --console-address :9090
    volumeMounts:
    - mountPath: /data
      name: localvolume # Corresponds to the `spec.volumes` Persistent Volume
  volumes:
    - name: localvolume
  hostPath: # MinIO generally recommends using locally-attached volumes
      path: /mnt/disk1/data
      type: DirectoryOrCreate
```
- Sau đó, truy cập vào cổng 9090 hoặc sử dụng câu lệnh:
```
 kubectl port-forward pod/minio 9000 9090 -n minio-dev 
```
- Để export dashboard ra cổng 9000 trên máy host.
- Trên trang dashboard, đăng nhập bằng 
  - Tài khoản: minioadmin
  - Mật khẩu: minioadmin    

### C. Cài đặt MinIO Client:

- Trước hết, cần chạy lệnh sau để xem loại cpu của máy:
```
lscpu
```
- Sau đó, chạy 1 trong 3 lệnh sau để tải MinIO Client về:
```
curl https://dl.min.io/client/mc/release/linux-amd64/mc \
  --create-dirs \
  -o $HOME/minio-binaries/mc #Với cpu amd/intel
```
```
curl https://dl.min.io/client/mc/release/linux-ppc64le/mc \
  --create-dirs \
  -o ~/minio-binaries/mc #Với cpu ppc
```
```
curl https://dl.min.io/client/mc/release/linux-arm64/mc \
  --create-dirs \
  -o ~/minio-binaries/mc #Với cpu arm
```
- Sau đó, cấu hình để sử dụng MinIO Client `mc`:
```
chmod +x $HOME/minio-binaries/mc
export PATH=$PATH:$HOME/minio-binaries/

mc --help
```