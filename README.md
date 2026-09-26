# Aweek Running Page

个人运动数据主页：**<https://running.aweek.me>**

基于 [yihong0618/running_page](https://github.com/yihong0618/running_page)（功能介绍、配置项说明、各运动平台同步方式都请看上游 README，这里不复述，只记录本实例自己的部署细节）。

## 这个实例在哪里

| 项目 | 说明 |
|---|---|
| 线上地址 | <https://running.aweek.me> |
| 源码 | <https://github.com/AweekJyz/running_page> |
| 服务器 | 家庭内网 Ubuntu 主机 `aweek-M9-PRO`（`192.168.4.112`，SSH 别名 `my-linux`，需经路由器 `my-wrt` 跳转） |
| 数据流 | Garmin Connect（国际版）→ intervals.icu → 服务器每半小时增量同步 |
| 托管方式 | Docker（nginx 静态站点），经已有 Cloudflare Tunnel 暴露到域名，服务器不开任何公网端口 |
| 运动目标 | 年 1080 / 月 90 / 周 21 km（`config.yml` 的 `goals`，随时可改） |

## 自动化与目录（都在服务器上）

- `~/yihong-running-page/` —— 项目工作目录（本仓库 + 本地数据）
- `deploy/sync_and_rebuild.sh` —— 同步入口：intervals.icu 增量同步 → 有新数据才生成 SVG、重建镜像、切换容器
- cron：每小时 **:17 和 :47** 自动执行；日志在 `deploy/sync.log`
- 运行中的容器：`aweek-blog`（镜像 `yihong-running-page:latest`），位于 Docker 网络 `personal-blog_blog-network`，**必须带网络别名 `blog`** —— Cloudflare Tunnel 的 ingress 指向 `http://blog:80`，重建容器时别名丢了网站立刻断

## 日常维护

```bash
# 手动触发一次同步（有新数据会自动重建并切换）
~/yihong-running-page/deploy/sync_and_rebuild.sh

# 看同步日志
tail -f ~/yihong-running-page/deploy/sync.log

# 改了 config.yml / 源码后，只重建前端并切换：
cd ~/yihong-running-page && docker build -t yihong-running-page:latest \
  --build-arg app=none --build-arg YOUR_NAME=aweek . \
&& docker stop aweek-blog \
&& docker rename aweek-blog aweek-blog-old-$(date +%s) \
&& docker run -d --name aweek-blog --network personal-blog_blog-network \
     --network-alias blog --restart unless-stopped yihong-running-page:latest
```

- 改主题 / 目标 / 头像：编辑 `config.yml` 后按上面第三条操作
- 地图底图：免费 Mapbox token，填在 `config.yml` 的 `mapbox_token`（每月 5 万次加载免费，个人站远用不到）
- 回滚：旧容器一律以 `aweek-blog-old-*` 命名保留，停新启旧即可

## 与上游的差异

- `Dockerfile`：`node:18` → `node:22`（上游版本跑不动 Vite 8）
- 页头品牌改为 `Aweek.Running.Page`，GitHub 图标指向本仓库
- 移除了上游的 GitHub Actions（CI / gh-pages / run_data_sync）并禁用了 Actions：本实例的全部自动化都在服务器 cron，GitHub 只存源码
- `.gitignore` 额外排除运动数据（轨迹文件、`data.db`、`activities.json`、生成的 SVG）与 `deploy/`、`.secrets/`——**本仓库不含任何个人隐私数据**
