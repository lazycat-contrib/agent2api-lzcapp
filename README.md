# agent2api-lzcapp

[Agent2API](https://github.com/aimod-cc/agent2api)（多提供商本地网关）的懒猫微服打包：**只发喵喵商店，镜像模式**（`delivery.mode: mirror`）。

- **镜像**：`aimodcc/agent2api:3.0.1`（Docker Hub；容器内 **3065** 一个端口既是面板也是 OpenAI 兼容网关，数据在 `/data`）。镜像模式只做只读校验，交付时把 manifest 的镜像行改写成 `docker.1ms.run/...`，不搬镜像到懒猫镜像源。
- **版本跟踪**：`channel: stable` + `sort: semver` + `tag_regex: '^[0-9]+\.[0-9]+\.[0-9]+$'`（Docker Hub 上还有 `latest` / `dev` 等别名，正则只收三段式正式版）。
- **运行用户**：镜像基于 `debian:bookworm-slim`、默认 root（Dockerfile 无 `USER`）；平台建的 `/lzcapp/var/agent2api` 属主是 root，SQLite 要写进去——manifest 里写 **`user: root`**（不用 `run_as`）。
- **设置向导（4 个参数，其中 2 个必填）**：

  | 向导参数 | 类型 | 必填 | 渲染到容器 |
  |---|---|---|---|
  | `admin_user` | string | ✅（默认 `admin`） | `AGENT2API_ADMIN_USER` |
  | `admin_password` | secret | ✅ | `AGENT2API_ADMIN_PASSWORD`（明文填，启动时转 bcrypt 哈希） |
  | `allow_no_key` | string(0/1) | 可选（默认 `0`） | `AGENT2API_ALLOW_NO_KEY=0/1`（0 = fail-closed） |
  | `captcha_enabled` | string(1/0) | 可选（默认 `1`） | `AGENT2API_CAPTCHA_ENABLED=1/0`（1 = 人机验证开） |

  两个必填参数保证装完即预置管理员、登录页不再有开放注册入口。两个开关用 0/1 字符串直填：上游按**字面值**判断（`ALLOW_NO_KEY` 只有 `"1"` 生效、captcha 只有 `"0"` 关闭），bool 渲染出的 `true/false` 会让两个开关都失灵，所以这里不用 bool 类型。
- **健康检查**：`curl -fsS http://127.0.0.1:3065/health`（镜像自带 curl，与镜像自身 HEALTHCHECK 一致；`/health` 见上游 README 验证一节）。
- **路由**：`/` → `agent2api:3065`，`public_path: [/]`——`/v1` 要给任意 API 客户端直接调用，不能要求微服账号；面板由预置管理员 + 人机验证保护，`/v1` 在没建 Key 前 fail-closed。
- **持久化**：`/lzcapp/var/agent2api:/data`（SQLite 库、配置、日志；镜像内 `AGENT2API_PROXY_HOME=/data`）。
- **未暴露的镜像变量**：`AGENT2API_PANEL_PORT`（面板分端口，微服上单入口无意义）、`AGENT2API_HOST` / `AGENT2API_PROXY_PORT`（用镜像默认 0.0.0.0:3065）——需要时再加。
- **图标**：上游 `desktop-tauri/src-tauri/icons/icon.png`（512×512）。

## 模板预检

`lzc-manifest.yml` 的环境变量是 4 条 `- "KEY={{ .U.x }}"` 直填（没有条件模板）；渲染结果（含带空格 / 冒号 / 井号 / 引号的密码）过 YAML 解析后再提交。

## 商店状态

- **喵喵商店**：`cloud.lazycat.app.agent2api` 由 Action 发布（`stores.private`）。
- **官方商店**：镜像模式不满足官方商店「镜像必须来自 `registry.lazycat.cloud`」的要求，只发喵喵商店。
