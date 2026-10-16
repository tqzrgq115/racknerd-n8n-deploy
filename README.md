# RackNerd部署n8n：一键安装与Docker双方案全流程，年付21.99美元起，附套餐选择和HTTPS配置

在 RackNerd 上部署 n8n 有两条路：用官方 ctrl 面板一键安装，或者自己装 Docker 手动部署。前者省事，15–45 分钟出结果，但要求 2 vCPU 和 2 GB 内存起步，而且安装过程会清空整台 VPS 的已有数据；后者门槛低，1 GB 内存的年付 21.99 美元套餐也能跑起来。这篇文章把两条路都走一遍，顺带讲清楚套餐怎么选、域名和 HTTPS 怎么配、部署之后怎么更新维护。

**选购速览**

> - 想省事：选 2 GB 套餐（年付 $35.99），在 ctrl 面板一键装 n8n，👉 [开通 2GB 套餐并一键安装 n8n](https://my.racknerd.com/aff.php?aff=16470&pid=953)
> - 想省钱：选 1 GB 套餐（年付 $21.99），用 Docker 手动部署，日常几十个工作流够用
> - 特惠套餐按原价终身续费，第二年不涨价
> - 计划接 Telegram、邮件这类 webhook 自动化的话，域名和 HTTPS 别省，下文有完整配法

## RackNerd 跑 n8n 够用吗？先把套餐和价格看清楚

够用。n8n 本身很轻，Docker 镜像只有 200 MB 左右，有第三方指南认为 1 GB 内存的方案足以跑几十个活跃工作流。但要注意一个容易踩的坑：RackNerd 面板里的 n8n 一键安装要求 2 vCPU 和 2 GB 内存起步，1 GB 套餐只能走 Docker 手动部署这条路。

RackNerd 特惠页当前在售的 KVM VPS 套餐如下，价格全部为美元年付：

| 套餐 | CPU / 内存 / 存储 | 月流量 | 年付价格 | 购买 |
| --- | --- | --- | --- | --- |
| 1 GB KVM VPS | 1 vCPU / 1 GB / 20 GB SSD | 3 TB | $21.99/年 | [ 立即开通](https://my.racknerd.com/aff.php?aff=16470&pid=952) |
| 2 GB KVM VPS | 2 vCPU / 2 GB / 35 GB SSD | 5 TB | $35.99/年 | [ 立即开通](https://my.racknerd.com/aff.php?aff=16470&pid=953) |
| 4 GB KVM VPS | 3 vCPU / 4 GB / 60 GB SSD | 7 TB | $59.99/年 | [ 立即开通](https://my.racknerd.com/aff.php?aff=16470&pid=954) |
| 6 GB KVM VPS | 6 vCPU / 6 GB / 100 GB SSD | 12 TB | $89.99/年 | [ 立即开通](https://my.racknerd.com/aff.php?aff=16470&pid=955) |
| 8 GB KVM VPS | 7 vCPU / 8 GB / 150 GB SSD | 20 TB | $119.99/年 | [ 立即开通](https://my.racknerd.com/aff.php?aff=16470&pid=956) |

所有套餐共享的配置：1 Gbps 带宽、RAID-10 SSD、1 个独立 IPv4、完整 root 权限、KVM/ctrl 面板、多个机房可选，系统支持 Ubuntu、Debian、AlmaLinux，也可以开工单装自定义 ISO。官方明确说明这些特惠价是终身循环价格，续费不涨价。

把账算一下：1 GB 套餐折合每月约 1.83 美元，2 GB 约 3 美元，跑 n8n 这个量级的成本已经很低。想省钱的话，👉 [查看 1GB 套餐的年付价格](https://my.racknerd.com/aff.php?aff=16470&pid=952)是整个系列的起点。

## 一键安装还是 Docker 手动部署？两条路的真实差别

先看两个问题：内存够不够 2 GB，介不介意重装系统。这两个答案基本就能替你定方式。

| 对比项 | ctrl 一键安装 | Docker 手动部署 |
| --- | --- | --- |
| 最低配置 | 2 vCPU、2 GB 内存 | 1 GB 内存即可 |
| 耗时 | 约 15–45 分钟 | 熟练后十分钟内 |
| 对已有数据 | 重装系统，清空 VPS 上全部数据 | 不影响已有内容 |
| 灵活性 | 标准镜像，改动余地小 | 端口、挂载、反代全由你定 |
| 适合谁 | 只要 n8n、不想碰命令行 | 想省预算或打算同机跑多个服务 |

一键安装有个必须知道的前提，RackNerd 官方在面板里写得很直白：

> 一键安装会永久擦除 VPS 上已有的系统、应用和文件，然后部署一个带所选应用的全新镜像。

所以一台新开的 VPS 随便点，但如果机器上已经有别的东西，先备份再动手。

## 方法一：ctrl 面板一键安装 n8n

流程只有四步。在特惠页下单一台 2 GB 及以上的 VPS，等开通邮件；登录 ctrl 面板，进入 Virtual Servers 下的 One-Click Apps；选择 n8n 并确认安装；之后等 15–45 分钟，SSH root 密码和 n8n 应用登录信息会直接显示在面板里。

整个过程不需要 SSH，也不需要自己配 Docker。缺什么补什么就行：内存不够就换 2 GB 方案，👉 [开一台 2GB 内存的 VPS 来一键装 n8n](https://my.racknerd.com/aff.php?aff=16470&pid=953)。唯一要记住的还是那条警告——点下去等于重装系统。

## 方法二：Docker 手动部署，1 GB 套餐也能跑

1 GB 套餐省下的钱要用几条命令换回来。流程如下：

**1. 连接服务器。** Windows 用 PowerShell，Mac 和 Linux 用终端，执行：

bash
ssh root@你的服务器IP


**2. 安装 Docker。** 以 Ubuntu 为例，先添加官方源：

bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc


bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker


**3. 一条命令启动 n8n：**

bash
docker run -d --restart unless-stopped --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n n8nio/n8n


这条命令会自动拉取镜像、后台运行，并在服务器重启后自动拉起 n8n。数据持久化在 `~/.n8n` 目录里。

**4. 放行端口并访问。** 测试阶段执行 `ufw allow 5678`，然后浏览器打开 `http://服务器IP:5678`，设置管理员账号就能用了。等下面配好反向代理，建议把 5678 关掉，只保留 80 和 443。

## 绑定域名和 HTTPS：让 n8n 真正能接 webhook

直接用 IP 加端口访问，只适合自己挂着浏览器玩。n8n 的大部分价值在自动化——而 webhook 触发的流程要求一个公网可访问的 HTTPS 地址，这步省不掉。

**第一步，把域名指到 VPS。** 推荐用 Cloudflare 管 DNS，在域名解析里加一条 A 记录：

| 字段 | 值 |
| --- | --- |
| 类型 | A |
| 主机名 | `n8n`（或 `@` 指向根域名） |
| 记录值 | 你的 RackNerd VPS IP |
| TTL | 3600（测试期可用 600 加快生效） |

**第二步，建工作目录并写配置。** 每个服务单独放一个目录，方便以后备份：

bash
sudo mkdir -p /opt/docker-data/n8n
cd /opt/docker-data/n8n


创建 `.env` 文件，把域名换成你自己的：


DOMAIN=n8n.yourdomain.com
N8N_HOST=n8n.yourdomain.com
WEBHOOK_URL=https://n8n.yourdomain.com/
N8N_ENCRYPTION_KEY=换成一段足够长的随机字符串


保存后执行 `chmod 600 .env` 锁权限，密钥单独保存一份。`N8N_ENCRYPTION_KEY` 用于加密工作流里保存的凭据，丢了会很麻烦。

**第三步，写 Caddyfile。** Caddy 是反向代理，最大的好处是证书全自动申请和续期：


n8n.yourdomain.com {
    reverse_proxy n8n:5678
}


**第四步，用 docker-compose 把 n8n 和 Caddy 跑起来。** 参考写法：

yaml
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    env_file: .env
    volumes:
      - ~/.n8n:/home/node/.n8n
  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
volumes:
  caddy_data:


执行 `docker compose up -d`，访问 `https://n8n.yourdomain.com` 即可。这样 n8n 不直接暴露在公网，TLS 由 Caddy 统一处理，webhook 地址也和 `WEBHOOK_URL` 对上了。

## 部署之后：更新和备份怎么做

Docker 方式更新 n8n 只要两步。先拉新镜像 `docker pull n8nio/n8n:latest`，然后停掉并删除旧容器，重新执行原来的 `docker run` 命令。数据在 `~/.n8n` 卷里，不会丢。

备份的重点就三样：`~/.n8n` 数据目录、`.env` 配置文件、Caddyfile。前面按 `/opt/docker-data/<服务名>/` 组织目录的好处在这里兑现——整个目录打一个包，恢复的时候原样解回去就行。用一键安装的话，重新安装一次系统再导入备份也可以，但自由度低一些。

如果完全不想碰维护，第三方指南一般会把 Hostinger、Cloudways 这类托管平台作为替代方案——有人替你管安全和更新，代价是价格高出不少。愿意自己维护的话，RackNerd 的 ctrl 面板还提供网络防火墙、HTML5 VNC 控制台和系统重装，日常管理不太需要额外买什么东西。

## RackNerd 部署 n8n 常见问题

**1 GB 内存真的够跑 n8n 吗？**

够起步。有第三方指南实测后认为 1 GB 方案能轻松承载几十个活跃工作流，n8n 镜像本身只有约 200 MB。工作流变重、开始跑大数据量时再升级套餐，RackNerd 支持原地升配，通常只需要一次重启。

**第二年续费会涨价吗？**

不会。官方特惠页明确写着这些价格是终身循环的，只要续费就一直按原价。这一点比很多“首年低价、续费翻倍”的主机方案厚道。

**直接把 5678 端口暴露在公网安全吗？**

不建议长期这么用。基础操作是强密码、定期 `apt update && apt upgrade`、用 UFW 管住入站；正式使用时按上面的方法配域名和 HTTPS，反代之后关掉 5678 的对外访问。RackNerd 的 ctrl 面板还有一层网络防火墙，可以直接在面板里配入站规则。

**SSH 连接被拒绝怎么办？**

大概率是服务器还在开通中，多数 VPS 会在下单后几分钟内就绪，等 5–10 分钟再试。超过 15 分钟还连不上，直接开工单找支持。

**n8n 能和网站共用一台 VPS 吗？**

可以。Docker 部署天然隔离，配合反向代理按域名分流，一台机器同时跑 n8n 和 WordPress 之类的站点是常见玩法，4 GB 套餐做这种组合会比较从容。顺便说一句，RackNerd 已经进入 2026 年 Inc. 5000 榜单（全国第 3693 位），在这个价位的供应商里算有迹可循的一家。

## 按你的情况做最后决定

只想尽快用上 n8n、不想碰命令行，直接上 2 GB 套餐走一键安装；预算优先、愿意花十分钟敲命令，1 GB 加 Docker 是最划算的组合，特惠价年付 21.99 美元，续费同价；工作流重或者打算一机多用，再考虑 4 GB。

下单前确认两件事就行：机房位置挑离你或你的服务对象近的；打算配 HTTPS 的话，提前把域名和 Cloudflare 准备好。确认无误就可以👉 [查看 RackNerd 全部特惠套餐和可选机房](https://bit.ly/my-rn)，开通邮件到了之后，本文任选一条路线照着走。
