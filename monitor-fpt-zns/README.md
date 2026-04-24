# prometheus-notes

## Dashboard provisioning

Các dashboard dưới đây được sử dụng theo cơ chế **Grafana provisioning**.
 
> Cần copy vào đúng thư mục provisioning/dashboard trên server Grafana.

## Danh sách dashboard

- [Blackbox-Exporter-HTTP-Prober-Custom.json](./grafana/provisioning/dashboards/Blackbox-Exporter-HTTP-Prober-Custom.json)
- [Dashboard-Monitoring-All-Server-Overview.json](./grafana/provisioning/dashboards/Dashboard-Monitoring-All-Server-Overview.json)
- [Server-Performance-Monitor-Node-Exporter.json](./grafana/provisioning/dashboards/Server-Performance-Monitor-Node-Exporter.json)
- [Zabbix-System-Overview-provisioning.json](./grafana/provisioning/dashboards/Zabbix-System-Overview-provisioning.json)
- [grafana-zabbix-overview-linux-nginx-phpfpm-provisioning.json](./grafana/provisioning/dashboards/grafana-zabbix-overview-linux-nginx-phpfpm-provisioning.json)
- [Nginx-Dashboard-Custom.json](./grafana/provisioning/dashboards/Nginx-Dashboard-Custom.json)
- [HAProxy-Overview-Custom.json](./grafana/provisioning/dashboards/HAProxy-Overview-Custom.json)
- [RabbitMQ-Overview-Custom.json](./grafana/provisioning/dashboards/RabbitMQ-Overview-Custom.json)
- [Redis-Overview-Custom.json](./grafana/provisioning/dashboards/Redis-Dashboard-Custom.json)
- [MySQL-Overview-Custom.json](./grafana/provisioning/dashboards/MySQL-Overview-Custom.json)

## Cách sử dụng

Đảm bảo các file dashboard được đặt trong thư mục:

```bash
./grafana/provisioning/dashboards/
```

Grafana sẽ tự động load dashboard khi container/service được khởi động lại:

```bash
docker compose restart grafana
```

---

## Lưu ý về datasource

Các dashboard provisioning cần datasource có **UID cố định**.

### Ví dụ cấu hình

```json
"datasource": {
  "type": "prometheus",
  "uid": "prometheus"
}
```

hoặc:

```json
"datasource": {
  "type": "alexanderzobnin-zabbix-datasource",
  "uid": "zabbix-datasource"
}
```

---

## Các lỗi thường gặp

Nếu UID datasource trên Grafana không khớp, dashboard có thể bị lỗi:

- Datasource not found  
- No data  
- Panel không hiển thị dữ liệu  

---

## Khuyến nghị

Không nên sử dụng UID random do Grafana tự sinh, ví dụ:

```json
"uid": "PBFA97CFB590B2093"
```

Nên chuẩn hóa UID datasource trong file provisioning:

```yaml
uid: prometheus
uid: zabbix-datasource
```