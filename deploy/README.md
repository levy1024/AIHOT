# embedded.news 的部署说明

本文档只讲**本站**（`embedded.news`）的部署，不是框架的通用部署指南。通用的部分看
[`docs/deploy.md`](../docs/deploy.md)。

按本文从头做一遍，就能在一台新主机上把站重新跑起来。

---

## 一、本站和别人不一样的地方

这三点决定了后面所有配置，先看懂再动手：

1. **域名没有 ICP 备案，主机在中国大陆**（当前是京东云）。所以 **80/443 不能用**——会被运营商拦。
   结果是站点对外走**非标端口 4321**，读者访问的地址必须带端口：`https://embedded.news:4321`。
   这个 `:4321` 去不掉，是非标端口的固有代价。

2. **因为 80/443 不可用，ACME 的 HTTP-01 和 TLS-ALPN-01 挑战都用不了**，证书只能走 **DNS-01**：
   靠 DNS 服务商的 API 密钥证明域名所有权，**完全不依赖任何入站端口**。好处是备案永远不会影响续期。

3. **域名和 DNS 在腾讯云（DNSPod），主机在京东云——两家控制台，别搞混**：

   | 去哪 | 干什么 |
   |---|---|
   | 腾讯云 → 访问管理 CAM | 建子用户拿 `SecretId`/`SecretKey`（授权 DNSPod 记录读写），DNS-01 用 |
   | 京东云 → 安全组 | 放行入站 **TCP 4321**（可选再放行 UDP 4321，见第六节） |

---

## 二、前置准备

在新主机上动手之前，把这五件事准备好：

1. **域名解析**：`embedded.news` 的 A 记录指向新主机的公网 IP。用 `getent hosts embedded.news`
   确认解析结果就是那台机器的公网 IP。

2. **腾讯云 CAM 子用户**（这一步在腾讯云，和主机在哪家云无关）：
   - 控制台 → 访问管理 CAM → 用户 → 新建子用户
   - **只勾「编程访问」**，不要勾控制台登录（它只需要调 API）
   - 策略里搜 **DNSPod**，选 **`QcloudDNSPodFullAccess`**
   - 拿到 `SecretId` 和 `SecretKey`

   > **必须是读写权限。** `QcloudDNSPodReadOnlyAccess` 不行：DNS-01 要**新建**一条
   > `_acme-challenge` TXT 记录来证明域名所有权，验证完还要**删除**它。只读会在第一步就失败，
   > 而且报错通常只说「权限不足」，不会点明是策略选窄了，排查很绕。
   >
   > **不要用主账号密钥**：主账号的 `SecretId`/`SecretKey` 是整个腾讯云账号的权限，泄露等于账号被接管。

3. **主机装好 Docker 和 Docker Compose**。Compose 要 **≥ 2.24.4**——本站的覆写文件用到了
   `!override` YAML 标签（原因见第四节），更早的版本不认。

4. **一个模型服务的 API Key**（当前用的是 DeepSeek）。

5. **新主机要能访问 GitHub**。大陆主机的 `github.com` 通常连不通（`api.github.com` 往往可以），
   连不通就 `git clone` 不了。本机用的办法是走 **`ssh.github.com:443`**，配置见第八节。

---

## 三、部署步骤

### 1. 拿代码

**先按第八节配好 SSH 通道**，否则大陆主机连不上 `github.com`，下面的克隆会一直挂着：

```bash
git clone git@github.com:levy1024/AIHOT.git myhot
cd myhot
```

`origin` 是你自己的 fork（`levy1024/AIHOT`），`upstream` 是框架原仓库（`KKKKhazix/AIHOT`）。
本站的部署配置就是[第四节](#四三个文件的内容)那三个文件。**它们需要提交到你自己的 fork（`origin`）**，
提交之后，克隆 `origin` 才会带上它们——从 `upstream` 克隆下来的代码里没有这三个文件，起不来。

### 2. 生成 `.env`

```bash
node scripts/init-env.ts --llm-key <你的模型 API Key>
```

它会生成 `.env`、填好随机密钥和管理员密码，并把管理员密码打印一次（记下来，登录 `/admin` 用）。
机器上没有 Node 就把 `.env.example` 复制成 `.env` 自己填。

### 3. 改 `.env`

除了 `init-env.ts` 填好的，重点确认这几项：

```dotenv
SITE_URL=https://embedded.news:4321      # 读者实际访问的地址；RSS、分享链接、sitemap 的绝对地址都用它
SITE_DOMAIN=embedded.news                # Caddy 签证书用的域名
SITE_PORT=4321                           # Caddy 对外监听的端口
PORT=127.0.0.1:3000                      # web 只绑回环，仅本机可访问
TRUST_PROXY=true                         # 前面有 Caddy，访客地址从 X-Forwarded-For 读

TENCENTCLOUD_SECRET_ID=<腾讯云 CAM 子用户的 SecretId>
TENCENTCLOUD_SECRET_KEY=<腾讯云 CAM 子用户的 SecretKey>
```

`MCP_ALLOWED_HOSTS` **不用设**：MCP 的允许列表已经包含 `SITE_URL` 的域名。只有当站点还需要从
**另一个域名**访问时才要列在这里。

> `TRUST_PROXY=true` 和 `PORT=127.0.0.1:3000` 是**配套的，不能只改一个**。
> `TRUST_PROXY=true` 意味着访客地址从 `X-Forwarded-For` 请求头里读，这个头是可以伪造的；
> 把 3000 绑死在回环上、只让本机的 Caddy 能连，伪造的路才被堵住。
> 如果哪天把 3000 开回对外，就得同时把 `TRUST_PROXY` 改回 `false`，否则有人可以直接伪造地址
> 绕过登录次数和反馈的限流。

### 4. 确认三个文件在

这三个文件是本站特有的，不在框架里：

```
docker-compose.override.yml
deploy/caddy/Dockerfile
deploy/caddy/Caddyfile
```

这三个文件**必须提交进你自己的 fork**（`origin`），否则在新主机上 `git clone` 下来的代码里没有它们，
起不来。内容见[第四节](#四三个文件的内容)，缺了就从那里逐字补上。

### 5. 放行端口

京东云控制台 → 安全组 → 入站规则 → 放行 **TCP 4321**，来源 `0.0.0.0/0`。
80/443 **不用开**。

### 6. 启动

```bash
docker compose --profile https up -d --build
```

第一次会构建 Caddy 镜像（要编译，几分钟）。之后 `git pull` 更新代码后，命令不变。

---

## 四、三个文件的内容

这三个文件是本站部署的核心，逐字抄下来即可。它们**刻意没有写进** `docker-compose.yml` 和
`deploy/Caddyfile`——那两个是框架的通用文件，改了就永远跟上游同步不上；这些改动只对本站成立。

### 1. `docker-compose.override.yml`

docker compose 会**自动加载**这个文件并与 `docker-compose.yml` 合并，所以启动命令不用变。

```yaml
# 本站的部署覆写。docker compose 会自动加载这个文件，和 docker-compose.yml 合并。
#
# 为什么单独放一个文件：这些改动只对本站成立（域名未备案、用腾讯云 DNS、走非标端口），
# 不属于框架的通用配置。放在这里，docker-compose.yml 和 deploy/Caddyfile 保持原样，
# 以后跟上游更新不会冲突。
#
# 用法不变：docker compose --profile https up -d --build
#
# 注意 ports 和 volumes 用了 !override 标签：compose 默认会把这两个列表「拼接」而不是替换，
# 不写 !override 就会同时挂上框架自带的 deploy/Caddyfile（挂到同一个容器路径）并继续发布 80/443。
services:
  caddy:
    build:
      context: .
      dockerfile: deploy/caddy/Dockerfile
    image: aihot-caddy
    ports: !override
      - "${SITE_PORT:-4321}:${SITE_PORT:-4321}"
      # Caddy 会在响应头广播 alt-svc: h3（HTTP/3 走 UDP）。不发布 UDP 的话浏览器会先试
      # QUIC、失败再回退到 TCP，每个新访客多花几十毫秒。发布出来即可；安全组不放行 UDP
      # 也不影响访问，只是回到上面的回退行为。
      - "${SITE_PORT:-4321}:${SITE_PORT:-4321}/udp"
    environment:
      SITE_DOMAIN: ${SITE_DOMAIN:-localhost}
      SITE_PORT: ${SITE_PORT:-4321}
      TENCENTCLOUD_SECRET_ID: ${TENCENTCLOUD_SECRET_ID:-}
      TENCENTCLOUD_SECRET_KEY: ${TENCENTCLOUD_SECRET_KEY:-}
    volumes: !override
      - ./deploy/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy:/data

  # web 只给本机用：Caddy 走 compose 内网访问 web:3000，主机侧只绑回环。
  # 框架默认的 PORT=127.0.0.1:3000 只绑 IPv4 回环，这里补上 IPv6 回环，
  # 免得哪天 localhost 被解析成 ::1 时连不上（本机当前 /etc/hosts 里 localhost 只指 127.0.0.1）。
  # 注意第二行写死了 3000：改了 PORT 的端口号，这里要跟着改。
  web:
    ports: !override
      - "${PORT:-3000}:3000"
      - "[::1]:3000:3000"
```

> **`!override` 这个坑必须记住。** compose 合并两个文件时，`ports` 和 `volumes` 这类列表是
> **拼接**而不是替换。不写 `!override` 会得到两个后果：caddy 同时挂上框架自带的
> `deploy/Caddyfile` 和本站的 Caddyfile（挂到同一个容器路径 `/etc/caddy/Caddyfile`，直接冲突），
> 并且继续发布 80/443。
>
> 改完用 `docker compose --profile https config` 检查合并结果，别靠猜。

### 2. `deploy/caddy/Dockerfile`

```dockerfile
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
```

### 3. `deploy/caddy/Caddyfile`

```caddyfile
# 本站对外用非标端口（默认 4321）：域名未备案，大陆机房对 80/443 的访问会被拦。
# 因此证书走 DNS-01，靠腾讯云 DNSPod 的 API 密钥证明域名所有权，不依赖任何入站端口，
# 所以 80/443 完全不用对外开放，也不会被备案影响续期。
#
# 与框架自带 deploy/Caddyfile 的区别：多了端口和 tls 块，其余（压缩、安全响应头）保持一致。
{$SITE_DOMAIN}:{$SITE_PORT} {
	encode zstd gzip
	header ?Strict-Transport-Security "max-age=31536000"
	header ?X-Frame-Options "SAMEORIGIN"
	header ?X-Content-Type-Options "nosniff"
	header ?Referrer-Policy "strict-origin-when-cross-origin"
	header ?Permissions-Policy "camera=(), microphone=(), geolocation=()"

	tls {
		dns tencentcloud {
			secret_id {env.TENCENTCLOUD_SECRET_ID}
			secret_key {env.TENCENTCLOUD_SECRET_KEY}
		}
	}

	reverse_proxy web:3000
}
```

---

## 五、验证

启动后按顺序跑一遍。**注意区分「从哪台机器发起」**——从主机自己发起的测试走的是 NAT 回环，
**可能绕过云安全组**，所以它只能证明 Caddy 和证书没问题，**不能证明外网能访问**。

```bash
# 1. 站点能访问、证书可信（ssl_verify_result=0 表示证书可信）
curl -s -o /dev/null -w "%{http_code} %{ssl_verify_result} %{http_version}\n" \
  https://embedded.news:4321/

# 2. 看证书内容和有效期
echo | openssl s_client -connect embedded.news:4321 -servername embedded.news 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# 3. 证书签发过程。首次部署才会有下面的输出：
#      trying to solve challenge  challenge_type: dns-01
#      certificate obtained successfully
#    证书签下来后就存在 caddy 数据卷里，容器重启不会重签，所以之后这条命令没有输出是正常的。
docker compose --profile https logs caddy | grep -iE "challenge|certificate obtained|error"

# 4. 临时验证记录已被清理（无输出 = 清理干净，也说明密钥的删除权限是通的）
dig +short TXT _acme-challenge.embedded.news @223.5.5.5

# 5. 站点自检（逐项打印 ✓，最后一行 "all checks passed"）
docker compose exec -T web node scripts/smoke.ts --base https://embedded.news:4321

# 6. 本机入口仍然可用，外部进不来
curl -s -o /dev/null -w "localhost %{http_code}\n" http://localhost:3000/
curl -s -o /dev/null -w "[::1]     %{http_code}\n" "http://[::1]:3000/"
curl -s -o /dev/null -w "公网 IP   %{http_code}（000=连不上，符合预期）\n" http://<公网IP>:3000/
```

**最后一步必须用外部网络验**（手机关掉 WiFi 用流量，或任何不在本机的机器）：

```bash
curl -I https://embedded.news:4321/
```

打不开就是安全组没生效——回到京东云控制台检查入站规则（协议 TCP、端口 4321、来源 `0.0.0.0/0`）。
**这一步在主机内部怎么测都测不出来。**

> `smoke.ts` **必须在容器里跑**。宿主机上没有 workspace 依赖，直接 `node scripts/smoke.ts`
> 会报 `Cannot find package '@aihot/site'`。

---

## 六、排错

**构建 Caddy 镜像失败，报 `libdns.Record has no field or method`**

`caddy-dns/dnspod` 这个模块（吃 DNSPod Token 的那个）**已经废弃**：它最新版 v0.0.4 锁的
`libdns/dnspod@v0.0.3` 还在用旧版 `libdns.Record` 结构体，而新版 libdns 已经把它改成接口了，
所以怎么构建都编译不过。**这是本站用 `caddy-dns/tencentcloud` 而不是 `dnspod` 模块的原因**，
也是为什么密钥必须从腾讯云 CAM 出、而不是 DNSPod 的 Token。
（DNSPod 控制台「密钥管理」里的 Token 在本方案里**完全用不到**。）

**构建时报模块下载失败**

`deploy/caddy/Dockerfile` 里的 `GOPROXY=https://goproxy.cn,direct` 是给境内用的。如果在境外主机上
构建，可以直接删掉那一行让 Go 走默认源。

**签证书失败，日志里是权限错误**

按顺序查：CAM 策略是不是 `QcloudDNSPodFullAccess`（只读不行）；`SecretId`/`SecretKey` 有没有填错；
**域名是不是在这个腾讯云账号下**——`dnspod.tencentcloudapi.com` 只能操作与该账号关联的 DNSPod 域名，
如果域名在一个没关联的独立 DNSPod 老账号里，会报找不到域名或权限不足。

**签证书慢**

国内 DNS 传播偶尔偏慢。可以在 Caddyfile 的 `tls` 块里加 `propagation_delay 60s`，
或用 `resolvers 223.5.5.5` 指定校验用的解析器。证书签下来之后就存进 `caddy` 数据卷了，
容器重启不会重签。

**浏览器访问慢一下才打开**

Caddy 会在响应头广播 `alt-svc: h3=":4321"`（HTTP/3 走 UDP）。如果安全组只放行了 TCP，
浏览器会先尝试 QUIC、失败后再回退到 TCP，每个新访客多花几十毫秒。要消除它，
在京东云安全组再放行一条 **UDP 4321** 即可。

**改了端口**

改 `SITE_PORT` 要同步三个地方，否则会出现「Caddy 起来了但访问不了」或「证书域名对不上」：
`.env` 的 `SITE_URL`、京东云安全组、以及 `docker-compose.override.yml` 里 `web` 那段写死的
`[::1]:3000:3000`（如果同时改了 `PORT` 的端口号）。

---

## 七、日常维护

- **证书续期是自动的**：Caddy 在到期前约 30 天自动续，不用配 cron。前提是 CAM 密钥一直有效——
  如果吊销了那个子用户，续期会失败，站点在证书过期后打不开。
- **更新代码**：

  ```bash
  git pull origin main       # 自己的 fork
  git pull upstream main     # 上游；需要先按第八节处理，否则拉不动
  docker compose --profile https up -d --build
  ```

  `docker-compose.override.yml` 和 `deploy/caddy/` 是本站特有的，与上游文件不冲突，`git pull` 不会打架。
- **`.env` 不进 Git**（已在 `.gitignore` 里），所以**新主机上必须重新填一遍**，包括
  `TENCENTCLOUD_*`。备份 `.env` 请用安全的渠道，别丢进版本库。
- **`worker` 停止有 210 秒宽限期**，这是为了防止正在跑的付费模型调用被中途杀掉（会留下已付费但
  状态未知的回执）。所以重启时别用 `docker compose kill`。

---

## 八、让新主机能连 GitHub（逃开对 `github.com` 的封锁）

大陆主机上 `github.com:443` 通常被墙，但 **`ssh.github.com:443` 一般可达**，把 GitHub 的 SSH 连接
改走 443 端口即可。实测：本机 `github.com` 超时、`api.github.com` 与 `ssh.github.com:443` 均通。

### 1. 生成密钥

```bash
ssh-keygen -t ed25519 -C "$(hostname)" -f ~/.ssh/id_ed25519_gh -N ""
```

`-N ""` 是不设口令，便于无人值守推送。代价是**这台机器被攻破就等于这把钥匙泄露**，
所以下面用**部署密钥**而不是账号级 SSH 密钥把它限制在单个仓库内。

### 2. 把公钥加成部署密钥（只对 `levy1024/AIHOT` 有效）

有 `gh` 且已登录（`gh auth login`）时：

```bash
gh api --method POST repos/levy1024/AIHOT/keys \
  -f title="$(hostname)" -f key="$(cat ~/.ssh/id_ed25519_gh.pub)" -F read_only=false
```

`read_only=false` 是**可写**（能 push）；只拉不推的话用 `true` 更安全。
没装 `gh` 就去 GitHub 网页端：仓库 → Settings → Deploy keys → Add deploy key，勾上
"Allow write access"。

### 3. 配置 SSH 走 443

写进 `~/.ssh/config`：

```
Host github.com
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519_gh
    IdentitiesOnly yes
```

然后加入主机密钥并**核对指纹**，别盲信第一次连接：

```bash
ssh-keyscan -p 443 -t ed25519 ssh.github.com | ssh-keygen -lf -
# 应输出：SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU（GitHub 官方公布的 Ed25519 指纹）
ssh-keyscan -p 443 -t ed25519 ssh.github.com | sed 's/^ssh.github.com/[ssh.github.com]:443/' >> ~/.ssh/known_hosts
```

### 4. 验证并切换 remote

```bash
ssh -T git@github.com
# 回显 "Hi levy1024/AIHOT! You've successfully authenticated" 即成功。
#   注意回显里是 用户名/仓库名，说明用的是仓库级部署密钥；账号级密钥只会显示用户名。

git remote set-url origin git@github.com:levy1024/AIHOT.git
```

### 已知限制：`upstream` 拉不动

部署密钥**只对该仓库有效**，所以 `upstream`（`KKKKhazix/AIHOT`）走 SSH 会被拒；而它默认的 HTTPS
地址又指向被墙的 `github.com`，于是 `git fetch upstream` 会卡住。要用上游更新，二选一：

- 把同一把公钥也加到**账号级** SSH 密钥（GitHub → Settings → SSH and GPG keys），并在 `~/.ssh/config`
  的 `Host github.com` 段里保留 `IdentityFile`——账号级密钥能访问所有公开仓库，但权限范围是整个账号；
- 或者给这台机器配一个出口代理，让 `git` 走代理（`.env` 里的 `EGRESS_PROXY_URL` 只作用于站点的
  抓取，不影响 `git`；要给 git 单独设 `git config --global http.proxy`）。
