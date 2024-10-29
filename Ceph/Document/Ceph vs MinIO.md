# So Sánh giữa MinIO và CEPH-ROOKCEPH.

## I. Bảng so sánh các tiêu chí giữa CEPH và MinIO


| **Tiêu chí**                          | **Ceph**                      | **MinIO**                                 |
|---------------------------------------|-------------------------------------------| -------------------------------------------|
| **Kiến trúc**                         | Ceph là hệ thống lưu trữ thống nhất hỗ trợ lưu trữ khối, tệp và đối tượng, Ceph cũng có thể triển khai trên k8s thông qua Ceph Rook. | MinIO tập trung vào lưu trữ đối tượng, nhẹ, hướng đến đám mây và tương thích với API S3. |
| **Tính linh hoạt trong triển khai**   | Ceph có thể triển khai trên phần cứng thường hoặc đám mây, yêu cầu cấu hình phức tạp, hoặc tự động hóa các quá trình triển khai thông qua rook trên k8s. | MinIO dễ dàng triển khai, phù hợp cho container và đám mây, hoặc triển khai trực tiếp bare-metal.Minio cũng hỗ trợ triển khai nhanh chóng dưới dạng instance hoặc cụm phân tán. |
| **Khả năng mở rộng và hiệu suất**     | Ceph có khả năng mở rộng cao, hỗ trợ lưu trữ phân tán trên nhiều nút với hiệu suất song song tốt. Ceph-Rook tận dụng khả năng mở rộng của Kubernetes. | MinIO tối ưu hóa hiệu suất cho khối lượng công việc lưu trữ đối tượng, sử dụng mã hóa xóa và tải lên song song. |
| **Bảo vệ và dư thừa dữ liệu**         | Ceph cung cấp sao chép và EC để bảo vệ dữ liệu, kết hợp với hệ thống crushmap giúp phân bổ dữ liệu đều, làm tăng tính HA. Ceph-Rook kế thừa tính năng bảo vệ này từ Ceph. | MinIO sử dụng EC hoặc sao chép đối tượng để đảm bảo dữ liệu an toàn và chịu lỗi, MinIO không có hệ thống crush map, thay vào nó, nó sẽ tạo các Erasure set để phân bổ đều các drives vào trong các ES. |
| **Tính tương thích API**              | Ceph cung cấp API RADOS, hỗ trợ nhiều ngôn ngữ lập trình. Ceph-Rook cũng hỗ trợ các API này thông qua Kubernetes. | MinIO hoàn toàn tương thích với API S3, dễ dàng tích hợp với các ứng dụng và công cụ S3. |
| **Cộng đồng và hệ sinh thái**         | Ceph có cộng đồng lớn, hỗ trợ mạnh mẽ từ các tổ chức lớn. Ceph-Rook có cộng đồng Kubernetes hỗ trợ triển khai Ceph. | MinIO có cộng đồng hoạt động mạnh mẽ và tích hợp tốt với hệ sinh thái cloud-native và các công cụ tương thích với S3. |
| **Các loại lưu trữ được hỗ trợ**      | Ceph & Ceph-Rook hỗ trợ lưu trữ khối, tệp và đối tượng. | MinIO chỉ hỗ trợ lưu trữ đối tượng. |

## II. Kết Luận:

- **Ceph** là một giải pháp lưu trữ hỗ trợ nhiều loại lưu trữ khác nhau, với khả năng mở rộng tốt, triển khai linh hoạt trên các nền tảng khác nhau. Tính chịu lỗi và bảo vệ dữ liệu cao thông qua CRUSH MAP, cùng với một cộng đồng mạnh mẽ. Tuy nhiên, cấu hình và quản trị của Ceph lại phức tạp, đòi hỏi quản trị viên có kiến thức sâu về hạ tầng và cấu hình. Hiệu năng sẽ không tối ưu được cho việc lưu trữ đối tượng, và yêu cầu tài nguyên của Ceph là không nhỏ. 
- Ta nên lựa chọn Ceph khi:
  - Ta cần lưu trữ nhiều loại dữ liệu khác nhau (khối, tệp, đối tượng)
  - Ta có một hệ thống thích hợp cho các tổ chức có quy mô lớn, yêu cầu khả năng mở rộng và tính năng chịu lỗi mạnh mẽ
  - Ta cần một hệ thống có tính linh hoạt cao, có thể điều chỉnh và mở rộng dễ dàng trên nhiều hạ tầng.

- **MinIO** là một giải pháp lưu trữ tập trung vào lưu trữ đối tượng, với hiệu xuất cao, triển khai dễ dàng, tích hợp lên trên k8s và độ tương thích lớn với API S3. Tuy nhiên, hệ thống này sẽ không hỗ trợ lưu trữ tệp hay lưu trữ khối, với một cộng đồng nhỏ hơn, và tính HA sẽ thấp hơn do không có hệ thống CRUSH map để phân tán đều các EC.

- Ta nên lựa chọn MinIO khi:
    - Ta chỉ cần một hệ thống lưu trữ đối tượng, và yêu cầu hiệu năng cao.
    - Ta ưu tiên việc triển khai nhanh và đơn giản.
    - Ta làm việc nhiều với môi trường k8s, cloud-native và API S3.
    - Thích hợp hơn cho các doanh nghiệp vừa và nhỏ.
## III. Tài liệu tham khảo:

- So sánh giữa MinIO và CEPH: https://stackshare.io/stackups/ceph-vs-minio#:~:text=Ceph%20is%20a%20unified%20solution,compatible%20with%20Amazon%20S3%20APIs.

- So sánh giữa MinIO và Ceph-Rook: https://stackshare.io/stackups/minio-vs-rook#:~:text=Data%20Resilience%3A%20Minio%20ensures%20data,Ceph%20to%20provide%20data%20resilience.

- MinIO vs Red Hat Ceph Storage comparison: https://www.peerspot.com/products/comparisons/minio_vs_red-hat-ceph-storage

