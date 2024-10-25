# Giám sát Nginx với Prometheus và Grafana

# 1. Giới thiệu

- Nginx Prometheus Exporter là công cụ giúp thu thập các số liệu từ Nginx và xuất chúng dưới định dạng mà Prometheus có thể hiểu.
- Tài liệu này hướng dẫn bạn cách cài đặt và cấu hình Nginx Prometheus Exporter phiên bản 1.3.0 trên hệ thống Linux.

# 2. Cài đặt Nginx exporter

- Tải và Giải nén Nginx Prometheus Exporter.

Chạy lệnh sau để tải xuống phiên bản 1.3.0 của Nginx Prometheus Exporter:

```jsx
$ curl -L https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.3.0/nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz -o nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
```

- Giải nén file vừa tải:

```jsx
$ tar -zxf nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
```

- Kiểm tra phiên bản đã cài đặt:

```
$ ./nginx-prometheus-exporter --version
```

- Di chuyển file `nginx-prometheus-exporter` từ thư mục giải nén vào `/usr/local/bin` :

```jsx
$ sudo cp nginx-prometheus-exporter /usr/local/bin/nginx-prometheus-exporter
```

- Cấu hình dịch vụ Nginx exporter.

Tạo file dịch vụ cho Nginx exporter:

```jsx
$ sudo vi /etc/systemd/system/nginx-exporter.service
```

```jsx
[Unit]
Description=Nginx Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=0

[Service]
User=nobody
Group=nogroup
Type=simple
Restart=on-failure
RestartSec=5s

ExecStart=/usr/local/bin/nginx-prometheus-exporter \
    -nginx.scrape-uri=http://localhost:99/basic_status

[Install]
WantedBy=multi-user.target
```

- Reload cấu hình và start Nginx exporter:

```jsx
$ sudo systemctl daemon-reload
$ sudo systemctl restart nginx-exporter.service
$ sudo systemctl enable nginx-exporter.service
```

- Kiểm tra trạng thái dịch vụ Nginx exporter:

```jsx
$ sudo systemctl status nginx-exporter.service
```

- Cấu hình firewall (nếu có). Nếu firewall đang chạy, hãy thêm quy tắc cho port `9113`:

```jsx
$ sudo firewall-cmd --permanent --zone=public --add-port=9113/tcp
$ sudo firewall-cmd --reload
```

# 3. Cấu hình NGINX

- Cấu hình Nginx:

Trước tiên, bạn cần cấu hình Nginx để cung cấp thông tin trạng thái qua endpoint `/basic_status`
Mở file cấu hình Nginx, thường là `/etc/nginx/nginx.conf` hoặc tạo một file trong `/etc/nginx/conf.d/`  và thêm đoạn sau:

```jsx
server {
    listen 99;                          # NGINX lắng nghe trên cổng 99
    server_name localhost;              # Tên máy chủ (có thể là localhost)

    location = /basic_status {
        stub_status on;                 # Kích hoạt chế độ stub_status
        access_log off;                 # Tắt ghi log cho endpoint này
        allow 127.0.0.1;                # Chỉ cho phép truy cập từ localhost
        deny all;                       # Từ chối truy cập từ mọi IP khác
    }
}
```

- Kiểm tra cấu hình Nginx và reload lại cấu hình:

```jsx
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

$ sudo nginx -s reload
```

- Xem các số liệu Nginx:

```jsx
$ curl http://localhost:99/basic_status

Active connections: 11
server accepts handled requests
 25300599 25300599 34489517
Reading: 0 Writing: 1 Waiting: 10
```

# 4. **Cấu Hình Prometheus**

- Truy cập server Prometheus, mở file `prometheus.yml` và thêm job để thu thập số liệu từ Nginx exporter:

```jsx
scrape_configs:
  - job_name: 'Nginx-API'
    scrape_interval: 10s
    static_configs:
      - targets: ['172.18.1.11:9113','172.18.1.12:9113','172.18.1.13:9113']
    relabel_configs:
    - source_labels: [__address__]
      separator: ':'
      regex: '(.*):(.*)'
      replacement: '${1}'
      target_label: instance
```

​

- Khởi động lại Prometheus:

```jsx
sudo systemctl restart prometheus
```

# 5. Tạo Dashboard trong Grafana

- Truy cập vào **Dashboards** trên Grafana → Chọn **New** → Chọn **Import**

![image.png](image.png)

- Tải dashboard ở đây: [Nginx-Dashboard.json](https://gist.github.com/thanhtlu213/33862b089b9c6d7a3985d0ca68f98fc9)
- Sau đó, upload file JSON và nhấn **Load**.

![image.png](image%201.png)

- Cấu hình Datasource:

Sau khi nhấn **Load**, bạn sẽ thấy một màn hình cho phép bạn cấu hình datasource. Chọn datasource mà bạn đã cấu hình cho Prometheus.

![image.png](image%202.png)

- Sau khi chọn datasource, nhấn **Import** để hoàn tất quá trình.
- Dashboard mới sẽ được thêm vào danh sách dashboard của bạn.

![image.png](image%203.png)

- Xem thêm chi tiết về các hàm trong Prometheus tại: [https://prometheus.io/docs/prometheus/latest/querying/functions/](https://prometheus.io/docs/prometheus/latest/querying/functions/)

# 6. Tùy chỉnh Dashboard

- Bạn có thể sử dụng Dashboard mà tôi đã tùy chỉnh ở đây: [Nginx-Dashboard-Custom.json](https://gist.github.com/thanhtlu213/4a5ba67c74f309da458a5fcfa5e7982f)
- Ở Dashboard mặc định, bạn không thể lọc giám sát theo nhóm **job** được, mà chỉ lọc được theo các **instance**. Và sẽ gây rối mắt khi có quá nhiều host.
- Vì vậy, trong phần này, mình sẽ hướng dẫn bạn cách thêm các biến để có thể chọn các **job** và chỉ hiển thị các **instance** liên quan tới **job** đó.

### a. Thêm Biến `job` và `host` vào Dashboard

- Thêm biến `job`

Trước tiên, ta sẽ tạo 1 biến `job` để có thể lọc các Job khác nhau trong hệ thống NGINX.

- Mở dashboard và truy cập vào phần **Settings** → chọn **Variables.**
- Xóa biến `instance` và thêm một biến mới. Chọn **New variable.**
    - Name: **job**
    - Type: **Query**
    - Label: **Job**
    - Query:
    
    ```jsx
    label_values(nginx_up,job)
    ```
    
    ![image.png](image%204.png)
    
- Sau khi cấu hình xong, chọn **Apply** để lưu lại.
- Thêm biến `host`

Sau khi thêm biến `job`, ta cần tạo một biến `host` để lọc và hiển thị các **instance** tương ứng với **job** đã chọn.

- Tiếp tục tạo biến mới.
    - Name: **host**
    - Type: **Query**
    - Label: **Host**
    - Query:
    
    ```jsx
    label_values(nginx_up{job="$job"},instance)
    ```
    
    ![image.png](image%205.png)
    
- Sau khi cấu hình xong, chọn **Apply** để lưu lại.

### b. Cập nhật Query trong Panel

- Status:

```jsx
nginx_up{job="$job", instance=~"$host"}
```

Để hiển thị mỗi instance một panel, bạn cần điều chỉnh như sau:

- Mở cài đặt panel Status → Tìm mục **Panel options**
- Chọn biến mà bạn muốn lặp lại, trong trường hợp này là biến `host`

![image.png](image%206.png)

- Kết quả:

![image.png](image%207.png)

- Metrics:
    - Processed connections:
    
    ```jsx
    irate(nginx_connections_accepted{job="$job", instance=~"$host"}[5m])
    irate(nginx_connections_handled{job="$job", instance=~"$host"}[5m])
    ```
    
    ![image.png](image%208.png)
    
    - Active Connections:
    
    ```jsx
    nginx_connections_active{job="$job", instance=~"$host"}
    nginx_connections_reading{job="$job", instance=~"$host"}
    nginx_connections_waiting{job="$job", instance=~"$host"}
    nginx_connections_writing{job="$job", instance=~"$host"}
    ```
    
    ![image.png](image%209.png)
    
    - Total requests:
    
    ```jsx
    irate(nginx_http_requests_total{job="$job", instance=~"$host"}[5m])
    ```
    
    ![image.png](image%2010.png)
    

### c. Kết quả

![image.png](image%2011.png)

# 7. Thiết lập cảnh báo

- Thiết lập 2 alert cơ bản cho Nginx:
    - Alert khi có lượng HTTP request cao
    - Alert khi Nginx không hoạt động.

### a. HighNginxHttpRequests

- **Mục đích**: Cảnh báo khi Nginx nhận được nhiều hơn 20 HTTP requests mỗi giây.
- **Cấu hình**:

```jsx
- alert: HighNginxHttpRequests
  expr: '(irate(nginx_http_requests_total[1m]) > 20) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}'
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: High HTTP Requests on Nginx (instance {{ $labels.instance }})
    description: "The Nginx instance {{ $labels.instance }} is receiving more than 1 HTTP request per second.\n  Current Rate = {{ $value }} requests/sec\n  Check the application for potential spikes in traffic."

```

- **Mô tả:**
    
    **`expr`**: Sử dụng hàm `irate` để tính toán tỷ lệ HTTP requests trong vòng 1 phút và so sánh với ngưỡng 20 requests/sec.
    
    **`for`**: Alert sẽ được kích hoạt nếu điều kiện xảy ra liên tục trong 2 phút.
    
    **`labels`**: Đánh dấu mức độ nghiêm trọng là "warning".
    **`annotations`**: Cung cấp thông tin chi tiết về alert khi nó được kích hoạt.
    

### b. NginxDown

- **Mục đích**: Cảnh báo khi Nginx không hoạt động.
- **Cấu hình**:

```jsx
- alert: NginxDown
  expr: '(nginx_up == 0) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}'
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: Nginx instance is down (instance {{ $labels.instance }})
    description: "The Nginx instance {{ $labels.instance }} is currently down.\n  Nodename: {{ $labels.nodename }}\n  Check the Nginx service."
```

- **Mô tả:**
    
    **`expr`**: Kiểm tra trạng thái `nginx_up`, nếu giá trị bằng 0, điều đó có nghĩa là Nginx đang không hoạt động.
    
    **`for`**: Alert sẽ được kích hoạt ngay lập tức khi điều kiện này xảy ra.
    
    **`labels`**: Đánh dấu mức độ nghiêm trọng là "critical".
    
    **`annotations`**: Cung cấp thông tin chi tiết về trạng thái không hoạt động của Nginx.
    

### c. Tổng hợp

- Tạo file `nginx-exporter.rules`:

```jsx
$ sudo vi /etc/prometheus/alert_rules/nginx-exporter.rules
```

```jsx
groups:

- name: NginxExporter

  rules:

    - alert: HighNginxHttpRequests
      expr: '(irate(nginx_http_requests_total[1m]) > 20) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}'
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: High HTTP Requests on Nginx (instance {{ $labels.instance }})
        description: "The Nginx instance {{ $labels.instance }} is receiving more than 1 HTTP request per second.\n  Current Rate = {{ $value }} requests/sec\n  Check the application for potential spikes in traffic."

    - alert: NginxDown
      expr: '(nginx_up == 0) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}'
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: Nginx instance is down (instance {{ $labels.instance }})
        description: "The Nginx instance {{ $labels.instance }} is currently down.\n  Nodename: {{ $labels.nodename }}\n  Check the Nginx service."

```

- Thêm cấu hình `rule_files` trong `prometheus.yml`:

```jsx
rule_files:
  - "/etc/prometheus/alert_rules/nginx-exporter.rules"
```

- Khởi động lại service Prometheus:

```jsx
$ systemctl restart prometheus
```

- Nội dung thông báo về Telegram:

```jsx
🔥 Alerts Firing 🔥

🪪 HighNginxHttpRequests
📝 High HTTP Requests on Nginx (instance 172.18.1.11)
📖 The Nginx instance 172.18.1.11 is receiving more than 1 HTTP request per second.
  Current Rate = 20 requests/sec
  Check the application for potential spikes in traffic.
🏷 Labels:
  alertname: HighNginxHttpRequests
  instance: 172.18.1.11
  job: Nginx-API
  monitor: zns monitoring
  nodename: prod-api01
  severity: warning
```

```jsx
🔥 Alerts Firing 🔥

🪪 NginxDown
📝 Nginx instance is down (instance 172.18.1.11)
📖 The Nginx instance 172.18.1.11 is currently down.
  Nodename: prod-api01
  Check the Nginx service.
🏷 Labels:
  alertname: NginxDown
  instance: 172.18.1.11
  job: Nginx-API
  monitor: zns monitoring
  nodename: prod-api01
  severity: critical
```