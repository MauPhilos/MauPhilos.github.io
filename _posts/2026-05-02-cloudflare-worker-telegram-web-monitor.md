---
layout: post
title: "Cloudflare Worker + Telegram 网页监控部署指南：自动提醒补货、优惠码和价格变化"
categories: [Cloudflare, 自动化监控, Telegram]
description: "本文介绍如何使用 Cloudflare Worker、KV 和 Telegram Bot 搭建一个网页自动监控系统，用于监控补货、价格变化、优惠码状态等页面变化。"
keywords: Cloudflare Worker, Telegram Bot, 网页监控, Cloudflare KV, Wrangler, 自动提醒, 补货监控, 价格监控
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# Cloudflare Worker + Telegram 网页监控部署指南：自动提醒补货、优惠码和价格变化

如果你想监控某个网页有没有补货、优惠码有没有恢复、价格有没有变化，最简单的方式不是一直手动刷新页面，而是让 Cloudflare Worker 定时帮你检查。

这个方案的优点是：**部署完成后不需要本地电脑一直开着**。  
本地电脑只负责创建项目、修改代码和部署，真正的监控任务会在 Cloudflare 云端自动运行。

本文以 VMRack 活动页为示例，演示如何部署一个 Cloudflare Worker + Telegram 的网页监控项目。

示例目标页面：

```text
https://www.vmrack.net/zh-CN/activity/2026-spring
```

---

## 一、这个监控项目适合做什么？

这个方案适合监控公开网页，例如：

- VPS 活动页
- 商品补货页面
- 优惠码领取页面
- 价格变化页面
- 限时活动页面
- 库存状态页面

只要目标页面可以被 Worker 正常访问，就可以根据页面内容做判断，然后通过 Telegram 推送通知。

---

## 二、运行原理

Cloudflare Worker 部署后，会在 Cloudflare 云端运行，不依赖本地电脑。

整体流程如下：

```text
Cloudflare Cron 定时触发
        ↓
Worker 抓取目标网页
        ↓
解析页面内容
        ↓
和 KV 里保存的上一次状态对比
        ↓
发现补货 / 低价 / 优惠码变化
        ↓
通过 Telegram Bot 通知
```

本地电脑只用于：

```text
创建项目
部署 Worker
设置密钥
修改代码
查看日志
```

部署成功后，本地 Node、终端、电脑都不需要一直开着。

---

## 三、准备条件

开始前需要准备：

```text
1. Cloudflare 账号
2. 已验证 Cloudflare 邮箱
3. Node.js 和 npm
4. Telegram Bot Token
5. Telegram Chat ID
```

先检查本地是否已经安装 Node.js 和 npm：

```bash
node -v
npm -v
```

如果能正常显示版本号，说明本地环境可用。

---

## 四、创建监控项目

先创建项目目录：

```bash
mkdir vmrack-activity-monitor
cd vmrack-activity-monitor
mkdir src
npm init -y
npm install -D wrangler
```

这里用到的 `wrangler` 是 Cloudflare 官方命令行工具，用来创建 KV、设置密钥、部署 Worker 和查看日志。

---

## 五、登录 Cloudflare

执行：

```bash
npx wrangler login
```

如果浏览器授权超时，可以试试下面这个命令：

```bash
npx wrangler login --callback-host=127.0.0.1 --callback-port=8787
```

检查是否登录成功：

```bash
npx wrangler whoami
```

如果能显示你的 Cloudflare 账号信息，就说明登录成功。

---

## 六、创建 KV 存储

KV 用来保存上一次监控状态，避免重复通知。

执行：

```bash
npx wrangler kv namespace create VMRACK_STATE
```

返回结果类似：

```toml
[[kv_namespaces]]
binding = "VMRACK_STATE"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

复制里面的 `id`，后面要填进 `wrangler.toml`。

---

## 七、创建 wrangler.toml 配置文件

在项目根目录创建 `wrangler.toml`：

```bash
cat > wrangler.toml <<'TOML'
name = "vmrack-activity-monitor"
main = "src/index.js"
compatibility_date = "2026-04-30"

kv_namespaces = [
  { binding = "VMRACK_STATE", id = "这里换成你的KV_ID" }
]

[triggers]
# 每15分钟检查一次。Cloudflare Cron 使用 UTC 时间。
crons = ["*/15 * * * *"]

[vars]
TARGET_URL = "https://www.vmrack.net/zh-CN/activity/2026-spring"
MAX_PRICE = "35"
TOML
```

把这里：

```text
这里换成你的KV_ID
```

替换成刚刚创建 KV 时返回的 `id`。

---

## 八、创建 Worker 监控代码

在 `src/index.js` 中放入你的监控代码。

可以用 nano 编辑：

```bash
nano src/index.js
```

也可以用 VS Code 打开：

```bash
code src/index.js
```

当前 VMRack 监控逻辑主要做这些事：

```text
1. 抓取 TARGET_URL 页面
2. 解析套餐名、价格、售罄状态
3. 检查是否有 ≤ MAX_PRICE 的可购买套餐
4. 检查“优惠码已领完”状态是否变化
5. 与 KV 中上一次状态对比
6. 有变化时发送 Telegram 通知
```

需要注意：如果换成其他网站，通常不只是修改网址，还可能需要修改 `src/index.js` 里的页面解析规则。

因为不同网站的 HTML 结构不同，套餐名、价格、售罄状态、优惠码状态的写法也不一样。

---

## 九、设置 Telegram Bot

### 1. 创建 Bot

打开 Telegram，搜索：

```text
@BotFather
```

发送：

```text
/newbot
```

按提示创建 bot，最后会拿到一个 Bot Token。

Bot Token 一般长这样：

```text
1234567890:AAxxxxxxxxxxxxxxxxxxxxxxxx
```

这个就是后面要用的：

```text
TG_BOT_TOKEN
```

---

### 2. 获取 Chat ID

先打开你的 bot，给它发一句：

```text
test
```

然后在浏览器打开：

```text
https://api.telegram.org/bot你的TOKEN/getUpdates
```

在返回内容里找到类似下面的内容：

```json
"chat": {
  "id": 5598873649,
  "type": "private"
}
```

这里的数字就是：

```text
TG_CHAT_ID
```

注意：

```text
bot 自己的 id 不能当 TG_CHAT_ID
bot 用户名也不能当 TG_CHAT_ID
```

错误示例：

```text
@get2026_bot
get2026_bot
8672540891
```

正确示例：

```text
5598873649
```

如果填错，常见结果就是 Telegram 403 错误。

---

## 十、设置 Cloudflare Secrets

Telegram Token 不要写进代码，也不要写进 `wrangler.toml`。  
正确做法是使用 Cloudflare Secret。

设置 Bot Token：

```bash
npx wrangler secret put TG_BOT_TOKEN
```

然后输入你的 bot token。

设置 Chat ID：

```bash
npx wrangler secret put TG_CHAT_ID
```

然后输入你的个人 chat id。

查看 secret 名称是否存在：

```bash
npx wrangler secret list
```

Cloudflare 只显示 secret 名称，不显示具体值，这是正常的。

---

## 十一、部署 Worker

执行：

```bash
npx wrangler deploy
```

第一次部署时，如果提示注册 `workers.dev` 子域名，可以输入一个唯一名称，例如：

```text
maurilos-monitor
```

不要输入：

```text
https://
.workers.dev
空值
```

部署成功后会得到一个 Worker 地址，例如：

```text
https://vmrack-activity-monitor.maurilos-monitor.workers.dev
```

---

## 十二、测试监控是否正常

部署完成后，建议按顺序测试。

### 1. 测试 Worker 是否在线

打开：

```text
https://你的worker地址/health
```

正常返回：

```text
ok
```

说明 Worker 已经可以访问。

---

### 2. 测试 Telegram 通知

打开：

```text
https://你的worker地址/test
```

如果 Telegram 收到测试消息，说明通知链路正常。

这个接口只测试 Telegram，不代表完整监控逻辑一定正常。

---

### 3. 手动执行一次完整监控

打开：

```text
https://你的worker地址/run
```

正常情况下会返回 JSON，并且 Telegram 会收到一条手动检查结果。

`/run` 才是完整执行一次监控流程。

---

## 十三、查看 Worker 日志

终端执行：

```bash
npx wrangler tail
```

保持窗口打开，然后访问：

```text
https://你的worker地址/test
```

或者：

```text
https://你的worker地址/run
```

如果 Worker 出错，真实错误信息会显示在终端里。

`wrangler tail` 只是用来看日志，不是让 Worker 运行的发动机。  
Worker 部署后会在 Cloudflare 云端运行。

---

## 十四、常见错误和解决方法

### 1. Cloudflare 邮箱未验证

错误提示：

```text
You need to verify your email address to use Workers.
```

处理方法：

```text
Cloudflare Dashboard
→ My Profile
→ 验证邮箱
```

验证后重新执行部署命令。

---

### 2. Telegram 403

错误提示：

```text
Forbidden: bots can't send messages to bots
```

原因通常是：

```text
TG_CHAT_ID 填成了 bot 用户名或 bot 自己的 id
```

解决方法：

```bash
npx wrangler secret put TG_CHAT_ID
```

重新输入你的个人 chat id，例如：

```text
5598873649
```

---

### 3. Telegram 401

错误提示：

```text
Telegram failed: 401
```

原因通常是：

```text
TG_BOT_TOKEN 错了
```

解决方法：

```bash
npx wrangler secret put TG_BOT_TOKEN
```

重新输入 BotFather 给你的 token。

---

### 4. Worker 1101

浏览器显示：

```text
Worker threw exception
Error code: 1101
```

处理方法：

```bash
npx wrangler tail
```

然后重新访问：

```text
https://你的worker地址/test
```

或者：

```text
https://你的worker地址/run
```

查看终端里的真实错误。

---

## 十五、修改监控频率

在 `wrangler.toml` 中修改：

```toml
[triggers]
crons = ["*/15 * * * *"]
```

常用配置如下。

每 15 分钟一次：

```toml
crons = ["*/15 * * * *"]
```

每小时一次：

```toml
crons = ["0 * * * *"]
```

每天 3 次：

```toml
crons = ["0 1,9,17 * * *"]
```

注意：Cloudflare Cron 使用 UTC 时间。

北京时间换算示例：

```text
UTC 01:00 = 北京时间 09:00
UTC 09:00 = 北京时间 17:00
UTC 17:00 = 北京时间 次日 01:00
```

修改后重新部署：

```bash
npx wrangler deploy
```

---

## 十六、修改监控网址

如果新页面结构和当前页面类似，只需要改 `wrangler.toml`：

```toml
[vars]
TARGET_URL = "新的网址"
MAX_PRICE = "35"
```

然后重新部署：

```bash
npx wrangler deploy
```

如果新网站页面结构不同，例如套餐名、价格、售罄状态格式不同，就需要修改 `src/index.js` 里的解析规则。

简单说：

```text
同类页面：大概率只改 TARGET_URL
不同网站：大概率要改解析逻辑
```

---

## 十七、新项目快速复制流程

如果你以后要监控新的网页，可以按下面流程新建项目。

创建新项目：

```bash
mkdir new-monitor
cd new-monitor
mkdir src
npm init -y
npm install -D wrangler
```

创建 KV：

```bash
npx wrangler kv namespace create NEW_MONITOR_STATE
```

创建 `wrangler.toml`：

```toml
name = "new-monitor"
main = "src/index.js"
compatibility_date = "2026-04-30"

kv_namespaces = [
  { binding = "NEW_MONITOR_STATE", id = "你的KV_ID" }
]

[triggers]
crons = ["*/15 * * * *"]

[vars]
TARGET_URL = "https://example.com"
MAX_PRICE = "35"
```

设置 Telegram：

```bash
npx wrangler secret put TG_BOT_TOKEN
npx wrangler secret put TG_CHAT_ID
```

部署：

```bash
npx wrangler deploy
```

测试：

```text
https://你的worker地址/test
https://你的worker地址/run
```

---

## 十八、本地文件说明

### wrangler.toml

Cloudflare Worker 配置文件。

主要包含：

```text
Worker 名称
入口文件
KV 绑定
Cron 定时规则
环境变量
```

---

### src/index.js

监控逻辑代码。

主要包含：

```text
网页抓取
内容解析
状态对比
Telegram 通知
测试接口
手动运行接口
```

---

### node_modules/

本地依赖目录，不需要上传到 GitHub。

---

### package.json

Node 项目配置文件。

---

### package-lock.json

依赖锁定文件，用来保证依赖版本稳定。

---

## 十九、几个关键提醒

部署前重点记住这几件事：

1. 本地电脑不需要一直开着。
2. Cloudflare Worker 部署后会在云端自动运行。
3. Telegram Token 不要写进代码，也不要发给别人。
4. Chat ID 是数字，不是 bot 用户名。
5. 换网站时，不一定只改 URL，可能还要改解析逻辑。
6. `/test` 只测试 Telegram 通知链路。
7. `/run` 才是手动执行完整监控。
8. `wrangler tail` 只是查看日志，不是运行 Worker 的必要条件。

---

## 总结

这个 Cloudflare Worker + Telegram 的监控方案，适合用来监控补货、优惠码、价格变化和活动页面状态。

它的核心价值是：**不用本地挂机，不用一直刷新网页，页面有变化时自动通知你。**

后续如果要监控其他网站，可以直接复制项目结构，再根据新页面修改：

```text
TARGET_URL
MAX_PRICE
src/index.js 解析规则
```

如果只是自己项目内部使用，也可以把这份文档放到每个监控项目根目录，文件名使用：

```text
README.md
```

以后新建监控项目时，照着这份流程走就行。