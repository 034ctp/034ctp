下面这份可以直接放进：

```text
docs/02-config-checklist.md
```

```md
# 02 Config Checklist：配置检查清单

这份清单用于排查：

```text
NapCat 已启动，但 QQ 没反应
AstrBot WebUI 能打开，但机器人不回复
DeepSeek 配好了，但群聊不触发
NapCat 和 AstrBot 看似都正常，但消息链路不通
```

建议按顺序检查，不要一上来重装。

---

## 1. 容器是否运行

查看所有容器：

```bash
sudo docker ps -a
```

至少应看到：

```text
napcat          Up ...
astrbot_clean   Up ...
```

如果没有运行：

```bash
sudo docker start napcat
sudo docker start astrbot_clean
```

或直接重启：

```bash
sudo docker restart astrbot_clean
sudo docker restart napcat
```

推荐重启顺序：

```text
先 AstrBot，后 NapCat
```

因为 NapCat 需要连接 AstrBot 的 WebSocket 服务。

---

## 2. 端口映射是否正确

查看端口：

```bash
sudo docker ps
```

推荐映射：

```text
NapCat:
0.0.0.0:6099->6099/tcp

AstrBot:
0.0.0.0:6186->6185/tcp
0.0.0.0:6198->6199/tcp
```

示例：

```text
6099：NapCat WebUI
6186：AstrBot WebUI
6198：AstrBot OneBot 反向 WebSocket，对外映射
```

注意：

```text
容器内部 AstrBot WebUI 是 6185
容器内部 OneBot 端口是 6199
浏览器访问用宿主机端口 6186
NapCat 容器连接用容器内部端口 6199
```

---

## 3. 腾讯云安全组

从自己电脑浏览器访问服务器时，需要腾讯云安全组放行。

至少放行：

```text
TCP 6099：NapCat WebUI
TCP 6186：AstrBot WebUI
```

如果使用插件 WebUI：

```text
TCP 5000 或你实际映射的插件端口
```

建议来源填写：

```text
你的电脑公网 IP/32
```

临时测试可用：

```text
0.0.0.0/0
```

但不建议长期暴露后台端口。

---

## 4. AstrBot WebUI 是否正常

在服务器上执行：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

结果含义：

```text
0：容器内部 WebUI 正常监听
111：WebUI 没有监听，通常是 AstrBot 没完整启动或插件导致
```

如果是 `0`，浏览器访问：

```text
http://服务器公网IP:6186
```

注意必须是：

```text
http
```

不是：

```text
https
```

---

## 5. AstrBot OneBot 端口是否正常

在服务器上执行：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6199)))"
```

结果含义：

```text
0：OneBot 反向 WebSocket 正常监听
111：OneBot 平台未启用或 AstrBot 没完整启动
```

如果是 `111`，进入 AstrBot WebUI 检查平台配置。

---

## 6. AstrBot 平台配置

进入 AstrBot WebUI：

```text
http://服务器公网IP:6186
```

检查：

```text
消息平台 / 机器人 / OneBot v11 / aiocqhttp
```

推荐配置：

```text
ID：QQ
启用：true
反向 WebSocket Host：0.0.0.0
反向 WebSocket Port：6199
Token：留空
```

如果设置了 Token，NapCat 里也必须填写同一个 Token。

---

## 7. Docker 网络是否连通

NapCat 和 AstrBot 应在同一个 Docker 网络中。

查看：

```bash
sudo docker network inspect botnet
```

应看到：

```text
napcat
astrbot_clean
```

如果缺少某个容器：

```bash
sudo docker network connect botnet napcat
sudo docker network connect botnet astrbot_clean
```

如果提示已经连接，可以忽略。

重启：

```bash
sudo docker restart astrbot_clean
sudo docker restart napcat
```

---

## 8. NapCat WebSocket 客户端配置

进入 NapCat WebUI：

```text
http://服务器公网IP:6099/webui
```

检查：

```text
网络配置 -> WebSocket 客户端
```

推荐配置：

```text
启用：true
URL：ws://astrbot_clean:6199/ws
Token：留空
```

不要填：

```text
ws://127.0.0.1:6199/ws
```

因为在 NapCat 容器里，`127.0.0.1` 指的是 NapCat 容器自己，不是 AstrBot。

也不要填旧容器名：

```text
ws://astrbot:6199/ws
```

如果当前 AstrBot 容器名是：

```text
astrbot_clean
```

就必须使用：

```text
ws://astrbot_clean:6199/ws
```

---

## 9. NapCat QQ 是否在线

进入 NapCat WebUI，检查：

```text
QQ 是否仍在线
是否需要重新扫码
是否被风控
```

如果 WebUI 需要 token：

```bash
sudo docker logs napcat 2>&1 | grep -i token
```

如果找不到：

```bash
sudo docker logs --tail 300 napcat
```

或重启后实时看：

```bash
sudo docker restart napcat
sudo docker logs -f napcat
```

---

## 10. DeepSeek 是否可用

在 AstrBot WebUI 中先测试模型。

检查：

```text
服务提供商 / 模型提供商
```

推荐配置：

```text
Base URL：https://api.deepseek.com/v1
模型：deepseek-chat
API Key：有效
```

然后在 AstrBot WebUI 的聊天测试区发送：

```text
你好，请用一句话介绍你自己
```

如果 WebUI 内能回复，但 QQ 群不回复，说明问题在 NapCat/OneBot/权限链路。

如果 WebUI 内也不能回复，说明问题在模型配置。

---

## 11. 管理员 ID

管理员 ID 必须是 QQ 数字 ID，不是昵称。

在 QQ 群中发送：

```text
/sid
```

找到自己的 QQ 数字 ID 后，在 AstrBot WebUI 添加：

```text
配置 -> 其他配置 -> 管理员 ID
```

错误示例：

```text
QQ群昵称
QQ名
astrbot
```

正确示例：

```text
123456789
```

---

## 12. 群聊触发方式

如果机器人不回复，先测试：

```text
@机器人 你好
```

再测试：

```text
/sid
```

如果只有 `/sid` 有反应，说明平台连接正常，但对话触发规则、人格或默认模型配置可能有问题。

如果两者都没反应，优先检查：

```text
NapCat 是否在线
WebSocket 是否连接
OneBot 平台是否启用
容器网络是否正确
```

---

## 13. 插件状态

安装插件后，如果机器人突然不回复或 WebUI 打不开，先看 AstrBot 是否还完整启动：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

如果输出：

```text
111
```

可能是插件导致 Dashboard 没启动。

临时移走插件目录：

```bash
sudo docker stop astrbot_clean
mkdir -p ~/astrbot_clean/plugin_backup
sudo mv ~/astrbot_clean/data/plugins ~/astrbot_clean/plugin_backup/plugins_bak
sudo mkdir -p ~/astrbot_clean/data/plugins
sudo docker start astrbot_clean
```

然后再测：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

如果恢复为 `0`，说明问题来自某个插件。

---

## 14. 日志检查命令

AstrBot 最近日志：

```bash
sudo docker logs --tail 200 astrbot_clean
```

NapCat 最近日志：

```bash
sudo docker logs --tail 200 napcat
```

筛选 AstrBot 错误：

```bash
sudo docker logs --tail 300 astrbot_clean | grep -i "error\|traceback\|exception\|aiocqhttp\|onebot\|websocket"
```

筛选 NapCat 连接日志：

```bash
sudo docker logs --tail 300 napcat | grep -i "websocket\|ws\|onebot\|connect\|error"
```

---

## 15. 最小可用状态

一个正常工作的最小状态应满足：

```text
1. docker ps 显示 napcat 和 astrbot_clean 都是 Up
2. AstrBot WebUI 可以通过 http://服务器公网IP:6186 打开
3. NapCat WebUI 可以通过 http://服务器公网IP:6099/webui 打开
4. AstrBot 容器内 6185 返回 0
5. AstrBot 容器内 6199 返回 0
6. NapCat WebSocket 客户端连接 ws://astrbot_clean:6199/ws
7. QQ 号在 NapCat 中在线
8. DeepSeek 在 AstrBot WebUI 中可回复
9. 群里 @机器人 有回应
```

满足以上条件后，再安装插件、配置知识库或导入跑团资料。
```