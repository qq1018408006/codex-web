# Linux 共享会话与手机访问

本项目基于 **0xcaff/codex-web**。这个 fork 增加了共享会话、Desktop 工具兼容和安装时选择局域网访问的功能。

## 它解决了什么

- 网页和 `codex-shared` 终端入口连接同一个后台，可以打开同一个会话。
- 网页重启时，后台继续运行，正在执行的任务不会因为网页服务重启而停止。
- Desktop 工具连接实际打开的网页，可以读取会话、列出会话；网页重启后可以重新连接。
- 安装时选择“仅本机”或“局域网”。修复手机通过普通 HTTP 打开时因 UUID 接口缺失导致的白屏。

它不会自动把所有原生 CLI、Desktop、VS Code 会话接入同一个后台。需要共享的终端会话请通过 `codex-shared` 打开。

## 准备条件

需要 Linux、可用的 `systemd --user`、Python 3、Node.js 22.12 或更高版本、Codex CLI 0.160.1 或更高版本，以及 `websocat`。

检查是否已安装：

```bash
node --version
codex --version
python3 --version
websocat --version
systemctl --user status
```

先用原来的 Codex 登录：

```bash
codex login --device-auth
```

共享后台默认使用 `~/.codex` 中的登录和配置。项目没有修改全局 `config.toml`，也没有替换原来的 `codex` 程序。

## 安装

下载源码、安装依赖并构建：

```bash
git clone https://github.com/qq1018408006/codex-web.git
cd codex-web
npm ci
```

`npm ci` 会自动构建网页，首次需要下载约 567 MiB 的 Desktop 安装包，并需要 `unzip`、`patch`。原生依赖若没有对应的预编译版本，还需要 C/C++ 编译工具。

只在电脑本机使用：

```bash
python3 scripts/install-services.py --access local --restart
```

允许同一局域网的手机访问：

```bash
python3 scripts/install-services.py --access lan --port 8214 --restart
```

`local` 监听 `127.0.0.1`；`lan` 监听 `0.0.0.0`。手机应打开 `http://电脑的局域网IP:8214`，例如 `http://192.168.1.100:8214`。电脑和手机需要互相可达；如果系统防火墙或路由器隔离设备，仍需配置对应的网络规则。

**网页没有内置登录保护。只在可信局域网使用，不要直接暴露到公网。** 能打开网页的人可以用这个电脑用户的权限操作 Codex。公网入口需要另行配置 VPN、SSH 隧道或认证代理。

可选参数：

| 参数                           | 用途                                        |
| ------------------------------ | ------------------------------------------- |
| `--host 192.168.1.100`         | 只监听指定地址，与 `--access` 二选一        |
| `--port 9000`                  | 更改端口                                    |
| `--codex /路径/codex`          | 指定 Codex；支持官方 npm 启动器或原生二进制 |
| `--node /路径/node`            | 指定 Node.js                                |
| `--websocat /路径/websocat`    | 指定 websocat                               |
| `--app-tools /路径/server.mjs` | 指定已安装的官方 Desktop 工具插件           |
| `--dry-run`                    | 只显示安装配置，不修改服务                  |
| `--restart`                    | 安装后启动或重启网页服务                    |

网络选择保存到 `~/.config/codex-web/server.json`。以后更新时，不传网络参数就保留上次的选择。

## 日常使用

打开网页：本机访问 `http://127.0.0.1:8214`，手机访问电脑的局域网地址。

新建共享终端会话：

```bash
~/.local/bin/codex-shared
```

重新打开已有共享会话：

```bash
~/.local/bin/codex-shared resume
```

也可以传具体会话 ID：

```bash
~/.local/bin/codex-shared resume <会话ID>
```

后台由两个用户服务管理：`codex-shared.service` 负责运行会话，`codex-web.service` 负责网页。

```bash
systemctl --user status codex-shared.service codex-web.service
systemctl --user restart codex-web.service
journalctl --user -u codex-web.service -n 50
```

更新源码并重新构建后，再运行安装命令即可。安装器把程序和依赖保存成版本快照，并切换 `current` 入口；更新时不主动重启共享后台，旧任务继续使用原来的运行程序。新版本的后台程序会在后台下一次启动时生效。

用户服务随用户会话启动。需要退出登录后或无人登录时持续运行，可自行开启 systemd linger：`loginctl enable-linger "$USER"`。

## Desktop 工具的条件

程序默认查找你已安装的官方 `codex-app-tools` **0.1.5** 插件：

```text
~/.codex/plugins/cache/openai-bundled/codex-app-tools/0.1.5/server.mjs
```

项目只在本机生成兼容副本，没有把官方插件文件、账号凭据或工具数据提交到 GitHub。如果没有这个插件，共享会话仍能使用，但不会提供这组 Desktop 工具。

工具需要至少一个已经加载完成的网页标签页。已在后台加载的 CLI 会话可能继续保留旧的工具配置；请等当前任务结束、关闭该会话的客户端后再从网页打开，或者在网页里新建会话。程序不会强制终止任务来刷新工具。

已验证的是会话工具调用和网页重连。浏览器自动控制、Linux computer use，以及其他插件的登录或权限问题不属于这次修复。

## 开发与验证

```bash
npm test
```

安装流程测试（使用临时目录，并拦截服务命令，不会修改日常服务）：

```bash
npm run test:install
```

实际模型测试：

```bash
npm run test:integration
```

实际测试需要 Google Chrome、已登录的 Codex 和上述官方插件，会消耗少量模型额度。它使用独立的临时目录、后台和端口 8220，不修改日常用户服务。测试结束删除临时登录文件；私有日志保留在输出的临时目录中。

可以通过 `CODEX_TEST_CODEX`、`CODEX_TEST_BROWSER`、`CODEX_TEST_APP_TOOLS`、`CODEX_TEST_HOST`、`CODEX_TEST_PORT` 调整测试环境。比如设置 `CODEX_TEST_HOST` 为电脑的局域网 IP，可以验证普通 HTTP 地址下的移动端页面。

构建兼容补丁针对 Desktop `26.901.41123`，共享后台已测试 Codex `0.160.1`。上游 Desktop 改变内部结构时，补丁会明确报错，需要重新适配；更高的 CLI 版本仍需验证。
