```nginx
server {
    listen xxx;
    server_name xxx.xxx.cn;

    location / {
        # 设置变量，默认不允许任何请求通过
        set $allowed 0;
	# 匹配允许访问的域名（正则匹配）
        if ($http_host ~* "steamcommunity.com|steampowered.com|steamstatic.com|api.steampowered.com|repo.hex.pm|npm.taobao.org|index.ruby-china.com|gems.ruby-china.com|mirrors.cloud.tencent.com|github.com|epicgames.com") {
            set $allowed 1;
        }

        if ($allowed = 0) {
            return 403;
        }

        proxy_pass http://127.0.0.1:xxx;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

