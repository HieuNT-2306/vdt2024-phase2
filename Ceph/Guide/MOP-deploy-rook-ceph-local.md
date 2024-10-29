## A. Chuẩn bị:

### 1. Quy hoạch
- 1 node master: ceph-rook-1, 192.168.100.11
- 2 node worker: 
  - ceph-rook-2, 191.168.100.12
  - ceph-rook-3, 191.168.100.13
- Tất cả các máy đều chạy dưới quyền root.
- Mỗi máy được kết nối với 3 ổ vdi, mỗi ổ chứa 1 OSD.

### 2. Cấu hình file netplan trên 2 máy:
```
nano /etc/netplan/00-installer-config.yaml


network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 192.168.100.11/24
      routes:
        - to: default
          via: 192.168.100.1/24
  version: 2


netplan apply
```
### 3. Cấu hình file host:

```
nano /etc/hosts

127.0.0.1 localhost
127.0.1.1 ceph-rook-1
192.168.100.11 ceph-rook-1
192.168.100.12 ceph-rook-2
192.168.100.12 ceph-rook-3
```
```
192.168.100.21 rook-node-1
192.168.100.22 rook-node-2
192.168.100.32 rook-node-3
```
### 4. Cấu hình proxy (Đối với các máy kết nối ra ngoài thông qua proxy):

```
nano ~/.bashrc

#Thêm các câu lệnh dưới đây vào:

export http_proxy="http://10.61.11.42:3128"
export https_proxy="http://10.61.11.42:3128"
export NO_PROXY=localhost,127.0.0.1,::1,10.0.2.15,10.96.0.0/12,10.244.0.0/16,192.168.100.11,192.168.100.12,192.168.100.13,192.168.0.0/16

source ~/.bashrc
```
```
export http_proxy="http://10.61.11.42:3128"
export https_proxy="http://10.61.11.42:3128"
export NO_PROXY=localhost,127.0.0.1,::1,10.0.2.15,10.96.0.0/12,10.244.0.0/16,192.168.100.21,192.168.100.22,192.168.100.23,192.168.0.0/16
```

## B. Cài đặt K8S cluster:

### 1. Tắt swaping trên tất cả các node:
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
# Hoặc để tắt vĩnh viễn:
nano /etc/fstab:
#/swap.img      none    swap    sw      0       0
```
### 2. Bật IPv4 Forward:
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```

### 3. Cài đặt và chạy containerd:
```
apt-get update
apt-get install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update && sudo apt-get install containerd.io && systemctl enable --now containerd
```

### 4. Cấu hình proxy cho containerd (Nếu cần thiết):
```
mkdir /etc/systemd/system/containerd.service.d/
nano /etc/systemd/system/containerd.service.d/http-proxy.conf

#Thêm các lệnh sau vào trong file:
[Service]
Environment="HTTP_PROXY=http://10.61.11.42:3128/"
Environment="HTTPS_PROXY=http://10.61.11.42:3128/"
Environment="NO_PROXY=localhost,127.0.0.1,::1,10.96.0.0/12,10.32.0.0/12,10.244.0.0/16,10.0.2.15, 192.168.100.11, 192.168.100.12"

#Khởi động lại service:
systemctl daemon-reload
systemctl restart containerd && systemctl status containerd 
```
### 5. Cài đặt CNI Packet:
```
curl -L -o cni-plugins-linux-amd64-v1.4.0.tgz https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.0.tgz
```

### 6. Cài đặt các luật trong iptables:
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
modprobe br_netfilter
sysctl -p /etc/sysctl.conf
```

- Bật overlay và br_netfilter 
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
### 7. Chỉnh sửa file config Containerd để hỗ trợ systemd:
```
nano /etc/containerd/config.toml
```
- Dán file sau vào:
```
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    enable_selinux = false
    enable_tls_streaming = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "k8s.gcr.io/pause:3.5"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
      max_conf_num = 1

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.btrfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0
```
```
sudo systemctl restart containerd && systemctl status containerd
```
### 8. Cài đặt kubeadm, kubelet, kubectl:
```
apt-get update
  apt-get install -y apt-transport-https ca-certificates curl gpg

mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key 
(--proxy http://10.61.11.42:3128)

sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt-get update -y
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
### 9. Tạo 1 cụm cluster mới trên node master:
```
kubeadm config images pull
kubeadm init --apiserver-advertise-address=192.168.100.11 --pod-network-cidr=10.244.0.0/16
...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf  

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.11:6443 --token 03oevc.04y6ru2b2yba8xtu \
        --discovery-token-ca-cert-hash sha256:072d865c*****
```
- Sau khi tạo xong, chạy lệnh:
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
- Kiểm tra cài đặt thông qua `kubectl config view`

```
root@ceph-rook-1:~# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.100.11:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```
- Cuối cùng, trên các worker node, chạy lệnh sau để thêm vào cluster:
```
kubeadm join 192.168.100.11:6443 --token 03oevc.04y6ru2b2yba8xtu \
        --discovery-token-ca-cert-hash sha256:072d865c*****

#Chạy lệnh sau trong trường hợp không lưu lại token và cert:

kubeadm token create --print-join-command
```
- Kiểm tra thông qua lệnh `kubectl get nodes`:
```
NAME          STATUS     ROLES           AGE   VERSION
ceph-rook-1   NotReady   control-plane   48m   v1.30.4
ceph-rook-2   Ready      <none>          46m   v1.30.4
ceph-rook-3   Ready      <none>          46m   v1.30.4
```
- Ta cũng có thể đặt label cho các node thông qua lệnh:
```
kubectl label node ceph-rook-3 node-role.kubernetes.io/worker
=worker
```

### 10. Cài đặt CNI sử dụng flannel:
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
<em>**Luư ý, cài đặt CNI cho cụm cluster trước khi add thêm worker vào node!** Nếu không, dễ có trường hợp CNI sẽ gán cho các worker cùng 1 ip tại network adapter, gây ra conflict khi cố gắng cài đặt những services yêu cầu cụm cluster phải có nhiều node như rook-ceph.
</em>

<em>
Ví dụ khi không cài đặt đúng cluster, kiểm tra trên node 2 và node 3, ở adapter cni0 sẽ được gán cùng 1 địa chỉ ip:
</em>

```
#Ở trên node 2:
root@ceph-rook-2:~# ip addr show cni0
38: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 3e:db:f3:98:95:16 brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.1/24 brd 10.244.1.255 scope global cni0
       valid_lft forever preferred_lft forever
    inet6 fe80::3cdb:f3ff:fe98:9516/64 scope link
       valid_lft forever preferred_lft forever
# Ở trên node 3:
root@ceph-rook-3:~# ip addr show cni0
38: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 3e:db:f3:98:95:16 brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.1/24 brd 10.244.1.255 scope global cni0
       valid_lft forever preferred_lft forever
    inet6 fe80::3cdb:f3ff:fe98:9516/64 scope link
       valid_lft forever preferred_lft forever
```


- Trên các worker node, chạy lệnh sau để thêm vào cluster:
```
kubeadm join 192.168.100.11:6443 --token 03oevc.04y6ru2b2yba8xtu \
        --discovery-token-ca-cert-hash sha256:072d865c*****
```
- Kiểm tra thông qua lệnh `kubectl get nodes`:
```
NAME          STATUS     ROLES           AGE   VERSION
ceph-rook-1   Ready   control-plane   48m   v1.30.4
ceph-rook-2   Ready      <none>          46m   v1.30.4
ceph-rook-3   Ready      <none>          46m   v1.30.4
```
- Ta có thể thêm 
### 11. Lỗi đang gặp phải:
- Trên node ceph-rook-1, gặp phải lỗi `Network plugin returns error: cni plugin not initialized`
```
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Mon, 19 Aug 2024 08:28:17 +0000   Mon, 19 Aug 2024 08:28:17 +0000   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Mon, 19 Aug 2024 08:55:57 +0000   Mon, 19 Aug 2024 08:09:59 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Mon, 19 Aug 2024 08:55:57 +0000   Mon, 19 Aug 2024 08:09:59 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Mon, 19 Aug 2024 08:55:57 +0000   Mon, 19 Aug 2024 08:09:59 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                False   Mon, 19 Aug 2024 08:55:57 +0000   Mon, 19 Aug 2024 08:09:59 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```
- Cho dù flannel đã được cài đặt và đang chạy:
```
 kubectl get pods -n kube-flannel -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
kube-flannel-ds-kpgfz   1/1     Running   0          31m   192.168.100.13   ceph-rook-3   <none>           <none>
kube-flannel-ds-nnm7h   1/1     Running   0          31m   192.168.100.12   ceph-rook-2   <none>           <none>
kube-flannel-ds-zqmq2   1/1     Running   0          31m   192.168.100.11   ceph-rook-1   <none>           <none>
```

- Fix lỗi:
```
systemctl stop apparmor
systemctl disable apparmor
systemctl restart containerd.service
```
<em>Lưu ý, AppArmor là một mô-đun bảo mật của kernel Linux cho phép quản trị viên hệ thống hạn chế khả năng của các chương trình thông qua các hồ sơ riêng cho từng chương trình. Các hồ sơ này có thể cho phép các quyền như truy cập mạng, truy cập socket thô, và quyền đọc, ghi hoặc thực thi các tệp ở các đường dẫn phù hợp. Cần cấu hình AppArmor nếu  muốn cho phép các dịch vụ k8s hoạt động một cách an toàn, không nên disable như trong cụm lab này.</em>

### 12.Reset cụm k8s trong trường hợp cấp thiết:
- Chạy các lênh sau để reset toàn bộ cụm lệnh k8s:
```
kubeadm reset
rm -f $HOME/.kube/config
rm -rf /etc/cni/net.d
#Nếu có các config riêng trong iptables thì hãy bỏ qua 3 lệnh dưới đây.
sudo iptables -F       
sudo iptables -t nat -F 
sudo iptables -X       
```


## C. Cài đặt rook-ceph:
- Tải các file manifest cần thiết:
```
git clone --single-branch --branch v1.14.9 https://github.com/rook/rook.git
cd rook/deploy/examples
```
- Apply các file manifest để tạo pod ceph-operator
```
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```
- Kiểm tra pod chạy thông qua:
```
kubectl -n rook-ceph get pod
NAME                                            READY   STATUS      RESTARTS        AGE
rook-ceph-operator-7d5565fbc7-25m4x             1/1     Running     0               1m23s
```
- Tạo 1 cluster thông qua:
```
kubectl create -f <cluster-file>.yaml
```