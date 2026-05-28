# Tencent Cloud Security Group Example

本示例记录腾讯云 CVM 安全组中常见端口。

## 端口说明

```text
6099：NapCat WebUI
6186：AstrBot WebUI，示例中宿主机 6186 映射容器 6185
6198：AstrBot OneBot 端口，示例中宿主机 6198 映射容器 6199
5000：部分插件 WebUI，例如 meme_manager，可选
```

## 推荐开放策略

后台端口只建议对自己的公网 IP 开放：

```text
来源：你的公网IP/32
```

临时测试可以使用：

```text
来源：0.0.0.0/0
```

但不要长期这样暴露管理后台。

## 最小入站规则

如果只是从自己电脑访问 WebUI：

```text
TCP 6099    来源：你的公网IP/32
TCP 6186    来源：你的公网IP/32
```

如果需要访问插件 WebUI：

```text
TCP 5000    来源：你的公网IP/32
```

通常不需要把 `6198` 对公网开放，因为 NapCat 和 AstrBot 推荐通过 Docker 内网互连：

```text
ws://astrbot_clean:6199/ws
```

## 常见判断

服务器内能访问，但自己电脑浏览器打不开：

```text
大概率是安全组或系统防火墙问题
```

服务器内也不能访问：

```text
大概率是容器、端口监听或应用启动问题
```

服务器本地测试：

```bash
curl -v http://127.0.0.1:6186
```

容器内部测试：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

