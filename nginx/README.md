# NGINX HTTP/3 Docker 镜像

这是一个基于 Alpine Linux、从源码编译的 NGINX 镜像，支持 HTTP/2、HTTP/3、QUIC、Brotli 压缩及自定义响应头。

## 功能特性

- NGINX `1.30.3`
- Alpine Linux `3.22`
- HTTP/2
- HTTP/3 与 QUIC
- TLS/SSL
- Brotli 动态压缩和静态文件支持
- headers-more-nginx-module
- Stream TCP/UDP 代理
- Stream SSL 与 SNI 预读取
- 多阶段构建
- 第三方源码版本固定及 SHA256 完整性校验
- 访问日志和错误日志输出到容器标准输出

## 编译模块

主要编译参数如下：

```text
--with-http_gzip_static_module
--with-http_realip_module
--with-http_ssl_module
--with-http_stub_status_module
--with-http_sub_module
--with-http_v2_module
--with-http_v3_module
--with-stream
--with-stream_ssl_module
--with-stream_ssl_preread_module
--add-module=ngx_brotli
--add-module=headers-more-nginx-module
```

## 构建镜像

在当前目录执行：

```bash
docker build -t nginx-http3:1.30.3 .
```

也可以在项目根目录执行：

```bash
docker build \
  -f http3/Dockerfile \
  -t nginx-http3:1.30.3 \
  http3
```

查看 NGINX 版本和编译参数：

```bash
docker run --rm nginx-http3:1.30.3 nginx -V
```

检查默认配置：

```bash
docker run --rm nginx-http3:1.30.3 nginx -t
```

## 运行容器

HTTP/3 使用 QUIC，因此 `443` 端口必须同时映射 TCP 和 UDP。

```bash
docker run -d \
  --name nginx-http3 \
  --restart unless-stopped \
  -p 80:80 \
  -p 443:443/tcp \
  -p 443:443/udp \
  nginx-http3:1.30.3
```

查看运行日志：

```bash
docker logs -f nginx-http3
```

停止容器时，镜像使用 `SIGQUIT` 让 NGINX 优雅退出。

## HTTP/3 配置示例

HTTP/3 必须使用 HTTPS，并需要有效的 TLS 证书。可以将下面内容保存为 `nginx.conf`：

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    access_log logs/access.log;
    error_log  logs/error.log warn;

    sendfile on;

    server {
        listen 80;
        server_name example.com;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen 443 quic reuseport;
        http2 on;

        server_name example.com;

        ssl_certificate     /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;
        ssl_protocols       TLSv1.3;

        add_header Alt-Svc 'h3=":443"; ma=86400' always;
        add_header X-Protocol $server_protocol always;

        brotli on;
        brotli_static on;
        brotli_comp_level 6;
        brotli_types
            text/plain
            text/css
            application/javascript
            application/json
            application/xml
            image/svg+xml;

        location / {
            root  /etc/nginx/html;
            index index.html;
        }
    }
}
```

挂载配置、证书和静态文件：

```bash
docker run -d \
  --name nginx-http3 \
  --restart unless-stopped \
  -p 80:80 \
  -p 443:443/tcp \
  -p 443:443/udp \
  -v "$PWD/nginx.conf:/etc/nginx/conf/nginx.conf:ro" \
  -v "$PWD/certs:/etc/nginx/certs:ro" \
  -v "$PWD/html:/etc/nginx/html:ro" \
  nginx-http3:1.30.3
```

> 使用 Docker Desktop、云服务器安全组或主机防火墙时，也需要放行 `443/udp`。

## 验证 HTTP/3

使用支持 HTTP/3 的 curl：

```bash
curl --http3 -I https://example.com
```

也可以在 Chrome 或 Edge 开发者工具的 Network 面板中查看 Protocol 列。成功启用后，协议通常显示为 `h3`。

如果客户端首次访问仍然使用 HTTP/2，通常是因为客户端需要先通过 HTTPS 响应中的 `Alt-Svc` 头获知 HTTP/3 入口。刷新后再检查即可。

## 自定义构建版本

Dockerfile 支持以下构建参数：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `ALPINE_VERSION` | `3.22` | Alpine Linux 版本 |
| `NGINX_VERSION` | `1.30.3` | NGINX 版本 |
| `NGINX_SHA256` | Dockerfile 内置值 | NGINX 源码包校验和 |
| `HEADERS_MORE_VERSION` | `0.40` | headers-more 模块版本 |
| `HEADERS_MORE_SHA256` | Dockerfile 内置值 | headers-more 源码包校验和 |
| `NGX_BROTLI_COMMIT` | Dockerfile 内置值 | ngx_brotli Git commit |

例如：

```bash
docker build \
  --build-arg NGINX_VERSION="<新版本>" \
  --build-arg NGINX_SHA256="<对应的 SHA256>" \
  -t nginx-http3:<新版本> \
  .
```

修改源码版本时，必须同时更新对应的 SHA256 校验和，否则构建会按预期失败。

## 常见问题

### 浏览器没有使用 HTTP/3

依次确认：

1. `443/tcp` 和 `443/udp` 均已映射并放行。
2. 配置中存在 `listen 443 quic reuseport;`。
3. HTTPS 响应包含正确的 `Alt-Svc` 响应头。
4. 域名证书有效，且客户端支持 HTTP/3。
5. CDN、负载均衡器或反向代理没有拦截 UDP。

### 容器启动失败

先检查配置：

```bash
docker run --rm \
  -v "$PWD/nginx.conf:/etc/nginx/conf/nginx.conf:ro" \
  nginx-http3:1.30.3 \
  nginx -t
```

然后查看日志：

```bash
docker logs nginx-http3
```

### 如何确认模块已编译

```bash
docker run --rm nginx-http3:1.30.3 nginx -V 2>&1
```

输出中应包含 `--with-http_v3_module`、`ngx_brotli` 和 `headers-more` 等编译参数。

## 安全建议

- 使用受信任的正式 TLS 证书。
- 定期更新 Alpine、NGINX 和第三方模块。
- 更新版本时同步更新 SHA256 或固定 commit。
- 只挂载容器运行所需的目录，并尽量使用只读挂载。
- 在 CI 中执行镜像构建、`nginx -t` 和漏洞扫描。
- 生产环境应根据实际需求限制 TLS 协议、密码套件和请求大小。
