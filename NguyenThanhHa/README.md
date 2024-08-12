# Xây dựng giải pháp Cloud Backup-as-a-Service với Management Agents
| Tuần                         | Task                                         | Output                                                                                                                                             | Readme                                                |
|------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Tuần 1<br>05/08 - 11/08      | - Tìm hiểu chung về Backup, Backup agent      | - Hiểu được các khái niệm cơ bản: Backup là gì, Cơ chế hoạt động của Backup<br>- Backup agent là gì, dùng để làm gì, nhiệm vụ của backup trong mô hình BaaS |  [README WEEK1](./Week1/README.md)                   |
| Tuần 2<br>12/08 - 18/08      | - Tìm hiểu các provider cung cấp BaaS: Veeam, Veritas, Bizfly | - Hiểu được cơ bản mô hình hoạt động của dịch vụ BaaS: gồm những thành phần nào, control plane tổ chức ntn, dataplane tổ chức ntn, các thành phần kết nối ntn, ...<br>- Sử dụng thử Veeam, Veritas, Bizfly => hiểu được cách hoạt động |  [README WEEK2](./Week2/README.md)                   |
| Tuần 3 + 4<br>19/08 - 01/09  | - Thử nghiệm PA xây dựng data plane          | - Lấy được File, Folder => đẩy lên repos lưu trữ (S3, Local, ...)<br>- Restore File và vẫn đảm bảo kết quả sau khi restore giống như tại thời điểm thực hiện backup |  [README WEEK3_4](./Week3_4/README.md)               |
| Tuần 5 + 6<br>02/09 - 15/09  | - Hoàn thiện hoàn chỉnh Data Plane            | - Code xong Backup agent, đảm bảo agent collect dữ liệu và get dữ liệu về để restore                                                                |  [README WEEK5_6](./Week5_6/README.md)               |
| Tuần 7 + 8<br>16/09 - 29/09  | - Thử nghiệm, xây dựng control plane         | - Phương án, công cụ quản lý backup:<br>  + stop, start backup agent<br>  + create backup, restore backup<br>  + lập lịch backup, ...              |  [README WEEK7_8](./Week7_8/README.md)               |
| Tuần 9 + 10<br>30/09 - 13/10 | - Hoàn thiện Control Plane                   | - Code xong Control Plane<br>- Đảm bảo luồng backup hoạt động cơ bản                                                                                |  [README WEEK9_10](./Week9_10/README.md)             |
| Tuần 11                      | - Nghiên cứu cải thiện hệ thống, nâng cao bảo mật, hiệu suất | - sử dụng SSL certificate cho backup server<br>- Incremental backup, thin, thick backup, ...<br>- Thiết kế quy hoạch msg queue xử lý các task backup tuần tự tránh tắc nghẽn, miss request, lỗi hệ thống. |  [README WEEK11](./Week11/README.md)                 |