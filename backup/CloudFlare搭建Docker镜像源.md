## CloudFlare搭建Docker镜像源

> [示例网站](https://docker.3mz.cloudns.ch/) 使用的 [教程](https://blog.lty520.faith/%E5%8D%9A%E6%96%87/%E8%87%AA%E5%BB%BAdocker-hub%E5%8A%A0%E9%80%9F%E9%95%9C%E5%83%8F) 和 [仓库](https://github.com/Doublemine/container-registry-worker)，对仓库代码进行了部分自定义修改

`docker.ts`

```typescript
import HTML from './docker.html';

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const originalHost = request.headers.get("host");
    const registryHost = "registry-1.docker.io";

    if (path.startsWith("/v2/")) {
      const headers = new Headers(request.headers);
      headers.set("host", registryHost);

      const registryUrl = `https://${registryHost}${path}`;
      const registryRequest = new Request(registryUrl, {
        method: request.method,
        headers: headers,
        body: request.body,
        redirect: "follow", // 按照教程修改了这一行
      });

      const registryResponse = await fetch(registryRequest);

      const responseHeaders = new Headers(registryResponse.headers);
      responseHeaders.set("access-control-allow-origin", originalHost as string);
      responseHeaders.set("access-control-allow-headers", "Authorization");
      return new Response(registryResponse.body, {
        status: registryResponse.status,
        statusText: registryResponse.statusText,
        headers: responseHeaders,
      });
    } else {
      return new Response(HTML.replace(/{{host}}/g, originalHost as string), {
        status: 200,
        headers: {
          "content-type": "text/html"
        }
      });
    }
  }
}
```

## 如何部署

```shell
# 使用wrangler
wrangler publish --config wrangler-dockerhub.toml
```

## 如何自定义域名

访问CloudFlare后台 -> Workers和Pages -> 进入需要自定义的Worker -> 设置 -> 触发器 -> 自定义域

![image-20240620140432250](https://img.witter.top/file/519f26de32e9f4b53cc3b.png)