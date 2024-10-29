# Proxy cho các máy ảo:

1. Proxy cho toàn bộ user:

```
sudo nano /etc/enviroment
...
export http_proxy="hieu:1@10.61.11.42:3128"
export https_proxy="hieu:1@10.61.11.42:3128"
export fpt_proxy="hieu:1@10.61.11.42:3128"
export no_proxy="localhost,127.0.0.1,::1"
```

2. Proxy cho APT:
- Tạo file apt.conf
```
sudo nano /etc/apt/apt.conf
```
- Thêm câu lệnh sau vào trong file:

```
Acquire::http::Proxy "http://hieu:1@10.61.11.42:3128";
Acquire::https::Proxy "http://hieu:1@10.61.11.42:3128";
```

3. Proxy cho Docker:

- Tạo file proxy.conf trong docker.service.d:
```
sudo nano /etc/systemd/system/docker.service.d/proxy.conf 
```
- Thêm biến môi trường cho docker:
```
[Service]
Environment="HTTP_PROXY=http://10.61.11.42:3128/"
Environment="HTTPS_PROXY=http://10.61.11.42:3128/"
```
