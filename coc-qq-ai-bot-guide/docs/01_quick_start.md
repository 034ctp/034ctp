
# 01 Quickstart：腾讯云部署 QQ 群跑团 AI 机器人

这是一份实战向快速启动记录，因为本人使用了腾讯云的轻量应用服务器进行测试，所以本指南的目标是在腾讯云服务器上跑通（但经验应当可以共通）：

```text
NapCatQQ + AstrBot + DeepSeek + OneBot v11
```

适用场景：

```text
QQ群跑团
克苏鲁的呼唤 COC
DND / TRPG 辅助
AI 守秘人 / 跑团主持人
```

本篇不替代官方文档，只记录最小可用部署路径和容易踩坑的地方。

---

## 1. 整体架构

```text
QQ群
  ↓
机器人 QQ 号
  ↓
NapCatQQ
  ↓ OneBot v11 WebSocket
AstrBot
  ↓
DeepSeek
```

其中：

```text
NapCatQQ：负责登录 QQ，接收和发送 QQ 群消息
AstrBot：负责机器人逻辑、插件、人格设定、模型调用
DeepSeek：负责 AI 对话和跑团叙事
OneBot v11：NapCat 和 AstrBot 之间的通信协议
```

---

## 2. 推荐端口

本文示例使用：

```text
6099：NapCat WebUI
3001：NapCat OneBot 服务端口，可选
6186：宿主机访问 AstrBot WebUI
6198：宿主机映射 AstrBot OneBot 端口
5000：部分插件 WebUI，例如 meme_manager，可选
```

容器内部实际端口：

```text
AstrBot WebUI：6185
AstrBot OneBot：6199
```

如果使用本文的 `astrbot_clean` 示例：

```text
宿主机 6186 -> 容器 6185
宿主机 6198 -> 容器 6199
```

浏览器访问 AstrBot 时使用：

```text
http://服务器公网IP:6186
```

注意是 `http`，不是 `https`。

---

## 3. 部署 NapCat

创建目录：

```bash
mkdir -p ~/napcat
cd ~/napcat
mkdir -p napcat/config ntqq
```

创建 `docker-compose.yml`：

```yaml
services:
  napcat:
    image: mlikiowa/napcat-docker:latest
    container_name: napcat
    restart: always
    environment:
      - NAPCAT_UID=${NAPCAT_UID}
      - NAPCAT_GID=${NAPCAT_GID}
    ports:
      - "6099:6099"
      - "3001:3001"
    volumes:
      - ./napcat/config:/app/napcat/config
      - ./ntqq:/app/.config/QQ
```

启动：

```bash
sudo env NAPCAT_UID=$(id -u) NAPCAT_GID=$(id -g) docker compose up -d
```

查看日志：

```bash
sudo docker logs -f napcat
```

访问 NapCat WebUI：

```text
http://服务器公网IP:6099/webui
```

如果需要 token：

```bash
sudo docker logs napcat 2>&1 | grep -i token
```

---

## 4. 部署 AstrBot

建议先使用干净目录：

```bash
mkdir -p ~/astrbot_clean/data
```

启动 AstrBot：

```bash
sudo docker run -itd \
  -p 6186:6185 \
  -p 6198:6199 \
  -v ~/astrbot_clean/data:/AstrBot/data \
  --name astrbot_clean \
  --restart=always \
  soulter/astrbot:latest
```

查看日志：

```bash
sudo docker logs -f astrbot_clean
```

正常启动后应看到类似：

```text
AstrBot v4.x WebUI is ready
Initial username: astrbot
Initial password: xxxxxxxx
Running on http://0.0.0.0:6185
```

访问 AstrBot：

```text
http://服务器公网IP:6186
```

如果浏览器提示响应超时，优先检查：

```text
1. 是否用了 http 而不是 https
2. 腾讯云安全组是否放行 TCP 6186
3. 容器是否正常运行
```

测试 WebUI 是否在容器内监听：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

输出：

```text
0
```

表示正常。

---

## 5. 连接 NapCat 和 AstrBot

创建 Docker 网络：

```bash
sudo docker network create botnet
```

把两个容器加入网络：

```bash
sudo docker network connect botnet napcat
sudo docker network connect botnet astrbot_clean
```

如果提示已经存在或已经连接，可以忽略。

重启：

```bash
sudo docker restart astrbot_clean
sudo docker restart napcat
```

在 AstrBot WebUI 中创建 OneBot v11 平台：

```text
类型：OneBot v11 / aiocqhttp
ID：QQ
启用：true
反向 WebSocket Host：0.0.0.0
反向 WebSocket Port：6199
Token：留空
```

在 NapCat WebUI 中添加 WebSocket 客户端：

```text
类型：WebSocket 客户端
URL：ws://astrbot_clean:6199/ws
Token：留空
启用：true
```

注意：

```text
ws://astrbot_clean:6199/ws
```

这是填在 NapCat WebUI 里的地址，不是在 Linux 终端执行的命令。

---

## 6. 配置 DeepSeek

在 DeepSeek 平台创建 API Key：

```text
https://platform.deepseek.com/
```

进入 AstrBot WebUI：

```text
服务提供商 / 模型提供商 -> 新增
```

推荐配置：

```text
类型：DeepSeek 或 OpenAI 兼容
Base URL：https://api.deepseek.com/v1
API Key：你的 DeepSeek API Key
模型：deepseek-chat
```

日常跑团建议先用：

```text
deepseek-chat
```

如果需要更强推理，可另建：

```text
deepseek-reasoner
```

然后在 AstrBot 普通配置中，把默认对话模型设为 DeepSeek。

---

## 7. 腾讯云安全组

至少需要临时放行：

```text
TCP 6099：NapCat WebUI
TCP 6186：AstrBot WebUI
```

如果使用插件 WebUI，例如 meme_manager：

```text
TCP 5000
```

更安全的做法：

```text
来源只填写自己的公网 IP/32
```

不要长期向公网开放：

```text
6099
6186
5000
```

---

## 8. 常用维护命令

查看容器：

```bash
sudo docker ps -a
```

重启：

```bash
sudo docker restart astrbot_clean
sudo docker restart napcat
```

查看日志：

```bash
sudo docker logs --tail 100 astrbot_clean
sudo docker logs --tail 100 napcat
```

实时日志：

```bash
sudo docker logs -f astrbot_clean
sudo docker logs -f napcat
```

检查 AstrBot WebUI：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

检查 AstrBot OneBot 端口：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6199)))"
```

检查 Docker 网络：

```bash
sudo docker network inspect botnet
```

---

## 9. 最小测试流程

1. 打开 AstrBot WebUI：

```text
http://服务器公网IP:6186
```

2. 确认 DeepSeek 能在 WebUI 聊天窗口回复。

3. 打开 NapCat WebUI：

```text
http://服务器公网IP:6099/webui
```

4. 确认 QQ 在线。

5. 确认 NapCat WebSocket 客户端连接到：

```text
ws://astrbot_clean:6199/ws
```

6. 在 QQ 群中发送：

```text
@机器人 你好
```

或：

```text
/sid
```

如果能回复，说明基础链路已经跑通。

---

## 10. 新手最容易踩的坑


### 容器名冲突

如果出现：

```text
container name "/napcat" is already in use
```

说明已经有同名容器。先检查：

```bash
sudo docker ps -a
```

### WebSocket 地址填错

NapCat 连接 AstrBot 时，不要填：

```text
ws://127.0.0.1:6199/ws
```

在 NapCat 容器里，`127.0.0.1` 指 NapCat 自己。

应该填：

```text
ws://astrbot_clean:6199/ws
```

### AstrBot WebUI 打不开

先确认是否用了正确协议：

```text
http://服务器公网IP:6186
```

不要写成：

```text
https://服务器公网IP:6186
```

再检查安全组是否放行 `6186`。

### 插件导致 Dashboard 起不来

如果安装插件后 AstrBot WebUI 变成不可访问，先检查：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

如果输出 `111`，说明 WebUI 没有监听。

可以临时移走插件目录：

```bash
sudo docker stop astrbot_clean
mkdir -p ~/astrbot_clean/plugin_backup
sudo mv ~/astrbot_clean/data/plugins ~/astrbot_clean/plugin_backup/plugins_bak
sudo mkdir -p ~/astrbot_clean/data/plugins
sudo docker start astrbot_clean
```

---

## 11. 跑团机器人建议

人格设定放在 AstrBot 的 Persona / System Prompt 中。

规则书不要直接塞进提示词，建议使用知识库：

```text
人格设定：守秘人的风格、边界、裁定原则
知识库：COC 规则、模组、NPC、线索、团记
插件：骰点、角色卡、理智检定
```

推荐先跑通：

```text
QQ 收发消息
DeepSeek 回复
管理员 ID
基础人格
```

然后再逐步添加：

```text
骰子插件
图库插件
知识库
角色卡
团记
```

每添加一个插件，都建议测试一次：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

确保 WebUI 仍然正常。
```