### Deploy Kolla - Ansible:
### 1. Tải các thư viện cần thiết
```
sudo apt update
sudo apt upgrade -y
sudo apt install python3-dev libffi-dev gcc libssl-dev python3-pip git -y
sudo apt install ansible -y # Tải bản 2.16 trở lên
```
### 2. Tải kolla-ansible thông qua pip:
```
sudo pip3 install kolla-ansible
```
### 3. Tạo thư mục cho kolla-ansible:
```
sudo mkdir -p /etc/kolla
sudo chown 777 /etc/kolla
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
```
### 4. Cấu hình cho Kolla-ansible:
- Tạo 1 passwd:
```
kolla-genpwd
```
- Cấu hình cho kolla ansible thông qua file global.yml, ở Lab này, ta sẽ cấu hình trước các service ceph radosgw, cinder, nova: 
```
...
kolla_internal_vip_address: "10.208.190.189"
network_interface: "eth0"
neutron_external_interface: "eth2"
...
kolla_base_distro: "ubuntu"
...
enable_ceph_rgw: "yes"
enable_ceph_rgw_loadbalancer: "no"
enable_cinder: "yes"
enable_cinder_backup: "yes"
...
#Cinder
ceph_cinder_user: "cinder"
ceph_cinder_keyring: "client.{{ ceph_cinder_user }}.keyring"
ceph_cinder_pool_name: "volumes"
ceph_cinder_backup_user: "cinder-backup"
ceph_cinder_backup_keyring: "client.{{ ceph_cinder_backup_user }}.keyring"
ceph_cinder_backup_pool_name: "backups"
#Nova
ceph_nova_keyring: "{{ ceph_cinder_keyring }}"
ceph_nova_user: "{{ ceph_cinder_user }}"
ceph_nova_pool_name: "vms"
...
# Enable / disable Cinder backends
cinder_backend_ceph: "yes"
cinder_backend_vmwarevc_vmdk: "yes"
cinder_backend_vmware_vstorage_object: "yes"
cinder_volume_group: "cinder-volumes"
...
```

### 5. Tạo các keyring cho các dịch vụ cinder và nova để giao tiếp được với cụm ceph:
```
#Cinder
ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes'
ceph auth get-or-create client.cinder -o /etc/ceph/ceph.client.cinder-backup.keyring
sudo cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume/
...
#Cinder-backup
ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes'
ceph auth get-or-create client.cinder-backup -o /etc/ceph/ceph.client.cinder-backup.keyring
sudo cp /etc/ceph/ceph.client.cinder-backup.keyring /etc/kolla/config/cinder/cinder-backup/
...
#Nova
cp /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring /etc/kolla/config/nova/localhost/ceph.client.cinder.keyring
cp /etc/kolla/config/cinder/cinder-volume/ceph.client.cinder.keyring /etc/kolla/config/nova/ceph.client.cinder.keyring
```


### 6. Triển khai cụm Openstack:

- Tải các dependency trong kolla-ansible:
```
kolla-ansible install-deps
```
- Khởi tạo các máy chủ trước khi tiến hành triển khai OpenStack.
```
kolla-ansible -i ./all-in-one bootstrap-servers
```
- Kiểm tra các yêu cầu hệ thống, cấu hình và các điều kiện tiên quyết
```
kolla-ansible -i ./all-in-one prechecks
```
- Triển khai cụm openstack:
```
kolla-ansible -i ./all-in-one deploy
```
- Cài đặt OpenStack CLI client:

```
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master

```
- Tạo file clouds.yaml chứa thông tin đăng nhập người dùng admin:
```
kolla-ansible post-deploy
```
- Kiểm tra kết quả:
```
cat /etc/kolla/admin-openrc.sh
for key in $( set | awk '{FS="="}  /^OS_/ {print $1}' ); do unset $key ; done
export OS_PROJECT_DOMAIN_NAME='Default'
export OS_USER_DOMAIN_NAME='Default'
export OS_PROJECT_NAME='admin'
export OS_TENANT_NAME='admin'
export OS_USERNAME='admin'
export OS_PASSWORD='TblZyEDlaWk5RBfL3ZO2U5OmwngyCE6ZPx8JF91p'
export OS_AUTH_URL='http://10.208.190.189:5000'
export OS_INTERFACE='internal'
export OS_ENDPOINT_TYPE='internalURL'
export OS_IDENTITY_API_VERSION='3'
export OS_REGION_NAME='RegionOne'
export OS_AUTH_PLUGIN='password'
```