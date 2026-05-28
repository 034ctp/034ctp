
# 03 Troubleshooting：常见故障排查

本篇记录部署 NapCat + AstrBot + DeepSeek 过程中常见错误。

建议排查顺序：

```text
先看容器是否运行
再看端口是否监听
再看 Docker 网络
再看插件
最后再考虑重建容器
```

---

## 1. Docker 权限不足

### 现象

执行 Docker 命令时报错：

```text
permission denied while trying to connect to the Docker daemon socket
```

### 原因

当前用户没有权限访问 Docker。

### 解决

临时使用 `sudo`：

```bash
sudo docker ps
```

长期解决：

```bash
sudo usermod -aG docker $USER
```

然后退出 SSH，重新登录。

---


## 2. 环境变量未设置

### 现象

启动 NapCat 时出现：

```text
The "NAPCAT_UID" variable is not set
The "NAPCAT_GID" variable is not set
```

### 原因

`docker-compose.yml` 中使用了：

```yaml
- NAPCAT_UID=${NAPCAT_UID}
- NAPCAT_GID=${NAPCAT_GID}
```

但启动时没有传入变量。

### 解决

使用：

```bash
sudo env NAPCAT_UID=$(id -u) NAPCAT_GID=$(id -g) docker compose up -d
```

---

## 3. 容器名称冲突

### 现象

```text
Conflict. The container name "/napcat" is already in use
```

或：

```text
Conflict. The container name "/astrbot_clean" is already in use
```

### 原因

已经存在同名容器。

### 解决

查看：

```bash
sudo docker ps -a
```

如果旧容器还要保留，换一个容器名。

如果确定不要旧容器：

```bash
sudo docker stop 容器名
sudo docker rm 容器名
```

然后重新创建。

---

## 4. WebSocket 地址被当成命令执行

### 现象

输入：

```bash
ws://astrbot_clean:6199/ws
```

终端报错：

```text
No such file or directory
```

### 原因

`ws://astrbot_clean:6199/ws` 是 WebSocket 地址，不是 Linux 命令。

### 解决

把它填写到 NapCat WebUI：

```text
网络配置 -> WebSocket 客户端 -> URL
```

正确填写：

```text
ws://astrbot_clean:6199/ws
```

---

## 5. AstrBot WebUI 无法打开

### 现象

浏览器打不开：

```text
http://服务器公网IP:6186
```

或显示响应时间过长。

### 常见原因

```text
1. 用了 https 而不是 http
2. 腾讯云安全组没放行端口
3. 容器没启动
4. AstrBot Dashboard 没监听
5. 插件导致 AstrBot 没完整启动
```

### 排查

查看容器：

```bash
sudo docker ps -a
```

检查容器内 WebUI 是否监听：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

结果：

```text
0：正常
111：没有监听
```

浏览器必须使用：

```text
http://服务器公网IP:6186
```

不要使用：

```text
https://服务器公网IP:6186
```

---

## 6. curl 返回 Connection reset by peer

### 现象

```bash
curl -v http://127.0.0.1:6185
```

返回：

```text
Recv failure: Connection reset by peer
```

### 原因

连接建立了，但服务端立即断开。常见于：

```text
1. WebUI 服务没有真正启动
2. 容器端口映射存在，但容器内部服务异常
3. 插件或配置导致 AstrBot 启动流程中断
```

### 解决

进入容器内部检查：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

如果是 `111`，优先检查插件和日志：

```bash
sudo docker logs --tail 300 astrbot_clean
```

---

## 7. HTTPS 报 unexpected eof while reading

### 现象

```bash
curl -vk https://127.0.0.1:6185
```

返回：

```text
SSL routines::unexpected eof while reading
```

### 原因

AstrBot WebUI 默认是 HTTP，不是 HTTPS。

### 解决

使用：

```text
http://服务器公网IP:6186
```

不要使用：

```text
https://服务器公网IP:6186
```

---

## 8. connect_ex 返回 111

### 现象

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

输出：

```text
111
```

### 原因

容器内部该端口没有服务监听。

对于：

```text
6185
```

代表 AstrBot WebUI 没启动。

对于：

```text
6199
```

代表 OneBot 平台没启用或没启动。

### 解决

先看日志：

```bash
sudo docker logs --tail 300 astrbot_clean
```

如果最近安装过插件，先临时移走插件目录：

```bash
sudo docker stop astrbot_clean
mkdir -p ~/astrbot_clean/plugin_backup
sudo mv ~/astrbot_clean/data/plugins ~/astrbot_clean/plugin_backup/plugins_bak
sudo mkdir -p ~/astrbot_clean/data/plugins
sudo docker start astrbot_clean
```

再测试：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

---

## 9. 插件目录改名为 .disabled 后仍然报错

### 现象

```text
Loading plugin astrbot_plugin_xxx.disabled
ModuleNotFoundError
```

### 原因

AstrBot 会扫描 `data/plugins/` 下的目录。  
把插件目录改名为 `.disabled` 但仍留在 `plugins` 目录中，可能仍会被扫描。

### 解决

不要把问题插件留在：

```text
~/astrbot_clean/data/plugins/
```

应移动到外部备份目录：

```bash
sudo docker stop astrbot_clean
mkdir -p ~/astrbot_clean/plugin_bad
sudo mv ~/astrbot_clean/data/plugins/插件目录名 ~/astrbot_clean/plugin_bad/
sudo docker start astrbot_clean
```

---

## 10. 插件导致 Dashboard 起不来

### 现象

安装插件后：

```text
AstrBot WebUI 打不开
connect_ex 6185 返回 111
日志没有 WebUI is ready
```

### 原因

插件导入失败、依赖不兼容、启动时阻塞，都可能导致 AstrBot 无法完整启动。

### 解决

先移走所有插件：

```bash
sudo docker stop astrbot_clean
mkdir -p ~/astrbot_clean/plugin_backup
sudo mv ~/astrbot_clean/data/plugins ~/astrbot_clean/plugin_backup/plugins_bak
sudo mkdir -p ~/astrbot_clean/data/plugins
sudo docker start astrbot_clean
```

如果 WebUI 恢复，说明是插件问题。

之后每次只恢复一个插件：

```bash
sudo docker stop astrbot_clean
sudo cp -a ~/astrbot_clean/plugin_backup/plugins_bak/插件目录名 ~/astrbot_clean/data/plugins/
sudo docker start astrbot_clean
```

每恢复一个插件都测试：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

---

## 11. 插件 API 不兼容

### 现象

安装骰子插件时报错：

```text
AttributeError: type object 'filter' has no attribute 'platform_adapter_type'
```

### 原因

插件调用了当前 AstrBot 版本不支持的 API，或插件与 AstrBot 版本不匹配。

### 解决

```text
1. 卸载该插件
2. 移出插件目录
3. 换用其他兼容插件
4. 或等待插件作者更新
```

命令：

```bash
sudo docker stop astrbot_clean
mkdir -p ~/astrbot_clean/plugin_bad
sudo mv ~/astrbot_clean/data/plugins/插件目录名 ~/astrbot_clean/plugin_bad/
sudo docker start astrbot_clean
```

---

## 12. plugins.json 里有很多插件记录

### 现象

搜索插件时看到：

```text
/home/ubuntu/astrbot_clean/data/plugins.json
```

里面有很多插件，包括已经卸载的插件。

### 原因

`plugins.json` 通常是插件市场索引/缓存，不等于已安装插件列表。

真正会被加载的是：

```text
~/astrbot_clean/data/plugins/
```

### 解决

优先检查：

```bash
ls -la ~/astrbot_clean/data/plugins
```

不要仅因为 `plugins.json` 中出现某个插件名就删除整个文件。

---

## 13. Address already in use

### 现象

日志出现：

```text
OSError: [Errno 98] Address already in use
```

### 原因

端口被占用。可能是：

```text
1. 同一个容器内两个服务抢同一端口
2. 插件重复启动服务
3. 旧容器仍在运行
4. 宿主机端口映射冲突
```

### 排查

查看容器：

```bash
sudo docker ps -a
```

查看宿主机端口：

```bash
sudo ss -lntp | grep -E '6185|6186|6199|6198|5000'
```

注意：

```text
docker-proxy 占用宿主机端口通常是正常的端口映射，不一定是错误。
```

### 解决

如果旧容器还在：

```bash
sudo docker stop 旧容器名
sudo docker rm 旧容器名
```

如果插件端口冲突，例如 5000，可换宿主机端口：

```bash
-p 5001:5000
```

---

## 14. NapCat 无法访问 AstrBot

### 现象

QQ群消息无响应，NapCat WebSocket 连接失败。

### 原因

常见是 URL 填错或 Docker 网络不通。

### 检查 Docker 网络

```bash
sudo docker network inspect botnet
```

应看到：

```text
napcat
astrbot_clean
```

如果没有：

```bash
sudo docker network connect botnet napcat
sudo docker network connect botnet astrbot_clean
```

### 正确 URL

NapCat WebUI 中填写：

```text
ws://astrbot_clean:6199/ws
```

不要填：

```text
ws://127.0.0.1:6199/ws
```

---

## 15. QQ 群里机器人不回复

### 原因清单

```text
1. NapCat QQ 掉线
2. NapCat WebSocket 没连上 AstrBot
3. AstrBot OneBot 平台未启用
4. DeepSeek 模型配置错误
5. 群聊需要 @机器人 才触发
6. 机器人被禁言或风控
7. 插件或权限配置拦截
```

### 排查顺序

重启：

```bash
sudo docker restart astrbot_clean
sleep 10
sudo docker restart napcat
```

看日志：

```bash
sudo docker logs --tail 200 astrbot_clean
sudo docker logs --tail 200 napcat
```

测试：

```text
@机器人 你好
/sid
```

---

## 16. 管理员 ID 无效

### 现象

群里执行管理命令时提示：

```text
此命令仅限管理员使用
```

### 原因

管理员 ID 填成了昵称，而不是 QQ 数字 ID。

错误示例：

```text
伪人
大环
astrbot
```

正确示例：

```text
123456789
```

### 解决

在群里发送：

```text
/sid
```

获取 QQ 数字 ID，然后添加到 AstrBot 管理员 ID 列表。

---

## 17. scp 上传图片 No such file or directory

### 现象

```text
No such file or directory
```

### 原因

目标目录不存在。

### 解决

先在服务器创建目录：

```bash
mkdir -p "/home/ubuntu/astrbot_clean/data/plugin_data/astrbot_plugin_gallery/galleries/戳一戳"
```

再从 Windows 上传：

```powershell
scp "C:\Users\你的用户名\Desktop\s.jpg" "ubuntu@服务器公网IP:/home/ubuntu/astrbot_clean/data/plugin_data/astrbot_plugin_gallery/galleries/戳一戳/"
```

---

## 18. scp 上传图片 Permission denied

### 现象

```text
Permission denied
```

### 原因

目标目录存在，但 `ubuntu` 用户没有写权限。

### 解决

服务器执行：

```bash
sudo chown -R ubuntu:ubuntu "/home/ubuntu/astrbot_clean/data/plugin_data/astrbot_plugin_gallery"
chmod -R u+rwX "/home/ubuntu/astrbot_clean/data/plugin_data/astrbot_plugin_gallery"
```

然后重新上传。

---

## 19. NapCat token 找不到

### 现象

NapCat WebUI 要求 token，但不知道 token 在哪里。

### 解决

查看日志：

```bash
sudo docker logs napcat 2>&1 | grep -i token
```

如果没有：

```bash
sudo docker logs --tail 300 napcat
```

或重启后实时看：

```bash
sudo docker restart napcat
sudo docker logs -f napcat
```

---

## 20. AstrBot 初始密码找不到

### 现象

AstrBot WebUI 要求登录，但不知道用户名密码。

### 解决

查看日志：

```bash
sudo docker logs astrbot_clean 2>&1 | grep -i "Initial username\|Initial password\|WebUI is ready"
```

通常用户名是：

```text
astrbot
```

密码以日志输出为准。

---

## 21. 最后手段：干净数据目录测试

如果原数据目录一直导致 WebUI 无法启动，可以创建干净实例判断镜像是否正常。

```bash
mkdir -p ~/astrbot_clean_test/data
sudo docker run -itd \
  -p 6187:6185 \
  -p 6197:6199 \
  -v ~/astrbot_clean_test/data:/AstrBot/data \
  --name astrbot_clean_test \
  --restart=always \
  soulter/astrbot:latest
```

等待 30 秒：

```bash
sudo docker logs --tail 150 astrbot_clean_test
```

测试：

```bash
sudo docker exec -it astrbot_clean_test python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

如果输出 `0`，说明镜像没问题，问题在旧数据目录、插件或配置中。

---

## 22. 故障排查原则

```text
1. 不要一出问题就重装服务器
2. 不要一次安装多个插件
3. 每装一个插件都检查 6185 是否监听
4. 插件出问题时，移出 data/plugins，而不是只改名
5. plugins.json 不是已安装插件目录
6. 能用干净 data 跑通，就说明镜像本身没问题
7. 先恢复 WebUI，再恢复插件
8. 先跑通 QQ + DeepSeek，再做知识库和复杂插件
```
