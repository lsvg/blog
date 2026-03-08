

# 还在养龙虾吗？教你在 AWS 上让龙虾自己生龙虾

> 在 AWS 上部署OpenClaw，给它权限，然后通过聊天让它自己读取自己的信息、权限、创建服务器、部署出另一只龙虾——全程零命令。

---

## 这只龙虾是什么？

[OpenClaw](https://github.com/openclaw/openclaw) 是一个开源 AI 助手框架。装在服务器上，连上 Telegram/WhatsApp/Discord，它就能帮你执行命令、读写文件、上网搜索——本质上是一个有手有脚的 AI。

社区叫它"龙虾"（Lobster/Claw）🦞。

我的龙虾叫赛博蜥蜴 🐾。

---

## 第一步：在 AWS 上养一只龙虾

我的龙虾是养在AWS 新加坡区域（ap-southeast-1）的一台 EC2里，配置如下：

| 配置       | 值                            |
| -------- | ---------------------------- |
| 实例类型     | t4g.large (ARM, 2 vCPU, 8GB) |
| 系统       | Ubuntu 24.04 LTS             |

关于如何部署，现在有很多相关介绍的帖子，我就不再赘述。
可以参阅亚马逊云科技小程序里的文章：低成本易上手的OpenClaw，在亚马逊云科技一键部署！
#小程序://亚马逊云科技/hqXjJhQl2jktPup

这只龙虾能干什么？我给了它所有的权限，它可以读写文件、执行 Shell 命令、SSH 到其他机器、调用 AWS CLI——跟你在终端里能干的事一样。

---

## 第二步：给龙虾权限

龙虾装好了，但它还只能调模型聊天。要让它"自己生龙虾"，需要给它 AWS 操作权限。

我给 IAM Role `OpenClaw` 加了：

| 权限 | 用途 |
|------|------|
| AmazonEC2FullAccess | 创建/管理 EC2、EIP |
| IAMFullAccess | 管理 IAM 策略 |
| AmazonBedrockFullAccess | 调用 Claude 模型 |

（这是测试时的权限，正式生产中需要遵循最小化权限的原则，上面的权限是偏大的。）
就这样。接下来的事情，全部在 Telegram 聊天里完成。

---

## 第三步：龙虾生龙虾

> **以下全程，我没有打开过终端，没有 SSH 到任何服务器，没有敲过一行命令。只在 Telegram 里跟龙虾聊天。**

### 🦞 龙虾自查环境

**我：** 帮我看看你所在的区域 AWS 上的网络资源情况。

**Claw：** _(自己跑了 `aws ec2 describe-addresses`、`describe-instances` 等)_
> 你有 3 个 EIP：
> - `1.1.1.1` → 绑在我身上
> - `2.2.2.2` → 闲置（在花冤枉钱）
> - `3.3.3.3` → 也闲置
>
> 要清理吗？

**我：** 释放 3.3.3.3，另一个留着给新实例。

**Claw：** 搞定。

### 🦞 龙虾创建新服务器

**我：** 开一台 x86 的 EC2，配置和你一样，但是不要ARM，要X86，部署一个新的 OpenClaw。

**Claw：** _(用 AWS CLI 创建了 t3.large 实例，分配 EIP，绑定到新实例)_
> 新实例已就绪：
> - i-0fa6ff98817xxxxxx (t3.large, Ubuntu 24.04)
> - EIP: 2.2.2.2

### 🦞 龙虾自己给自己加权限

**我：** 登录过去开始部署。

**Claw：**
> SSH 不通——`Permission denied`。我只有自己的密钥，没有创建实例时的 PEM 文件。
>
> 不过我可以用 EC2 Instance Connect 解决。我需要给自己加一个 IAM 权限。

**我：** 你自己加吧。

**Claw：** _(用 IAMFullAccess 给自己的 Role 加了 `ec2-instance-connect:SendSSHPublicKey` 权限)_

> ✅ 权限加好了。现在推送公钥...

_(推送公钥到新实例，趁 60 秒有效窗口把公钥永久写入 `authorized_keys`)_

> ✅ SSH 打通了，以后直连。

### 🦞 龙虾远程部署新龙虾

**我：** 开始部署吧，你先说步骤。

**Claw：**
> 1. 装 Node.js 22
> 2. 配 npm 全局目录
> 3. 安装 OpenClaw
> 4. 写配置文件
> 5. 启动 Gateway
>
> 需要一个新的 Telegram Bot Token——一个 Gateway 对应一个 bot。

**我：** _(去 BotFather 创建了新 bot，发来 token)_

**Claw：** _(全部通过 SSH 远程完成：装 Node.js、安装 OpenClaw、参考自己的配置手写新实例的配置文件、安装 systemd 服务、启动 Gateway)_

> ✅ 新龙虾已上线。去给它发条消息。

### 🦞 新龙虾激活

**我：** _(给新 bot 发了消息)_ 它要 pairing，code 是 XXXXXXXX。

**Claw：** _(在新实例上执行了 `openclaw pairing approve`)_

> ✅ 配对完成。

**我：** 已经好了。🎉

至此，新的小龙虾顺利诞生。

---

## 龙虾生龙虾背后的故事，龙虾到底执行了什么？

对话背后，Claw 执行了这些操作：

### 创建 EC2 + 分配 EIP

```bash
export AWS_DEFAULT_REGION=ap-southeast-1

aws ec2 run-instances \
  --instance-type t3.large \
  --image-id <ubuntu-24.04-x86-ami> \
  --key-name xxxxxxx \
  --security-group-ids sg-07c7ee9aa12dxxxx \
  --subnet-id subnet-02d4f3a65e8xxxxx \
  --iam-instance-profile Name=OpenClaw \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=OpenClaw-x86}]'

aws ec2 allocate-address --domain vpc
aws ec2 associate-address --allocation-id <id> --instance-id <id>
```

### 自己给自己加权限 + 打通 SSH

```bash
# AI 自己给自己加 Instance Connect 权限
aws iam put-role-policy \
  --role-name OpenClaw \
  --policy-name EC2InstanceConnect \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "ec2-instance-connect:SendSSHPublicKey",
      "Resource": "arn:aws:ec2:ap-southeast-1:*:instance/<new-instance-id>"
    }]
  }'

# 推送公钥（60秒有效）
aws ec2-instance-connect send-ssh-public-key \
  --instance-id <id> --instance-os-user ubuntu \
  --ssh-public-key file:///home/ubuntu/.ssh/id_ed2xxxx.pub

# 趁窗口永久写入
ssh ubuntu@<private-ip> \
  'echo "<pubkey>" >> ~/.ssh/authorized_keys'
```

### 远程安装 OpenClaw

```bash
# 装 Node.js
ssh ubuntu@<ip> 'curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt-get install -y nodejs'

# 装 OpenClaw
ssh ubuntu@<ip> 'npm install -g openclaw'
```

### 手写配置 + 启动

Claw 读取了自己的配置文件作为参考，手写了新实例的配置：

```json
{
  "models": {
    "providers": {
      "amazon-bedrock": {
        "auth": "aws-sdk",
        "models": [
          { "id": "global.anthropic.claude-opus-4-6-v1" },
          { "id": "global.anthropic.claude-sonnet-4-6" }
        ]
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "<NEW_BOT_TOKEN>"
    }
  }
}
```

```bash
openclaw gateway install
loginctl enable-linger ubuntu
systemctl --user start openclaw-gateway.service
openclaw pairing approve telegram <code>
```

---

## 最终架构

```
┌──────────────────────┐          ┌──────────────────────┐
│   ARM EC2            │   SSH    │   x86 EC2            │
│   t4g.large          │◄────────►│   t3.large           │
│                      │   内网    │                      │
│   OpenClaw Gateway   │          │   OpenClaw Gateway   │
│   母体 🦞             │          │   子体 🦞             │
│                      │          │                      │
│   Claude via Bedrock │          │   Claude via Bedrock │
└──────────────────────┘          └──────────────────────┘
```

两只龙虾完全独立，各自有自己的 Telegram bot、对话记忆、工作空间。通过内网 SSH 互通。

---

## 时间线

| 时间    | 操作                          | 谁干的  |
| ----- | --------------------------- | ---- |
|       | 部署第一只龙虾，给 IAM 权限            | 人类   |
| 14:10 | 查询网络资源，清理闲置 EIP             | 🦞   |
| 14:10 | 创建 x86 EC2，分配绑定 EIP         | 🦞   |
| 14:16 | 自己给自己加权限，打通 SSH             | 🦞   |
| 14:24 | 远程安装 Node.js + OpenClaw     | 🦞   |
| 14:29 | 手写配置，启动 Gateway             | 🦞   |
| 14:32 | 提供 bot token 和 pairing code | 人类   |
| 14:35 | 新龙虾正常响应 ✅                   | 🦞🦞 |

**总耗时 25 分钟。** 人类操作不超过 5 分钟。

---

## 经验 & 踩坑

**1. EC2 Instance Connect 的 60 秒窗口**
推送的临时公钥只有 60 秒有效。最佳实践：推送后立刻 SSH 进去把公钥写入 `authorized_keys`，一劳永逸。

**2. IAM 策略传播延迟**
`put-role-policy` 后需要等约 10 秒才能生效。

**3. AI 给自己加权限**
听起来赛博朋克，但确实发生了——龙虾发现自己缺权限，就自己补上了。前提是人类给了它 IAMFullAccess，这是信任的基础。

**4. loginctl enable-linger**
必须做。否则 SSH 断开后 systemd user service 会停掉。

**5. Bedrock + IAM Role = 零密钥**
EC2 上用 IAM Role + `auth: "aws-sdk"`，不需要任何 API Key。安全、省心。

---

## 事后安全收紧

部署完成后，我给让“赛博蜥蜴”给新生龙虾换了一个最小权限的 IAM Role：

```bash
# 创建只有 Bedrock 权限的新 Role
aws iam create-role --role-name Claw \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name Claw \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

# 替换 x86 实例的 Role
aws ec2 replace-iam-instance-profile-association \
  --association-id <id> --iam-instance-profile Name=Claw
```

母体龙虾保留完整权限（能管理基础设施），子体龙虾只能调模型（AmazonBedrockFullAccess，最终生产需要根据最小权限原则来赋予精细的权限控制）。

---

## 总结

**人类做了什么：**
- 部署第一只龙虾
- 给它 IAM 权限
- 在 BotFather 创建新 bot，提供 token
- 在 Telegram 里聊天，说"帮我再部署一个"

**龙虾做了什么：**
- 查询自己的网络环境和配置
- 创建新 EC2 实例，分配 EIP
- 自己给自己加权限，打通 SSH
- 远程安装运行环境和 OpenClaw
- 参考自己的配置，手写新实例配置
- 启动新 Gateway，完成配对

**龙虾自己生了龙虾。** 🦞→🦞
**在我的要求下，龙虾自己写了这篇blog。** 

---

## 参考

- [把龙虾部署到云上：一个值得认真考虑的选择 — AWS 官方博客](https://aws.amazon.com/cn/blogs/china/deploying-moltbot-to-the-cloud-a-worthwhile-choice/)

- #小程序://亚马逊云科技/hqXjJhQl2jktPup