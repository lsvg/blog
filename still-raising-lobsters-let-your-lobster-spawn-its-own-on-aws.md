# Still Raising Lobsters? Let Your Lobster Spawn Its Own on AWS

> Deploy OpenClaw on AWS, give it permissions, and let it read its own configuration, create a new server, and deploy another lobster through chat — zero manual commands.

---

## What Is This Lobster?

[OpenClaw](https://github.com/openclaw/openclaw) is an open-source AI assistant framework. Install it on a server, connect it to Telegram/WhatsApp/Discord, and it can execute commands, read/write files, and search the web — essentially an AI with hands and feet.

The community calls it "Lobster" (Claw) 🦞.

My lobster goes by "Cyber Lizard" 🐾.

---

## Step 1: Raise a Lobster on AWS

My lobster lives on an EC2 instance in the AWS Singapore region (ap-southeast-1):

| Config | Value |
| ------------- | ----------------------------- |
| Instance Type | t4g.large (ARM, 2 vCPU, 8GB) |
| OS | Ubuntu 24.04 LTS |

There are plenty of guides on how to deploy OpenClaw, so I won't go into detail here.

This lobster has full permissions — it can read/write files, execute shell commands, SSH into other machines, and use the AWS CLI — just like what you'd do in a terminal.

---

## Step 2: Give the Lobster Permissions

The lobster is up, but it can only chat using AI models. To let it "spawn its own lobster," it needs AWS operational permissions.

I added the following to the IAM Role `OpenClaw`:

| Permission | Purpose |
|------|------|
| AmazonEC2FullAccess | Create/manage EC2 instances and EIPs |
| IAMFullAccess | Manage IAM policies |
| AmazonBedrockFullAccess | Invoke Claude models |

(These are test-phase permissions. In production, follow the principle of least privilege — the above is overly permissive.)

That's it. Everything from here was done entirely through Telegram chat.

---

## Step 3: Lobster Spawns Lobster

> **Throughout the following, I never opened a terminal, never SSH'd into any server, and never typed a single command. I only chatted with the lobster on Telegram.**

### 🦞 Lobster Self-Inspects Its Environment

**Me:** Check the network resources in your AWS region.

**Claw:** _(ran `aws ec2 describe-addresses`, `describe-instances`, etc.)_
> You have 3 EIPs:
> - `1.1.1.1` → attached to me
> - `2.2.2.2` → idle (wasting money)
> - `3.3.3.3` → also idle
>
> Want me to clean up?

**Me:** Release 3.3.3.3, keep the other one for the new instance.

**Claw:** Done.

### 🦞 Lobster Creates a New Server

**Me:** Spin up an x86 EC2 with the same config as yours, but x86 not ARM. Deploy a new OpenClaw on it.

**Claw:** _(used AWS CLI to create a t3.large instance, allocated an EIP, and bound it)_
> New instance ready:
> - i-0fa6ff98817xxxxxx (t3.large, Ubuntu 24.04)
> - EIP: 2.2.2.2

### 🦞 Lobster Grants Itself Permissions

**Me:** Log in and start deploying.

**Claw:**
> SSH failed — `Permission denied`. I only have my own key, not the PEM file used when the instance was created.
>
> But I can fix this with EC2 Instance Connect. I need to add an IAM permission to myself.

**Me:** Go ahead.

**Claw:** _(used IAMFullAccess to add `ec2-instance-connect:SendSSHPublicKey` to its own Role)_

> ✅ Permission added. Pushing public key now...

_(Pushed the public key to the new instance, and within the 60-second validity window, permanently wrote it to `authorized_keys`)_

> ✅ SSH is up. Direct connection from now on.

### 🦞 Lobster Remotely Deploys a New Lobster

**Me:** Start deploying. Walk me through the steps first.

**Claw:**
> 1. Install Node.js 22
> 2. Configure npm global directory
> 3. Install OpenClaw
> 4. Write config file
> 5. Start Gateway
>
> I'll need a new Telegram Bot Token — one Gateway per bot.

**Me:** _(created a new bot via BotFather and sent the token)_

**Claw:** _(all done remotely via SSH: installed Node.js, installed OpenClaw, referenced its own config to write the new instance's config file, installed the systemd service, and started the Gateway)_

> ✅ New lobster is online. Go send it a message.

### 🦞 New Lobster Activated

**Me:** _(sent a message to the new bot)_ It wants pairing, code is XXXXXXXX.

**Claw:** _(ran `openclaw pairing approve` on the new instance)_

> ✅ Pairing complete.

**Me:** It's working. 🎉

And just like that, a new baby lobster was born.

---

## Behind the Scenes: What Did the Lobster Actually Execute?

Behind the chat, Claw executed these operations:

### Create EC2 + Allocate EIP

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

### Self-Granting Permissions + Establishing SSH

```bash
# AI grants itself Instance Connect permission
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

# Push public key (valid for 60 seconds)
aws ec2-instance-connect send-ssh-public-key \
  --instance-id <id> --instance-os-user ubuntu \
  --ssh-public-key file:///home/ubuntu/.ssh/id_ed2xxxx.pub

# Write to authorized_keys within the window
ssh ubuntu@<private-ip> \
  'echo "<pubkey>" >> ~/.ssh/authorized_keys'
```

### Remote OpenClaw Installation

```bash
# Install Node.js
ssh ubuntu@<ip> 'curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt-get install -y nodejs'

# Install OpenClaw
ssh ubuntu@<ip> 'npm install -g openclaw'
```

### Hand-Written Config + Startup

Claw read its own config file as reference and hand-wrote the new instance's configuration:

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

## Final Architecture

```
┌──────────────────────┐          ┌──────────────────────┐
│   ARM EC2            │   SSH    │   x86 EC2            │
│   t4g.large          │◄────────►│   t3.large           │
│                      │  Private │                      │
│   OpenClaw Gateway   │  Network │   OpenClaw Gateway   │
│   Parent 🦞          │          │   Child 🦞            │
│                      │          │                      │
│   Claude via Bedrock │          │   Claude via Bedrock │
└──────────────────────┘          └──────────────────────┘
```

Two completely independent lobsters, each with their own Telegram bot, conversation memory, and workspace. Connected via private network SSH.

---

## Timeline

| Time  | Action | Who |
| ----- | ------ | --- |
|       | Deploy first lobster, grant IAM permissions | Human |
| 14:10 | Query network resources, clean up idle EIPs | 🦞 |
| 14:10 | Create x86 EC2, allocate and bind EIP | 🦞 |
| 14:16 | Self-grant permissions, establish SSH | 🦞 |
| 14:24 | Remotely install Node.js + OpenClaw | 🦞 |
| 14:29 | Hand-write config, start Gateway | 🦞 |
| 14:32 | Provide bot token and pairing code | Human |
| 14:35 | New lobster responds ✅ | 🦞🦞 |

**Total time: 25 minutes.** Human effort: less than 5 minutes.

---

## Lessons & Gotchas

**1. EC2 Instance Connect's 60-Second Window**
The temporary public key is only valid for 60 seconds. Best practice: SSH in immediately after pushing and write the key to `authorized_keys` permanently.

**2. IAM Policy Propagation Delay**
After `put-role-policy`, it takes about 10 seconds to take effect.

**3. AI Granting Itself Permissions**
Sounds cyberpunk, but it happened — the lobster discovered it lacked a permission and added it itself. The prerequisite is that the human gave it IAMFullAccess. That's the foundation of trust.

**4. loginctl enable-linger**
Must do. Otherwise, the systemd user service will stop when the SSH session disconnects.

**5. Bedrock + IAM Role = Zero Key Management**
Using IAM Role + `auth: "aws-sdk"` on EC2, no API keys needed. Secure and hassle-free.

---

## Post-Deployment Security Hardening

After deployment, I had "Cyber Lizard" swap the newborn lobster's IAM Role to a least-privilege one:

```bash
# Create a new Role with only Bedrock access
aws iam create-role --role-name Claw \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name Claw \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

# Replace the x86 instance's Role
aws ec2 replace-iam-instance-profile-association \
  --association-id <id> --iam-instance-profile Name=Claw
```

The parent lobster retains full permissions (infrastructure management), while the child lobster can only invoke models (AmazonBedrockFullAccess — in production, apply fine-grained permissions following least privilege).

---

## Summary

**What the human did:**
- Deployed the first lobster
- Granted it IAM permissions
- Created a new bot on BotFather, provided the token
- Chatted on Telegram: "Deploy another one for me"

**What the lobster did:**
- Queried its own network environment and configuration
- Created a new EC2 instance, allocated an EIP
- Granted itself permissions, established SSH
- Remotely installed the runtime and OpenClaw
- Referenced its own config to write the new instance's config
- Started the new Gateway, completed pairing

**The lobster spawned its own lobster.** 🦞→🦞
**At my request, the lobster wrote this blog post itself.**

---

## References

- [Deploying Lobsters to the Cloud: A Choice Worth Serious Consideration — AWS Official Blog](https://aws.amazon.com/cn/blogs/china/deploying-moltbot-to-the-cloud-a-worthwhile-choice/)
