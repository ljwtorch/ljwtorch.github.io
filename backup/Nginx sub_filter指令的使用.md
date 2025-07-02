### 配置：

 ```nginx
location / {
        sub_filter 'https://static.xxxx.hk/partner/assets' '/build';
        sub_filter_once off;
        sub_filter_types *;
        proxy_set_header Accept-Encoding "";
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        include vhost/ipdb;
        proxy_buffering on;
        proxy_pass http://192.168.8.100:3343;
}
 ```

1. **`sub_filter`**
   - 将响应内容中的字符串 `https://static.sonkwo.hk/partner/assets` 替换为 `/build`。
   - 场景：修改后端返回的 HTML/CSS/JS 中的静态资源路径，使其指向代理服务器本地路径或其他地址。
2. **`sub_filter_once off`**
   - 默认情况下，`sub_filter` 只替换第一次匹配的内容。设为 `off` 会替换所有匹配项。
3. **`sub_filter_types \*`**
   - 默认仅替换 `text/html` 类型的内容。通过 `*` 允许对所有类型的响应（如 CSS/JS/JSON）进行替换。
4. **`proxy_set_header Accept-Encoding ""`**
   - 禁止客户端使用压缩（如 gzip），因为压缩后的内容无法直接替换字符串。

### 用途：

1. 静态资源路径重写

- 问题：后端返回的页面中，静态资源（如图片、JS）链接指向了原始域名，但需要通过代理服务器重新路由；
- 解决：将原始路径替换为代理服务器的本地路径，避免跨域或路径错误；

2. CDN/代理层适配

- 当后端服务与代理服务器部署在不同环境时，通过替换内容统一资源路径；

3. 调试或本地开发

- 在开发环境中，将生产环境的资源路径替换为本地路径；