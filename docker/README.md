# 1. TỔNG QUAN

### Observability

- Là khả năng quan sát trạng thái hệ thống thông qua các dữ liệu như logs, metrics và traces. Đặc biệt quan trọng với các hệ thống phức tạp gồm hàng trăm đến hàng ngàn thành phần
- `Observability` giúp bạn có thể thực hiện dễ dàng việc:
    1. Theo dõi hiệu xuất hiện tại
    2. phân tích các sự kiện trong quá khứ
    3. phát hiện, giải quyết và dự đoán động

### Lợi ích

- các lợi ích mà `Observability` mang lại rất đa dạng và có thể áp dụng cho nhiều mục đích khác nhau:
1. Phát hiện và giải quyết sự cố nhanh chóng
2. nâng cao hiệu quả hệ thống
3. gia tăng trải nghiệm người dùng
4. nâng cao khả năng dự báo
5. Cải thiệt chất lượng sản phẩm
6. cải thiện tính khả dụng của cơ sở hạ tầng

### Triển khai Observability

- việc triển khai `Observability` một cách hiệu quả là sử dụng đúng công cụ và thiết kế cách mà hệ thống được vận hành và giám sát
1. **Intrumentation - Thiết bị đo lường**
    - Xác định và lựa chọn các công cụ nhằm thu thập Logs, Metrics, Traces từ toàn bộ hệ thống
        1. **Logs**: Ghi nhận các hoạt động quan trọng (Loki, Fluentd, Vector)
        2. **Metrics**: Thu thập các số liệu như CPU, memory, request, …. (Prometheus, Grafana Agent)
        3. **Traces**: Ghi nhận đường đi của các Request qua nhiều service (Jaeger, Tempo, Zipkin)
2. **Data correlation - liên kết và trực quan hóa dữ liệu**
    - Kết hợp Logs, Metrics, Traces để dễ dàng điều tra sự cố và quan sát hiệu năng ( Grafana)
        - Đảm bảo các service có thể trực quan hóa hoạt động và dữ liệu
        - Gắn traceID vào log và trace để dễ tra cứu (Elasticsearch, kibana)
        - Correlate dữ liệu theo thời gian và context
3. **Incident response - ứng phó sự cố**
    - Cảnh báo và phản ứng kịp thời khi có lỗi hoặc hành vi bất thường xảy ra.
        - Đảm bảo các thiết lập cảnh báo trên metrics/logs
        - Phân quyền cảnh báo theo nhóm chịu trách nhiệm (DevOps, Backend, Frontend)
        - Tích hợp với hệ thống on-call (PagerDuty, OpsGenie, Slack alerts, ….)
4. **AIOps - tự động hóa và machine learning**
    - Tối ưu hóa việc phát hiện, lọc và phân loại cảnh báo bằng Machine learning
        - Lọc “alert noise” bằng cách phát hiện mô hình thường xuyên lặp lại
        - Ưu tiên sự cố ảnh hưởng người dùng/khách hàng
        - Tự động group các sự cố liên quan

# 3. CẢI THIỆN MÔ HÌNH

### Kong

- Đóng vai trò như nhà phân phối các request đến service backend
- Trong mô hình hệ thống lớn thường sẽ bị quá tải do có nhiều truy cập vượt mức

Kong sẽ được triển khai theo 2 phương hướng chủ yếu là Horizontal Scaling và Vertical Scaling:

1. Vertical Scaling: 
    - Tăng cường tài nguyên cho từng Kong node (CPU, RAM).
    - Thích hợp khi traffic chưa quá lớn và scaling out phức tạp
    - Không phải lúc nào cũng hiệu quả, vẫn có giới hạn I/O network
2. Horizontal Scaling: 
    - Tăng số lượng instance  Kong Gateway
    - dùng Load Balancer để phân phối các request

### OpenTelemetry Collector

- Đóng vai trò thu thập dữ liệu: logs, metrics, traces từ các service
- Trong mô hình nhiều thành phần thường dễ bị quá tải khi lượng dữ liệu quá lớn

OpenTelemetry Collector sẽ có 3 phương thức Scaling chính

1. Horizontal Scaling: là cách dễ tiếp cận nhất, tăng thêm nhiều instance để có thể thu thập dễ dàng hơn.
2. Vertical Scaling: tăng cường tài nguyên cho instance, dễ dàng thực hiện  nhưng có giới hạn về mặt tài nguyên.
3. Sharding: phân vùng dữ liệu đo được từ xa, đặc biệt phù hợp với bộ dữ liệu lớn

### Loki

![MyImage](/assets/Loki_workflow.png)

- Đóng vai trò như nhà kho lưu trữ dữ liệu logs
- trong mô hình thường sẽ bị quá tải khi request lượng dữ liệu quá lớn

Loki sẽ có 2 phương thức Scaling chính là Horizontal scaling và Vertical scaling:

1. Horizontal scaling: Tăng cường tài nguyên dành cho instance Loki
2. Vertical scaling: tăng số lượng instance loki

**Các thành phần Loki**

1. `Distributor`
   - Nhận log từ client, kiểm tra hợp lệ, chuẩn hóa label, rate-limit theo tenant.
   - Chuyển log hợp lệ tới Ingester, nhân bản theo replication_factor và dùng quorum để xác nhận ghi.
   - Stateless, dễ scale, cần load balancer phía trước.

2. `Ingester`
   - Lưu log tạm trong RAM, flush định kỳ sang storage.
   - Trạng thái: PENDING, JOINING, ACTIVE, LEAVING, UNHEALTHY.
   - Log chia thành chunk, đóng và flush khi đầy/hết hạn.
   - Rủi ro mất dữ liệu nếu chết trước khi flush → khắc phục bằng replication & WAL.
   - Hỗ trợ out-of-order log, timestamp check, duplicate handling.
   - Có thể ghi ra filesystem (BoltDB, chỉ single process).

3. `Query Frontend (tùy chọn)`
   - API cho querier, tối ưu đọc dữ liệu.
   - Queueing: tránh OOM, phân phối công bằng.
   - Splitting: chia nhỏ query chạy song song.
   - Caching: kết quả metric, log query (negative cache).

4. `Query Scheduler (tùy chọn)`
   - Xếp hàng nâng cao, mỗi tenant có queue riêng.
   - Stateless, nên chạy nhiều instance (replication_factor=2).

5. `Querier`
   - Thực thi LogQL query, lấy dữ liệu từ Ingester và storage.
   - Loại bỏ trùng lặp theo timestamp, label, log message.

6. `Index Gateway (dùng với shipper stores)`
   - Xử lý metadata queries: log volume, chunk references.
   - Chạy chế độ Simple (toàn bộ index) hoặc Ring (consistent hash theo tenant).

7. `Compactor (dùng với shipper stores)`
   - Gộp file index thành index/date/tenant, tăng tốc truy vấn.
   - Quản lý retention và xóa log.
   - Thường chỉ chạy 1 instance.

8. `Ruler`
   - Quản lý rule/alert từ config, lưu trên object storage hoặc local.
   - Có thể chạy phân tán qua hash ring
   - Hỗ trợ remote evaluation qua query frontend.

**Phương thức triển khai**
1. Simple scalable mode:
    - tại phương thức triển khai này sẽ chia các thành phần của loki thành 3 phần chính:
        - Write path: gồm Distributor và ingester
        - Read path: gồm Query frontend và Querier
        - backend target: gồm Compactor, index Gateway, Query Scheduler và Ruler
2. Microservice mode
    - tại phương thức này sẽ triển khai các thành phần của loki như các service riêng biệt
