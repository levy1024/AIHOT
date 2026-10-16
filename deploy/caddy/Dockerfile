# 本站的 Caddy 镜像。
#
# 官方 caddy 镜像只含内置模块，第三方 DNS 插件必须在构建时静态编进去（Caddy 没有运行时插件加载）。
# 域名未办备案、对外走非标端口，ACME 的 HTTP-01 和 TLS-ALPN-01 都用不了，只能走 DNS-01，
# 所以需要 caddy-dns/tencentcloud 这个插件。
#
# 构建时用国内 Go 模块代理，否则在境内拉不到模块。
FROM caddy:2-builder AS builder
ARG GOPROXY=https://goproxy.cn,direct
ENV GOPROXY=${GOPROXY}
# 锁定版本：插件上游变动会直接导致构建失败，写死版本才能复现。
RUN xcaddy build --with github.com/caddy-dns/tencentcloud@v0.4.3

FROM caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
