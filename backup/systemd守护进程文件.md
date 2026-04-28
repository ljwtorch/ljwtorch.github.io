## 一些systemd指令

1. 列出开机自启的进程：`systemctl list-unit-files --type=service --state=enabled`
2. 列出当前正在运行的进程：`systemctl list-units --type=service --state=running`

## Nginx

```ini
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=network.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /usr/local/openresty/nginx/conf/nginx.conf
ExecStart=/usr/sbin/nginx -c /usr/local/openresty/nginx/conf/nginx.conf
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/usr/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## Prometheus

```ini
[Unit]
Description=Prometheus service
After=network.target

[Service]
User=root
Type=simple
ExecStart=/home/xxx/monitor/prometheus/prometheus \
--web.listen-address=0.0.0.0:9090 \
--storage.tsdb.path=/etc/prometheus/data \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.retention.time=60d \
--web.enable-lifecycle \
--web.enable-admin-api
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

