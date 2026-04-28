> 对 `docker daemon` 配置额外的环境变量，来使 `docker pull` 指令经过网络代理访问
>
> 参考文档：[DockerDocs:Daemon proxy configuration](https://docs.docker.com/engine/daemon/proxy/#environment-variables)

```shell
# 使用systemd的系统中，添加额外的配置文件
mkdir -p /etc/systemd/system/docker.service.d

cat << EOF >> /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment=HTTP_PROXY=http://xxxx:41001
Environment=HTTPS_PROXY=http://xxxx:41001
Environment="NO_PROXY=localhost,127.0.0.1,xxxx"
EOF

systemctl daemon-reload
systemctl reload docker
systemctl restart docker
```