# 在 Ubuntu 服务器上部署个人 QQ 聊天助手

> 使用 Ubuntu + 1Panel + AstrBot + NapCat，将一个闲置 QQ 号接入大语言模型。
>
> 本教程面向个人学习和实验用途。请遵守 QQ、云厂商、模型服务商及相关法律法规，不要用于骚扰、刷屏、诈骗或其他违规行为。

## 目录

- [效果与架构](#效果与架构)
- [安全提醒](#安全提醒)
- [准备工作](#准备工作)
- [1. 购买并初始化 Ubuntu 服务器](#1-购买并初始化-ubuntu-服务器)
- [2. 安装 AstrBot](#2-安装-astrbot)
- [3. 部署 NapCat](#3-部署-napcat)
- [4. 配置端口和网络连接](#4-配置端口和网络连接)
- [5. 登录 QQ](#5-登录-qq)
- [6. 配置 AstrBot](#6-配置-astrbot)
- [7. 安装插件](#7-安装插件)
- [故障排查](#故障排查)
- [仓库结构](#仓库结构)
- [参考资料](#参考资料)

## 效果与架构

最终链路如下：

```text
QQ 用户
   │
   ▼
NapCat（QQ 协议侧）
   │  OneBot v11 / WebSocket
   ▼
AstrBot（机器人、模型和插件）
   │
   ▼
DeepSeek 或其他兼容提供商
```

本仓库只保存教程、脱敏截图和示例配置，不保存 QQ 登录数据、NapCat Token、AstrBot 密码或模型 API Key。

## 安全提醒

1. **不要把真实 IP、实例 ID、登录 Token、QQ 二维码、AstrBot 初始密码或 API Key 提交到 Git。** 本仓库中的截图已经做了脱敏处理。
2. NapCat WebUI、1Panel 和 AstrBot 管理后台都属于管理入口。首次配置可以临时开放端口，但完成后建议使用安全组、1Panel 防火墙、VPN 或 SSH 隧道限制访问来源。
3. `127.0.0.1` 只表示当前容器或当前主机。NapCat 和 AstrBot 位于不同容器时，不能直接把 `127.0.0.1` 当作对方地址。
4. 建议使用专门的闲置 QQ 号，不要使用包含重要联系人、支付信息或敏感资料的主账号。
5. API Key 建议单独创建、限制额度，并在泄露后立即撤销和重新生成。

## 准备工作

- 一台可以安装 Ubuntu 22.04/24.04 的云服务器，建议至少 2 vCPU、2 GiB 内存。
- 一个闲置 QQ 号，用于扫码登录 NapCat。
- 一个模型服务商账号和 API Key，例如 DeepSeek。
- 可以通过 SSH 或云厂商网页终端登录服务器。
- 基本的 Linux、Docker、端口和 WebSocket 常识。

> 云服务器规格和价格会随活动变化。购买时请以云厂商当前页面为准；如果云厂商提供带宽赠送的 Ubuntu 镜像，可以优先选择，但不要把某个镜像名称当成必须条件。

## 1. 购买并初始化 Ubuntu 服务器

### 1.1 选择系统和镜像

在云厂商控制台创建轻量应用服务器或普通云服务器：

1. 选择 Ubuntu 22.04 LTS 或 Ubuntu 24.04 LTS。
2. 选择适合自己的地域和配置。
3. 记录服务器公网 IP，但不要将真实 IP 写进公开仓库。
4. 创建后打开服务器详情，确认实例状态正常。

![选择 Ubuntu 镜像](images/02-choose-ubuntu-image.png)

### 1.2 安装 1Panel

通过云厂商网页终端或 SSH 登录服务器，执行 1Panel 官方提供的快速安装命令：

```bash
bash -c "$(curl -sSL https://resource.fit2cloud.com/1panel/package/v2/quick_start.sh)"
```

安装完成后，终端通常会输出 1Panel 访问地址、用户名和初始密码。请立刻复制到密码管理器中，并在首次登录后修改密码。

![在终端安装 1Panel](images/03-install-1panel.png)

> 1Panel 的访问端口可能由安装过程生成，不一定是固定值。不要照抄截图里的端口；以你自己的终端输出和面板设置为准。

### 1.3 放行 1Panel 访问端口

如果无法打开 1Panel：

- 在云厂商安全组/防火墙中放行安装输出的 1Panel 端口。
- 如果启用了 1Panel 自带防火墙，也要添加对应规则。
- 只允许自己的公网 IP 访问管理端口，不建议对 `0.0.0.0/0` 长期开放。

![服务器详情（已脱敏）](images/04-server-details-redacted.png)
![云防火墙配置](images/05-firewall-rule.png)

## 2. 安装 AstrBot

进入 1Panel 后：

1. 打开 **应用商店**。
2. 搜索 `AstrBot`。
3. 点击安装。
4. 按需设置持久化目录和管理端口。
5. 如果安装页面有“放行端口”选项，请只勾选确实需要的端口。

AstrBot WebUI 默认常见端口为 `6185`，但不同版本或安装方式可能有所不同，请以安装页面和容器日志为准。

![在 1Panel 应用商店安装 AstrBot](images/06-1panel-install-astrbot.png)
![AstrBot 安装进度](images/07-astrbot-install-progress.png)

## 3. 部署 NapCat

### 3.1 使用 1Panel 编排创建容器

在 1Panel 中进入 **容器 → 编排 → 创建**，粘贴下面的示例配置。仓库中也提供了同样的文件：[`docker-compose.napcat.yml`](docker-compose.napcat.yml)。

```yaml
services:
  napcat:
    image: mlikiowa/napcat-docker:latest
    container_name: napcat
    ports:
      - "6099:6099" # NapCat WebUI；完成配置后建议限制来源
      - "3000:3000"
      - "3001:3001"
    environment:
      - NAPCAT_UID=0
      - NAPCAT_GID=0
    volumes:
      - ./napcat-data/config:/app/napcat/config
      - ./napcat-data/QQ:/app/.config/QQ
    restart: unless-stopped
```

说明：

- `6099`：NapCat WebUI，首次配置会用到。
- `3000`、`3001`：NapCat 的 HTTP/WebSocket 相关端口，是否需要公网暴露取决于你的连接方式。
- `./napcat-data`：登录数据和配置目录，不要提交到 Git。
- `latest` 便于快速体验，但不利于复现；长期运行时建议固定到经过验证的版本标签，并定期备份配置。

![创建 NapCat 编排](images/08-create-napcat-compose.png)
![NapCat 编排配置](images/09-napcat-compose-editor.png)

创建并启动容器后，进入 **容器 → NapCat → 日志**，确认容器已经正常启动。

## 4. 配置端口和网络连接

### 4.1 需要关注的端口

| 端口 | 用途 | 建议 |
| --- | --- | --- |
| 1Panel 安装输出的端口 | 1Panel 管理后台 | 仅允许自己的 IP 访问 |
| `6185` | AstrBot WebUI（以实际安装结果为准） | 仅允许自己的 IP 访问 |
| `6099` | NapCat WebUI | 配置完成后限制来源或改用 SSH 隧道 |
| `3000`、`3001` | NapCat 连接相关端口 | 按实际拓扑开放，不要盲目全网开放 |
| `6199` | AstrBot 的 OneBot v11 WebSocket 服务示例端口 | 尽量只在容器网络/内网可达 |

云安全组和 1Panel 防火墙需要分别检查。只配置其中一处，另一处仍可能拦截连接。

![云安全组配置](images/10-firewall-port-rule.png)
![1Panel 防火墙配置](images/11-1panel-firewall-rule.png)

### 4.2 配置 OneBot v11 WebSocket

推荐使用 **NapCat WebSocket 客户端 → AstrBot WebSocket 服务端** 的方向，避免同时维护两套重复连接：

1. 在 AstrBot 的平台配置中启用 OneBot v11，并创建 WebSocket 服务端。
2. 示例端口填写 `6199`，路径填写 `/ws`；如果设置了 Token，客户端必须使用相同 Token。
3. 在 NapCat 的 **网络配置 → WebSocket 客户端** 中，新建客户端。
4. 将 URL 填为：

```text
ws://<AstrBot可达地址>:6199/ws
```

`<AstrBot可达地址>` 必须是 NapCat 容器能够访问的地址：

- 同一个 Docker 网络：优先使用 AstrBot 的服务名或容器名。
- 不同编排/不同网络：使用宿主机或网关地址，并确认宿主机端口映射正确。
- 不要在跨容器场景中填写 `127.0.0.1`，因为它通常指向 NapCat 自己。

![NapCat 网络配置（已脱敏）](images/13-napcat-network-config-redacted.png)
![WebSocket 服务端配置](images/14-websocket-server-config.png)
![容器地址（已脱敏）](images/15-container-ip-redacted.png)
![WebSocket 客户端配置（已脱敏）](images/16-websocket-client-config-redacted.png)

## 5. 登录 QQ

### 5.1 获取 NapCat WebUI Token

在 NapCat 容器日志中找到 WebUI Token。然后在浏览器访问：

```text
http://<服务器公网IP>:6099
```

输入 Token 后进入 WebUI，选择闲置 QQ 号并扫码登录。Token、二维码和公网 IP 都属于敏感信息，不要截图公开。

![NapCat Token 和二维码（已脱敏）](images/12-napcat-token-redacted.png)

### 5.2 确认连接状态

登录成功后：

- NapCat 日志不应持续出现登录失败或重连错误。
- AstrBot 的 OneBot 平台状态应显示在线或已连接。
- 用另一个 QQ 号发送一条简单消息，观察 AstrBot 是否收到事件。

## 6. 配置 AstrBot

### 6.1 首次登录

在 AstrBot 容器日志中查找初始登录信息。首次登录后会要求修改密码，请使用高强度密码，不要继续使用日志中的初始密码。

![AstrBot 登录信息（敏感信息已脱敏）](images/17-astrbot-login-redacted.png)
![修改 AstrBot 密码](images/18-change-password.png)\n![AstrBot 首页快速引导](images/19-astrbot-welcome.png)

### 6.2 选择模型提供商

以 DeepSeek 为例：

1. 在 DeepSeek 开放平台创建 API Key。
2. 给账户充值或确认账户有可用额度。
3. 在 AstrBot 的模型提供商配置中，新增 DeepSeek。
4. 填写 API Base URL：

```text
https://api.deepseek.com/v1
```

5. 粘贴 API Key，并点击“获取模型列表”。
6. 从当前实际返回的模型列表中选择一个可用模型。

模型名称和可用性会随服务商调整，不要把截图里的某个模型名称当成永久固定值。API Key 只应保存在服务器本地配置中。

![DeepSeek API Keys（已脱敏）](images/20-deepseek-api-keys-redacted.png)
![DeepSeek 账户/充值页面](images/21-deepseek-billing.png)
![AstrBot 提供商配置](images/22-astrbot-provider-config.png)
![选择模型](images/23-select-model.png)

### 6.3 启用 QQ 平台

在 AstrBot 中：

1. 打开平台配置。
2. 选择 **OneBot v11**。
3. 检查 WebSocket 服务端端口、路径和 Token 是否与 NapCat 客户端一致。
4. 保存后查看平台日志，确认连接状态正常。

![AstrBot 平台配置](images/24-astrbot-platform-config.png)
![AstrBot 模型配置](images/25-astrbot-model-config.png)

### 6.4 设置人格

在“人格”或类似配置页面中：

- 创建一个人格 ID。
- 编写系统提示词，明确机器人的语气、边界和不应执行的操作。
- 只授予必要的插件和工具权限。
- 先在私聊或小范围群聊中测试，再考虑扩大使用范围。

![创建人格](images/26-create-persona.png)
![人格提示词](images/27-persona-prompt.png)

保存并退出后，用测试 QQ 号发送消息验证回复。

## 7. 安装插件

AstrBot 的插件市场可以直接探索和安装插件。安装前建议：

1. 阅读插件官方文档和权限说明。
2. 检查插件是否会访问文件、执行命令或调用外部服务。
3. 先在测试环境安装，确认日志和资源占用正常。
4. 记录插件版本，避免无意间自动升级导致配置变化。

![AstrBot 插件市场](images/28-plugin-market.png)

## 故障排查

### 浏览器无法打开面板

- 检查服务是否运行：`docker ps`、1Panel 容器状态和日志。
- 检查云安全组和 1Panel 防火墙是否同时放行。
- 确认访问的是实际端口，而不是照抄截图里的端口。
- 若只允许本地访问，使用 SSH 隧道或 VPN，不要直接开放到公网。

### NapCat 与 AstrBot 无法建立 WebSocket

- 确认 WebSocket 方向只有一套配置在工作。
- 确认 URL 的主机、端口、路径 `/ws` 和 Token 完全一致。
- 从 NapCat 容器内部测试 AstrBot 地址是否可达。
- 跨容器时不要使用 `127.0.0.1`。
- 如果使用宿主机端口，检查 AstrBot 是否真的映射了 `6199`。

### QQ 扫码后仍然离线

- 检查 NapCat 日志中的二维码有效期、设备验证和重连信息。
- 使用闲置 QQ 号，不要频繁重复扫码。
- 确认容器的数据目录有持久化，否则重建容器可能丢失登录状态。

### 模型调用失败

- 检查 API Key 是否有效、是否有余额或额度。
- 检查 API Base URL 是否与服务商文档一致。
- 在 AstrBot 中重新获取模型列表，选择当前可用模型。
- 不要把 API Key 直接写进 README、截图或提交记录。

## 仓库结构

```text
.
├── README.md
├── docker-compose.napcat.yml
├── docs/
│   └── 在Ubuntu服务器上配置个人QQ聊天助手_优化版.docx
├── images/
│   ├── 01-server-purchase.png
│   ├── ...
│   └── 28-plugin-market.png
├── .gitignore
└── image-map.txt
```

## 参考资料

- [1Panel 文档](https://1panel.cn/docs/)
- [AstrBot 官方网站与文档](https://astrbot.app/)
- [NapCatQQ 官方仓库](https://github.com/NapNeko/NapCatQQ)
- [OneBot v11 标准](https://11.onebot.dev/)
- [DeepSeek 开放平台](https://platform.deepseek.com/)

## 许可证

教程文本和整理后的示例配置可按 MIT License 使用；第三方项目、Docker 镜像、截图中的商标和软件内容分别归其原作者所有。

