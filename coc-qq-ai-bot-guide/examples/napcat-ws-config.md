# NapCat WebSocket Client Config

本示例用于让 NapCat 主动连接 AstrBot 的 OneBot v11 反向 WebSocket。

## 前提

两个容器需要在同一个 Docker 网络中：

```bash
sudo docker network create botnet
sudo docker network connect botnet napcat
sudo docker network connect botnet astrbot_clean
```

如果提示网络已存在或容器已连接，可以忽略。

## AstrBot 侧配置

在 AstrBot WebUI 中创建 OneBot v11 / aiocqhttp 平台：

```text
ID：QQ
启用：true
反向 WebSocket Host：0.0.0.0
反向 WebSocket Port：6199
Token：留空
```

## NapCat 侧配置

进入 NapCat WebUI：

```text
http://服务器公网IP:6099/webui
```

添加：

```text
网络配置 -> WebSocket 客户端
```

填写：

```text
名称：astrbot
启用：true
URL：ws://astrbot_clean:6199/ws
Token：留空
```

如果你的 AstrBot 容器名是 `astrbot`，则 URL 改成：

```text
ws://astrbot:6199/ws
```

## 常见错误

不要填写：

```text
ws://127.0.0.1:6199/ws
```

因为在 NapCat 容器内，`127.0.0.1` 指的是 NapCat 自己，不是 AstrBot。



