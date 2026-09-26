# AGENT.md —— 操作本仓库的 Agent 必读

## 仓库性质

公开仓库，**只存源码**。上游是 [yihong0618/running_page](https://github.com/yihong0618/running_page)，通用功能/配置/各平台同步方式读上游 README。本文只讲这个实例的私有细节。

## 运行架构（改动前先读这段）

- 唯一生产环境：家庭服务器 `aweek-M9-PRO`（Ubuntu + Docker）。GitHub 不参与部署，Actions 已整仓禁用。
- 数据流：`Garmin Connect → intervals.icu → 服务器 cron（每 :17/:47）→ ~/yihong-running-page/deploy/sync_and_rebuild.sh →（有新数据才）重建镜像并切换容器`。
- 对外暴露：Cloudflare Tunnel 容器 → ingress `running.aweek.me → http://blog:80`（Docker 内网别名）。
- 因此容器有**三个硬约束**：名字 `aweek-blog`、网络 `personal-blog_blog-network`、别名 `blog`。缺任何一个，公网立刻 502/301。

## 机密（只存在于服务器，永不入库）

| 机密 | 位置 |
|---|---|
| Mapbox token | `~/yihong-running-page/config.yml` 的 `mapbox_token`（仓库内是空占位；服务器工作区该文件是"脏"的，正常） |
| intervals.icu API Key / Athlete ID | `~/yihong-running-page/deploy/sync_and_rebuild.sh` 内 |
| Cloudflare Tunnel token | `~/personal-blog/.secrets/`（不在本项目范围，勿动） |
| 同步通知 SMTP 凭据 | `~/.config/running-notify/smtp.json`（可选，缺失则静默跳过） |

Garmin 账号密码已弃用（直连被 SSO 429 限流）；如需恢复 Garmin 直连，见上游 README 的 `get_garmin_secret.py` 流程。

## 修改线上效果的正确姿势

1. 在服务器工作目录改 `config.yml` 或 `src/`
2. `docker build -t yihong-running-page:latest --build-arg app=none --build-arg YOUR_NAME=aweek .`
3. 切换容器（命令见 README「日常维护」），**必须保留 `--network-alias blog`**
4. 提交 git 前：把 `config.yml` 的 token 置空脱敏，确认 `.gitignore` 覆盖所有新增数据文件

## 常见坑

- 直接用上游未打补丁的 `Dockerfile` 构建会失败：本仓库已改为 `node:22`（Vite 8 要求 Node 20.19+）。
- `gen_svg.py` 全量生成约 320 条约 1 小时，只在新数据到达时运行；网站本体只依赖 `src/static/activities.json`，SVG 仅作分享图。
- SSH 通路：本机 `~/.ssh/config` 的 `my-linux`（经 `my-wrt` 跳转，私钥 `codex_linux`）。服务器走 WiFi，长命令用 `nohup ... &` + 日志文件，短会话轮询，不要挂长 SSH。
- 服务器仓库与 GitHub 的历史同步是单向手工的（bundle 搬运）：在服务器提交 → `git bundle create` → Mac 拉取推送。

## 数据红线（公开仓库，违反即事故）

不得提交：任何运动轨迹文件（FIT/GPX/TCX）、`run_page/data.db`、`src/static/activities.json`、生成的 SVG 路线图、任何 token / 密码 / 邮箱 / API Key。`.gitignore` 已覆盖上述项；新增数据类文件时同步补规则。
