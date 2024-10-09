# Hướng dẫn triển khai Grafana và Prometheus bằng Docker

date: October 5, 2024
slug: install-grafana-prometheus-with-docker
author: Thành Nguyễn
status: Public
tags: Prometheus
summary: Thiết lập một môi trường giám sát nhanh chóng với Prometheus và Grafana qua Docker Compose
type: Post
updatedAt: October 5, 2024 10:45 PM
category: Monitor

# 1. Cài đặt **Docker và Docker Compose**

- Cài đặt Docker bằng script:

```jsx
$ sudo curl -fsSL https://get.docker.com/ | sh
$ sudo systemctl status docke
$ sudo systemctl enable docker
```

- Cấu hình network cho Docker:

```jsx
$ sudo bash -c 'touch /etc/docker/daemon.json' && sudo bash -c "echo -e \"{\n\t\\\"bip\\\": \\\"55.55.1.1/24\\\"\n}\" > /etc/docker/daemon.json"
```

- Cài đặt Docker Compose:

```jsx
$ sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
```

# 2. Cài đặt Prometheus và Grafana với Docker Compose

- Clone repository chứa cấu hình Docker Compose cho Prometheus và Grafana về server. Mở terminal và chạy lệnh sau:

```jsx
$ git clone git@github.com:thanhtlu213/Grafana-Prometheus-Docker.git
$ cd Grafana-Prometheus-Docker
$ sudo docker-compose up -d
```

- Kiểm tra trạng thái container:

```jsx
$ docker ps -a
CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS                 PORTS                    NAMES
70df95287d0a   prom/alertmanager:v0.27.0   "/bin/alertmanager -…"   5 hours ago   Up 5 hours             0.0.0.0:9093->9093/tcp   alertmanager
b0198d7e981d   grafana/grafana:11.2.2      "/run.sh"                5 hours ago   Up 5 hours             0.0.0.0:3000->3000/tcp   grafana
cd89ef71e1b0   prom/prometheus:v2.53.2     "/bin/prometheus --c…"   5 hours ago   Up 5 hours             0.0.0.0:9090->9090/tcp   prometheus
e01e1fc12dbe   prom/pushgateway:v1.10.0    "/bin/pushgateway"       5 hours ago   Up 5 hours             0.0.0.0:9091->9091/tcp   pushgateway
17c8718c0b3e   prom/node-exporter:v1.8.2   "/bin/node_exporter …"   5 hours ago   Up 5 hours             0.0.0.0:9100->9100/tcp   nodeexporter
```

- Truy cập Grafana với đường dẫn `http://<IP-server>:3000/`  với thông tin đăng nhập sau:
    - User: **admin**
    - Password: **admin**