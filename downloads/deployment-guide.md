# 线索数据中转工具功能与部署说明

## 1. 功能概述

第三方平台的线索推送可能包含多层 JSON、嵌套对象或数组，而微盛广告管家「自定义广告平台」使用单层字段进行线索映射。客户可将本工具部署在服务器上，接收第三方 Webhook，校验推送来源，转换字段结构，再将线索推送至微盛接收接口。

**业务结果**是让客户在目标线索池获得包含手机号、姓名和来源等信息的线索。中转服务处理接收、验签、转换和转发，字段映射及入库结果由微盛广告管家管理。

## 2. 链路与职责

| 环节 | 处理内容 |
| --- | --- |
| 第三方平台 | 按其接口规范生成线索事件，将 JSON 通过 HTTPS POST 推送到客户提供的回调 URL |
| 客户公网 HTTPS 入口 | 使用有效 TLS 证书接收请求，将请求体和验签所需请求头原样转给中转服务 |
| 中转服务 | 校验签名，识别线索事件，解析嵌套 JSON，生成单层字段，去重并转发 |
| 微盛广告管家 | 接收转换后的 JSON，按来源字段映射写入目标线索池，提供同步结果 |

`第三方平台 → 公网 HTTPS 入口 → 中转服务 → 微盛接收接口 → 目标线索池`

## 3. 数据转换与映射

当前版本以 JSON 对象作为输入。转换会展开嵌套对象，保留带点号的原始路径。叶子字段值无冲突时，也会生成同名的顶层字段。单元素数组会转换为其元素的文本，多元素标量数组会拼接成文本。空值和空数组会跳过。有冲突的同名叶子字段应使用完整路径映射。

以下为结构示例。所有标识和手机号均为占位符。

**来源报文**

```json
{
  "client_id": "LEAD_ID_PLACEHOLDER",
  "client_name": "示例线索",
  "source": "示例渠道",
  "customer_attrs": {
    "tel": ["PHONE_PLACEHOLDER"]
  }
}
```

**转发报文**

```json
{
  "client_id": "LEAD_ID_PLACEHOLDER",
  "client_name": "示例线索",
  "source": "示例渠道",
  "customer_attrs.tel": "PHONE_PLACEHOLDER",
  "tel": "PHONE_PLACEHOLDER"
}
```

**微盛字段映射示例**

| 来源字段 | 线索字段 |
| --- | --- |
| `tel` | 手机号码 |
| `client_name` | 线索姓名 |
| `source` | 渠道来源 |

在微盛「自定义广告平台」选择目标线索池，按实际转发字段设置映射，并复制完整的「线索接收地址」。入库验收使用客户批准的有效测试手机号，示例中的文本占位符用于本地自检。

## 4. 平台兼容性

本包提供可部署的中转程序和一个内置来源适配器。不同平台能否直接接入，取决于推送协议、验签与事件字段是否匹配。

| 对接项 | 当前内置适配器 | 接入其他平台时核对 |
| --- | --- | --- |
| 传输方式 | HTTPS POST，JSON 对象，请求体默认上限 1 MiB | 请求方法、内容类型、字符编码与请求体大小 |
| 来源验证 | 读取 `X-MeiqiaAI-Signature`，按原始请求字节拼接约定密钥后计算 SHA-1 | 实际签名头、算法、编码和签名原文 |
| 事件结构 | 接收顶层线索对象，可解包指定形态的 `data` 对象，识别渠道授权类非线索事件 | 事件类型、报文包裹层、线索 ID 和手机号位置 |
| 去重 | 优先读取支持的事件 ID 请求头，其次使用规范化 JSON 哈希 | 平台事件 ID 是否稳定，重推时是否沿用同一 ID |
| 字段转换 | 将多层结构转换为单层 JSON，业务字段在微盛配置映射 | 必填字段、字段含义、同名冲突和数组处理方式 |

若上游协议不同，应先适配来源验证和数据解析，再使用该平台的脱敏样例验证转换结果。具备匹配的适配器和字段规则后，再接入正式回调。

## 5. 响应与运行行为

| 中转响应 | 处理结果 |
| --- | --- |
| `200 accepted` | 微盛接收接口返回成功 |
| `200 duplicate` | 已处理过的事件，本次跳过转发 |
| `200 ignored` | 已识别的非线索事件 |
| `400` / `411` / `413` | 请求格式、长度或大小需调整 |
| `401` | 签名缺失或校验失败 |
| `404` | 请求路径与部署配置不匹配 |
| `502` | 转发失败或微盛返回业务错误 |
| `503` | 同一事件正在处理 |

转发默认单次超时 8 秒，最多尝试 3 次，适用于网络错误和可重试的响应。SQLite 用于保存去重和处理状态。最终入库结果在微盛「同步记录」和目标线索池核对。

## 6. 环境与配置

| 项目 | 要求 |
| --- | --- |
| 服务器 | Linux 使用 Python 3.10+ 和 `systemd`。Windows 10/11 x64 使用包内便携运行时 |
| 公网入口 | 一个第三方平台可访问的 HTTPS 域名或地址及有效证书。现有 Nginx、网关或隧道可承担入口 |
| 监听 | 服务默认绑定 `127.0.0.1:8080`，HTTPS 代理转发到该地址 |
| 接收路径 | 内置适配器默认使用 `/webhook/meiqia`，公网健康检查路径为 `/healthz` |
| 出网 | 中转服务器需要能通过 HTTPS 访问配置的微盛接收地址 |
| 凭据保护 | 真实接收地址与验签密钥由客户在目标主机录入，部署和排障时以脱敏状态为准 |

| 配置项 | 填写内容 |
| --- | --- |
| `WSHOTO_CALLBACK_URL` | 客户微盛「自定义广告平台」生成的完整线索接收地址 |
| `MEIQIA_WEBHOOK_SECRET` | 当前内置适配器使用的 Webhook 验签密钥，由上游按其配置提供 |
| `PUBLIC_BASE_URL` | 客户已经配置可达的 HTTPS 根地址，例如 `https://lead.example.com` |
| `HOST` / `PORT` | 同机网关部署保留 `127.0.0.1` / `8080` |
| `ALLOW_UNSIGNED_WEBHOOKS` | 正式部署保持 `false` |
| `CAPTURE_RAW_PAYLOADS` | 日常部署保持 `false` |

在上游填写公网根地址与接收路径拼接出的 URL，例如 `https://lead.example.com/webhook/meiqia`。填写 `PUBLIC_BASE_URL` 用于地址展示和预检，公网可达性由 HTTPS 网关或隧道提供。

## 7. 部署

### Linux 服务器

进入包内 `linux/`。以下命令适用于全新安装，执行身份需具备管理员权限。按已有服务器策略使用服务账号，并确认 `/usr/bin/python3` 指向支持的 Python 版本。

```bash
python3 scripts/self_test.py

sudo install -d -m 0755 /opt/meiqia-wshoto-lead-bridge
sudo cp -R . /opt/meiqia-wshoto-lead-bridge/
sudo useradd --system --user-group \
  --home-dir /opt/meiqia-wshoto-lead-bridge \
  --shell /usr/sbin/nologin wshoto-bridge
sudo install -d -m 0700 -o wshoto-bridge -g wshoto-bridge \
  /opt/meiqia-wshoto-lead-bridge/data
sudo install -m 0640 -o root -g wshoto-bridge .env.example \
  /opt/meiqia-wshoto-lead-bridge/.env
sudoedit /opt/meiqia-wshoto-lead-bridge/.env
```

按配置表在目标主机填入客户自己的值。参考 `deploy/nginx.conf.example` 将公网 HTTPS 的回调和健康检查转发到本机服务，配置实际域名与证书，并校验代理设置。使用现有网关时，确保保留原始请求体和签名头。随后启动服务并运行公网预检。

```bash
sudo install -m 0644 deploy/meiqia-wshoto-bridge.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now meiqia-wshoto-bridge.service
sudo /usr/bin/python3 /opt/meiqia-wshoto-lead-bridge/scripts/preflight.py --human
```

### Windows x64

1. 解压到可写目录，进入 `windows-x64/`，运行 `start_control_panel.bat`。
2. 填写控制台的「公网 URL」「微盛 Webhook 地址」和「美洽 Webhook 密钥」，保持验签开启。
3. 保存配置并启用服务。配置公网 HTTPS 代理，将控制台生成的回调地址填到匹配的上游 Webhook。
4. 在该目录用 PowerShell 运行本地自检和公网预检。

```powershell
.\python.exe scripts\self_test.py
.\python.exe scripts\preflight.py --human
```

Windows 控制台可配置在用户登录后自动启用服务。控制台监听在本地，部署时通过 HTTPS 公开线索回调和健康检查路径。

## 8. 验收与排查

| 验收项 | 通过标准 |
| --- | --- |
| 本地自检 | `self_test.py` 显示 6/6 项通过 |
| 公网及验签 | `preflight.py` 通过，公网 `/healthz` 返回 `status=ok`，签名探针返回 `200 ignored` |
| 字段转换 | 微盛查看传入参数时，能找到用于映射的单层字段及有效测试手机号 |
| 实际入库 | 用上游真实事件推送一条经授权的测试线索，微盛同步记录显示成功，目标线索池能查到该条线索 |

本地自检使用模拟微盛。公网预检会发送一个非线索签名探针。正式入库验收由客户在上游发起测试推送。

| 常见现象 | 核对顺序 |
| --- | --- |
| 公网访问失败 | HTTPS 证书、DNS 或公网地址、网关路由、后端监听 |
| 客户推送后返回 `401` | 签名头、客户密钥、原始报文是否被代理层改写 |
| 云端返回 `502` | 微盛地址、出网连通性、微盛响应与重试状态 |
| 接口返回 `200` 但线索池无记录 | 微盛同步记录、手机号、字段映射、目标线索池 |

## 9. 持续运行与迁移

Linux 服务由 `systemd` 托管，配置开机自启和异常重启。持续接收依赖服务器、网络和 HTTPS 入口可用。证书按客户网关的运维方式续期。

升级或迁移时，安全备份配置和 SQLite 去重库 `data/bridge.sqlite3`。在新主机配置 HTTPS 并通过预检后，按切换窗口同步去重状态，将上游改到新回调地址。确认新链路完成一次真实入库后，再退役旧实例。

交付记录保留工具版本、预检结果、测试时间与微盛同步结论，凭据和客户线索字段按客户的数据管理要求处理。
