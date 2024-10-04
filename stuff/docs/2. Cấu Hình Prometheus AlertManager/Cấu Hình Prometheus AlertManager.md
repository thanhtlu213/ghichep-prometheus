# Cấu Hình Prometheus AlertManager

date: September 26, 2024
slug: install-prometheus-alertmanager
author: Thành Nguyễn
status: Public
tags: Prometheus
summary: Tìm hiểu cách cài đặt và cấu hình AlertManager để xử lý các cảnh báo được gửi từ máy chủ Prometheus.
type: Post
updatedAt: October 3, 2024 2:16 PM
category: Monitor

# 1. Cài đặt **AlertManager**

- Tạo `user` và `folder` cần thiết:

```jsx
sudo groupadd -f alertmanager
sudo useradd -g alertmanager --no-create-home --shell /usr/sbin/nologin alertmanager
sudo mkdir -p /etc/alertmanager/templates
sudo mkdir /var/lib/alertmanager
sudo chown -R alertmanager:alertmanager /etc/alertmanager
sudo chown -R alertmanager:alertmanager /var/lib/alertmanager
```

- Tải gói cài đặt và giải nén:

```jsx
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
```

```jsx
tar -xvf alertmanager-0.27.0.linux-amd64.tar.gz
mv alertmanager-0.27.0.linux-amd64 alertmanager-files
```

- Cài đặt Prometheus AlertManager.

Sao chép các file  `alertmanager` và `amtool` từ thư mục đã giải nén vào `/usr/bin` và thay đổi quyền sở hữu:

```jsx
sudo cp alertmanager-files/alertmanager /usr/bin/
sudo cp alertmanager-files/amtool /usr/bin/
sudo chown alertmanager:alertmanager /usr/bin/alertmanager
sudo chown alertmanager:alertmanager /usr/bin/amtool
```

- Cấu hình file config Alertmanager.

Di chuyển file `alertmanager.yml` từ thư mục đã giải nén vào thư mục cấu hình và thay đổi quyền sở hữu:

```jsx
sudo cp alertmanager-files/alertmanager.yml /etc/alertmanager/alertmanager.yml
sudo chown alertmanager:alertmanager /etc/alertmanager/alertmanager.yml
```

- Cấu hình dịch vụ AlertManager.

Tạo file dịch vụ cho AlertManager:

```
sudo vi /usr/lib/systemd/system/alertmanager.service
```

```jsx
[Unit]
Description=AlertManager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/bin/alertmanager \
    --config.file /etc/alertmanager/alertmanager.yml \
    --storage.path /var/lib/alertmanager/

[Install]
WantedBy=multi-user.target
```

```jsx
sudo chmod 664 /usr/lib/systemd/system/alertmanager.service
```

- Reload cấu hình và start AlertManager:

```jsx
sudo systemctl daemon-reload
sudo systemctl start alertmanager.service
sudo systemctl enable alertmanager.service
```

- Kiểm tra trạng thái dịch vụ AlertManager:

```jsx
sudo systemctl status alertmanager.service
```

- Cấu hình firewall (nếu có). Nếu firewall đang chạy, hãy thêm quy tắc cho port `9093`:

```jsx
sudo firewall-cmd --permanent --zone=public --add-port=9093/tcp
sudo firewall-cmd --reload
```

# 2. Cấu hình **AlertManager**

- Mở file `/etc/alertmanager/alertmanager.yml` và sửa lại như sau:

```jsx
# See https://github.com/prometheus/alertmanager/pull/2827
# https://prometheus.io/docs/alerting/latest/configuration/#telegram_config
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 5m
  receiver: 'telegram'
  
receivers:
  - name: 'telegram'
    telegram_configs:
    - api_url: https://api.telegram.org
      bot_token: 'YOUR_BOT_TOKEN'
      chat_id: -123456
      
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
    
 templates:
  - '/etc/alertmanager/templates/*.tmpl'
```

- Kiểm tra cấu hình AlertManager với `amtool` :

```jsx
amtool check-config /etc/alertmanager/alertmanager.yml
```

# 3. Cấu hình template

- Template default:  [default.tmpl](https://github.com/prometheus/alertmanager/blob/main/template/default.tmpl)
- Tạo file template `/etc/alertmanager/templates/alert.tmpl` với nội dung sau:

```jsx
{{ define "__thanhnv_text_alert_list" }}{{ range . }}
🪪 <b>{{ .Labels.alertname }}</b>
{{- if .Annotations.summary }}
📝 {{ .Annotations.summary }}{{ end }}
{{- if .Annotations.description }}
📖 {{ .Annotations.description }}{{ end }}
🏷 Labels:
{{ range .Labels.SortedPairs }}  <i>{{ .Name }}</i>: <code>{{ .Value }}</code>
{{ end }}{{ end }}
{{ end }}

{{ define "telegram.thanhnv.message" }}
{{ if gt (len .Alerts.Firing) 0 }}
🔥 Alerts Firing 🔥
{{ template "__thanhnv_text_alert_list" .Alerts.Firing }}
{{ end }}
{{ if gt (len .Alerts.Resolved) 0 }}
✅ Alerts Resolved ✅
{{ template "__thanhnv_text_alert_list" .Alerts.Resolved }}
{{ end }}
{{ end }}
```

- Cập nhật file cấu hình Alertmanager.

Mở file `/etc/alertmanager/alertmanager.yml`, trong phần cấu hình `route` hoặc `receivers`, thêm dòng sau để gửi thông báo sử dụng template:

```jsx
message: '{{ template "telegram.thanhnv.message" . }}'
```

- File `/etc/alertmanager/alertmanager.yml` hoàn chỉnh:

```jsx
# See https://github.com/prometheus/alertmanager/pull/2827
# https://prometheus.io/docs/alerting/latest/configuration/#telegram_config
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 15m
  receiver: 'telegram'
  
receivers:
  - name: 'telegram'
    telegram_configs:
    - api_url: https://api.telegram.org
      bot_token: 'YOUR_BOT_TOKEN'
      chat_id: -123456
      message: '{{ template "telegram.thanhnv.message" . }}'
      
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
    
 templates:
  - '/etc/alertmanager/templates/*.tmpl'
```

- Khởi động lại service:

```jsx
sudo systemctl restart alertmanager.service
```

- Message nhận được qua Telegram:

```jsx
🔥 Alerts Firing 🔥

🪪 RabbitmqTooManyConnections
📝 RabbitMQ too many connections (instance 10.150.130.27)
📖 The total connections of a node is too high
  VALUE = 1736
🏷 Labels:
  alertname: RabbitmqTooManyConnections
  instance: 10.150.130.27
  job: cloud-rmq-01
  monitor: RabbitMQ
  severity: warning
```