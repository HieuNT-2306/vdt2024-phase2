### Quy hoạch
+ Node hieuserver (192.168.1.1) labels: _admin, rgw, mon, osd
+ Node ceph-osd-01 (192.168.1.21) labels: osd

### 1. Trên các cụm máy ảo, dưới quyền root tải docker, openssh, ceph:
- Docker:
```
apt update
apt install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install -y docker-ce

systemctl start docker
systemctl enable docker
```

- OpenSSH:

```
apt install openssh-server
systemclt enable ssh
```

- Ceph:
```
apt update -y
apt install -y cephadm ceph-common
```
- Bật PermitRootLogin cho các máy OSD trong /etc/ssh/sshd_config:
```
PermitRootLogin yes
```

### 2. Cấu hình host cho hieuserver và các node trong /etc/hosts:
```
192.168.1.1 hieuserver
192.168.1.21 ceph-osd-01
```
 
### 3. Tạo cluster thông qua lệnh bootstrap trên hieuserver:

```
cephadm bootstrap --mon-ip 192.168.1.1 --ssh-user root --cluster-network 192.168.1.0/24
```
- Sau khi tạo xong, lưu lại mật khẩu admin dashboard và đăng nhập:

```
Ceph Dashboard is now available at:

             URL: https://hieuserver:8443/
            User: admin
        Password: **********

Enabling client.admin keyring and conf on hosts with "admin" label
```

### 4. Mở các cổng cần thiết thông qua iptables ở tất cả các máy:
```
iptables -A INPUT -s 192.168.1.0/24 -p tcp -m state --state NEW -m multiport --dports 3300,6789,6800:7300,9283,18080,9100,9222 -j ACCEPT
iptables -A OUTPUT -d 192.168.1.0/24 -p tcp -m state --state NEW -m multiport --dports 3300,6789,6800:7300,9283,18080,9100,9222 -j ACCEPT

iptables -A INPUT -s 192.168.1.0/24 -p tcp -m state --state NEW --dport 22 -j ACCEPT
iptables -A OUTPUT -d 192.168.1.0/24 -p tcp -m state --state NEW --dport 22 -j ACCEPT

```
### 5. Tắt tự động deploy monitor, manager, và chỉ định rõ node hieuserver làm monitor, manager:
```
ceph orch apply mon --unmanaged
ceph orch apply mon --placement="hieuserver"
ceph orch apply mgr --unmanaged
ceph orch apply mgr --placement="hieuserver"
```
### 6. Gửi ceph.pub từ hieuserver đến các node khác và thêm ceph-osd-01 vào host:
```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd-01

ceph orch host add ceph-osd-01 --labels osd
```

### 7. Thêm osd vào host bằng 2 cách:

#### 7.a Sử dụng ceph-volume:

- Trên các node osd như ceph-osd-01:
```
#Trên node monitor:
ceph auth get-or-create-key client.bootstrap-osd mon 'allow profile bootstrap-osd' -i /var/lib/ceph/bootstrap-osd/ceph.keyring

#Gửi keyring cho các host:
scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@ceph-osd-01:/var/lib/ceph/bootstrap-osd/

#Trên các node osd, kích hoạt keyring:
ceph auth import -i /var/lib/ceph/bootstrap-osd/ceph.keyring

#Thêm osd thông qua ceph-volume:
ceph-volume lvm create --data /dev/sdb
```

#### 7.b Sử dụng file osd_spec.yaml
- Tạo 1 file osd_spec.yaml như sau:
```
service_type: osd
service_id: osd_spec_default
placement:
  hosts:
    - hieuserver
    - ceph-osd-01
spec:
  data_devices:
    all: true
```
- Chạy thử lệnh thông qua cờ --dry-run
```
ceph orch apply -i osd_spec.yaml --dry-run
# Chờ 3p, chạy lại để kiểm kết quả:
ceph orch apply -i osd_spec.yaml --dry-run
```
- Sau khi có kết quả, chạy:
```
ceph orch apply -i osd_spec.yaml 
```

```
ceph osd out osd.2
ceph osd crush remove osd.2
ceph auth del osd.2
ceph osd rm osd.2
```
Các lệnh sau để xóa osd
