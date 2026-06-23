# Dockerfiles

个人维护的 Docker 镜像构建文件集合。

## 镜像列表

### NGINX

基于 Alpine Linux 从源码构建的 NGINX 镜像，支持 HTTP/2、HTTP/3、QUIC、Brotli 和 Stream 代理。

[查看 NGINX 镜像详细说明](./nginx/README.md)

## 使用说明

进入对应的子目录，按照目录内的 `README.md` 完成镜像构建、运行和配置。

敏感信息请通过环境变量或 Docker Secret 管理，不要将密钥、密码和证书私钥提交到仓库。
