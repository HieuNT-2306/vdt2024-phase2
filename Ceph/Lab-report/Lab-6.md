# Thử nghiệm với Cephadm

## 1. Cài đặt và tạo 1 bản cephadm riêng:
- Đầu tiên, ta cần clone repo của ceph về:
```
git clone https://github.com/ceph/ceph.git
cd ceph
```
- Cephadm được viết bằng python, nên trước hết, ta cần phải tải các python compiler về và tạo 1 môi trường ảo:
```
sudo apt-get update
sudo apt-get install python3 python3-venv python3-pip
python3 -m venv venv
source venv/bin/activate
```
- Tải các dependency cần thiết:
```
pip install pyyaml
pip install jinja2
```
- Sau đó, ta có thể chạy cephadm thông qua python3:
```
python3 -m cephadm -h
```

## 2. CephadmContext:
```
from cephadmlib.context import CephadmContext
```
- CephadmContext hay `ctx` là một object chứa các thông tin và trạng thái cần thiết để thực hiện các thao tác liên quan đến Ceph.
- Nó lưu trữ toàn bộ các thông tin cần thiết về các thao tác của `cephadm`, và nó cần thiết để các function có thể tham chiếu và lấy các giá trị từ nó ra để thực hiện các tác vụ khác, VD về 1 file `ctx` sau khi tạo một cụm ceph bằng `cephadm bootstrap` sẽ trông như sau:
```
{
    "_args": {
        "image": null,
        "docker": false,
        "data_dir": "/var/lib/ceph",
        "log_dir": "/var/log/ceph",
        "logrotate_dir": "/etc/logrotate.d",
        "sysctl_dir": "/etc/sysctl.d",
        "unit_dir": "/etc/systemd/system",
        "verbose": false,
        "log_dest": null,
        "timeout": null,
        "retry": 15,
        "env": [],
        "no_container_init": false,
        "no_cgroups_split": false,
        "config": null,
        "mon_id": null,
        "mon_addrv": null,
        "mon_ip": "172.26.224.220",
        "mgr_id": null,
        "fsid": "ed3d02ca-6fed-11ef-b972-9faeee99031f",
        "output_dir": "/etc/ceph",
        "output_keyring": "/etc/ceph/ceph.client.admin.keyring",
        "output_config": "/etc/ceph/ceph.conf",
        "output_pub_ssh_key": "/etc/ceph/ceph.pub",
        "skip_admin_label": false,
        "skip_ssh": false,
        "initial_dashboard_user": "admin",
        "initial_dashboard_password": null,
        "ssl_dashboard_port": 8443,
        "dashboard_key": null,
        "dashboard_crt": null,
        "ssh_config": null,
        "ssh_private_key": null,
        "ssh_public_key": null,
        "ssh_signed_cert": null,
        "ssh_user": "root",
        "skip_mon_network": false,
        "skip_dashboard": false,
        "dashboard_password_noupdate": false,
        "no_minimize_config": false,
        "skip_ping_check": false,
        "skip_pull": false,
        "skip_firewalld": false,
        "allow_overwrite": false,
        "no_cleanup_on_failure": false,
        "allow_fqdn_hostname": false,
        "allow_mismatched_release": false,
        "skip_prepare_host": false,
        "orphan_initial_daemons": false,
        "skip_monitoring_stack": false,
        "with_centralized_logging": false,
        "apply_spec": null,
        "shared_ceph_folder": null,
        "registry_url": null,
        "registry_username": null,
        "registry_password": null,
        "registry_json": null,
        "container_init": true,
        "cluster_network": null,
        "single_host_defaults": false,
        "log_to_file": false,
        "deploy_cephadm_agent": false,
        "custom_prometheus_alerts": null,
        "func": "<function command_bootstrap at 0x7f04d5e0d090>"
    },
    "_conf": "<cephadmlib.context.BaseConfig object at 0x7f04d5e04d60>",
    "error_code": 0,
    "meta_properties": {
        "service_name": "mgr",
        "memory_request": null,
        "memory_limit": null,
        "ports": [
            9283,
            8765,
            8443
        ]
    }
}
```
- Nó sẽ chứa các thông tin cần thiết khi tạo 1 cụm ceph, tương tự với `cephadm rm-cluster` hay bất cứ các lệnh khác, các trường sẽ phụ thuộc vào các `parser` hay các tham số dòng lệnh được truyền vào.

```
...
def cephadm_init_ctx(args: List[str]) -> CephadmContext:
    ctx = CephadmContext()
    ctx.set_args(_parse_args(args))
    return ctx
...

def main() -> None:
    av: List[str] = []
    av = sys.argv[1:]
    ctx = cephadm_init_ctx(av)
```
## 3. Tạo 1 hàm mới trong cephadm:
- Tạo 1 hàm mới:
```
def command_custom_function(ctx):
    ...
    return 0
```
- Sau đó, ta cần phải tạo 1 `parser` cho hàm:
```
    #Custom parser
parser_custom = subparsers.add_parser(
    'custom', help='FOR TESTING PURPOSE - Write out context for ceph')
parser_custom.set_defaults(func=command_context_write)
parser_custom.add_argument(
    '--value', 
    help="A test value for custom parser",
    type=int,
    default=1
)
```
- Với `help` là trường để giải nghĩa cho hàm khi ta gọi cờ `-h`
- Với mỗi 1 argument mới, ta cần phải thêm bằng `add_argument` để chỉnh kiểu, giá trị mặc định, định nghĩa,....
- Và để gọi argument đó ra, ta cần phải gọi thông qua context như `ctx.value`.