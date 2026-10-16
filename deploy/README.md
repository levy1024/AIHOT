# embedded.news 的部署说明

本文档只讲**本站**（`embedded.news`）的部署，不是框架的通用部署指南。通用的部分看
[`docs/deploy.md`](../docs/deploy.md)。

按本文从头做一遍，就能在一台新主机上把站重新跑起来。

---

## 一、本站和别人不一样的地方

这三点决定了后面所有配置，先看懂再动手：

1. **域名没有 ICP 备案，主机在中国大陆**（当前是京东云）。未备案的域名在大陆链路上会被拦——
   读者打开会弹窗提示域名未备案。**这跟站点本身无关**：服务器、DNS 全都正常，是链路按备案拦的。
   所以本文不折腾域名的 HTTPS。

2. **对外的读者入口是 IP 加端口**：`http://117.72.35.55:3000`，明文 HTTP。
   域名和 HTTPS 现在都不对外提供服务。

3. **框架自带的 HTTPS profile 是关着的**。框架有 `docker compose --profile https` 走 Caddy
   自动签证书的现成路径，本站不用它——证书签给一个访问不到的域名没有意义。
   启动命令**不带 `--profile`**。

   > **以后备案下来了想开 HTTPS**，直接用框架自带那条路就行，不需要任何插件或密钥：
   > 按 [`docs/deploy.md`](../docs/deploy.md) 设 `SITE_DOMAIN`、`PORT=127.0.0.1:3000`、
   > `TRUST_PROXY=true`，再带 `--profile https` 启动。
   >
   > 本站**曾经**用过一套自制的 Caddy 镜像 + DNS-01 证书（因为当时 80/443 被备案拦着，
   > HTTP-01 和 TLS-ALPN-01 都用不了，只能靠腾讯云 DNSPod 的 API 密钥签发）。
   > 那套连同腾讯云 CAM 子账号后来都删掉了：域名在备案前根本访问不到，证书服务不了任何人，
   > 而备案后框架自带的路子就够了。**不要再把它加回来。**

4. **京东云安全组**要放行入站 **TCP 3000**。
   注意域名注册和 DNS 在**腾讯云**、主机在**京东云**，两家控制台别搞混——不过现在域名不提供服务，
   腾讯云那边已经用不到了。

---

## 二、前置准备

在新主机上动手之前，把这四件事准备好：

1. **主机装好 Docker 和 Docker Compose。**

2. **一个模型服务的 API Key**（当前用的是 DeepSeek）。

3. **新主机要能访问 GitHub**。大陆主机的 `github.com` 通常连不通（`api.github.com` 往往可以），
   连不通就 `git clone` 不了。本机用的办法是走 **`ssh.github.com:443`**，配置见第八节。

4. **京东云安全组的入站权限**（第三节第 5 步会用到）。

---

## 三、部署步骤

### 1. 拿代码

**先按第八节配好 SSH 通道**，否则大陆主机连不上 `github.com`，下面的克隆会一直挂着：

```bash
git clone git@github.com:levy1024/AIHOT.git myhot
cd myhot
```

`origin` 是你自己的 fork（`levy1024/AIHOT`），`upstream` 是框架原仓库（`KKKKhazix/AIHOT`）。
本站的部署配置就是第四节那一个文件。**它需要提交到你自己的 fork（`origin`）**，提交之后，
克隆 `origin` 才会带上它——从 `upstream` 克隆下来的代码里没有这个文件，端口会少一条。

### 2. 生成 `.env`

```bash
node scripts/init-env.ts --llm-key <你的模型 API Key>
```

它会生成 `.env`、填好随机密钥和管理员密码，并把管理员密码打印一次（记下来，登录 `/admin` 用）。
机器上没有 Node 就把 `.env.example` 复制成 `.env` 自己填。

### 3. 改 `.env`

除了 `init-env.ts` 填好的，重点确认这三项：

```dotenv
SITE_URL=http://117.72.35.55:3000   # 读者实际访问的地址；RSS、分享链接、sitemap 的绝对地址都用它
PORT=0.0.0.0:3000                   # web 直接对外，绑所有网卡的 3000
TRUST_PROXY=false                   # 前面没有反向代理
```

- `SITE_URL` **一定要写成读者实际访问的地址**：生成的链接、RSS、分享图、MCP 都用它。
  写错的话页面能开，但里面的链接全指向别处。
- `PORT` **必须写全 `0.0.0.0:`**。只写 `3000` 的话 docker 会把 `[::]:3000` 一起占掉，
  和第四节里那条 `[::1]:3000:3000` 撞车，web 容器直接起不来。原因见第六节。
- `TRUST_PROXY` **保持 `false`**。它是个纯布尔（`apps/web/server.ts`），没有 CIDR 白名单；
  站点直接对外时设成 `true`，任何人加一个 `X-Forwarded-For` 请求头就能伪造自己的地址，
  绕过登录次数和反馈的限流。
- `MCP_ALLOWED_HOSTS` **不用设**：MCP 的允许列表已经包含 `SITE_URL` 的地址。

### 4. 确认覆写文件在

这个文件是本站特有的，不在框架里：

```
docker-compose.override.yml
```

它**必须提交进你自己的 fork**（`origin`），否则在新主机上 `git clone` 下来的代码里没有它，
少一条端口映射。内容见第四节，缺了就从那里逐字补上。

### 5. 放行端口

京东云控制台 → 安全组 → 入站规则 → 放行 **TCP 3000**，来源 `0.0.0.0/0`。

### 6. 启动

```bash
docker compose up -d
```

**不要带 `--profile https`**（原因见第一节）。之后 `git pull` 更新代码后，命令不变。

---

## 四、`docker-compose.override.yml` 的内容

docker compose 会**自动加载**这个文件并与 `docker-compose.yml` 合并，所以启动命令不用变。
它刻意没有写进 `docker-compose.yml`——那是框架的通用文件，改了就永远跟上游同步不上。

```yaml
# 本站的部署覆写。docker compose 会自动加载这个文件，和 docker-compose.yml 合并。
#
# 为什么单独放一个文件：这里的改动只对本站成立（域名没备案，读者只能从 IP 加端口进来），
# 不属于框架的通用配置。放在这里，docker-compose.yml 保持原样，以后跟上游更新不会冲突。
#
# 用法：docker compose up -d
#
# 这里只「追加」一条 IPv6 回环的端口映射。compose 合并时 ports 是拼接而不是替换，
# 框架自带的那条 ${PORT:-3000}:3000 会原样保留，所以不需要 !override 整段替换
# ——那样反而会把框架那条顶掉，将来上游改了端口写法我们也跟不到。
services:
  # web 直接对公网发布。框架自带的那条只绑 IPv4 的所有网卡，这里补上 IPv6 回环，
  # 免得哪天 localhost 被解析成 ::1 时连不上（本机当前 /etc/hosts 里 localhost 只指 127.0.0.1）。
  # 注意端口号写死了 3000：改了 .env 里 PORT 的端口号，这里要跟着改。
  web:
    ports:
      - "[::1]:3000:3000"
```

改完用 `docker compose config` 检查合并结果，别靠猜。合并后 `web` 应该有两条端口：
`0.0.0.0:3000`（来自框架）和 `[::1]:3000`（来自这里）。

---

## 五、验证

启动后按顺序跑一遍：

```bash
# 1. 站点自检（逐项打印 ✓，最后一行 "all checks passed"）
docker compose exec -T web node scripts/smoke.ts --base http://localhost:3000

# 2. 三个本机入口都应该是 200
curl -s -o /dev/null -w "localhost %{http_code}\n" http://localhost:3000/
curl -s -o /dev/null -w "[::1]     %{http_code}\n" "http://[::1]:3000/"
curl -s -o /dev/null -w "eth0      %{http_code}\n" http://172.16.0.3:3000/   # 换成这台机器的私网地址

# 3. 生成的链接指向 SITE_URL，而不是 localhost
curl -s http://localhost:3000/robots.txt | grep -i sitemap
```

**外部可达性**：

```bash
curl -I http://<公网IP>:3000/
```

打不开就是安全组那条规则没生效——回到京东云控制台检查（协议 TCP、端口 3000、来源 `0.0.0.0/0`）。

> 从主机自己 curl 自己的公网 IP 走的是 NAT 回环，**而且同样要过安全组**。
> 所以这条命令的成败跟外部一致，可以拿它先快速判断规则有没有生效；不放心的话再用一台
> 真正在外部网络的机器确认一次。

> `smoke.ts` **必须在容器里跑**。宿主机上没有 workspace 依赖，直接 `node scripts/smoke.ts`
> 会报 `Cannot find package '@aihot/site'`。

---

## 六、排错

**容器起不来，报 `failed to bind host port [::1]:3000/tcp: address already in use`**

`.env` 里的 `PORT` 写成了裸的 `3000`。裸 `3000` 会让 docker 把 `[::]:3000` 一起占掉，
和覆写文件里那条 `[::1]:3000:3000` 撞车。改成 `0.0.0.0:3000`——只占 IPv4，
本机没有全局 IPv6，所以不占 `[::]` 也不会漏掉任何访客。

**域名打不开**

**那是备案拦的，不是服务器故障，别去查服务。** 判断方法：从主机自己访问
`http://localhost:3000/` 和 `http://<公网IP>:3000/` 都正常、`getent hosts embedded.news`
解析也对，就说明服务器这一侧没有任何问题，没有可修的东西——域名在大陆链路上被拦，
换端口、换证书、改配置都不会有用。入口就是 IP 加端口。

**外部连不上，本机却正常**

安全组没有放行 TCP 3000，或者放行的是别的端口。回到京东云控制台核对。

**改了端口**

要同步四个地方，否则会出现「容器起不来」或「页面能开但链接指向旧地址」：
`.env` 的 `SITE_URL` 和 `PORT`、京东云安全组、以及 `docker-compose.override.yml` 里
`web` 那段写死的 `[::1]:3000:3000`。

---

## 七、日常维护

- **更新代码**：

  ```bash
  git pull origin main       # 自己的 fork
  git pull upstream main     # 上游；需要先按第八节处理，否则拉不动
  docker compose up -d
  ```

  `docker-compose.override.yml` 是本站特有的，与上游文件不冲突，`git pull` 不会打架。
- **`.env` 不进 Git**（已在 `.gitignore` 里），所以**新主机上必须重新填一遍**。
  备份 `.env` 请用安全的渠道，别丢进版本库。
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
