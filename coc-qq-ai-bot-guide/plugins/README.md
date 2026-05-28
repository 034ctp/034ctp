# Plugins

这里用于存放自定义 AstrBot 插件，或记录插件开发计划。

建议不要直接提交从插件市场下载的第三方插件源码，除非你确认许可证允许再分发。更好的做法是：

```text
1. 在 README 中记录插件名称、仓库地址和版本
2. 记录安装后的配置方法
3. 记录兼容性和已知问题
4. 自己编写的插件单独放一个目录或单独建仓库
```

## 推荐的插件仓库结构

```text
astrbot_plugin_coc_basic/
  main.py
  metadata.yaml
  requirements.txt
  _conf_schema.json
  README.md
  LICENSE
```

## metadata.yaml 示例

```yaml
name: astrbot_plugin_coc_basic
display_name: COC Basic Tools
version: 0.1.0
author: your-name
desc: 克苏鲁的呼唤基础跑团工具，提供骰点、技能检定和理智检定命令。
repo: https://github.com/your-name/astrbot_plugin_coc_basic
```

## requirements.txt

如果插件需要第三方依赖，写入：

```text
package-name>=1.0.0
```

如果只使用 Python 标准库和 AstrBot API，可以留空或不创建。

## 配置文件

如果插件需要可视化配置，可以添加：

```text
_conf_schema.json
```

例如：

```json
{
  "command_prefix": {
    "description": "命令前缀",
    "type": "string",
    "default": "."
  }
}
```

## 插件开发建议

- 先写小插件，避免一开始做大而全的骰娘。
- 每次新增功能后重启 AstrBot 并确认 WebUI 仍可访问。
- 数据持久化应放在 AstrBot 的 data/plugin_data 下，不要写死绝对路径。
- 网络请求使用异步库，例如 aiohttp 或 httpx。
- 不要在插件中硬编码 API Key、QQ 号、服务器 IP。
- 不要使用危险的 `eval`、`exec`、`subprocess`，除非有充分理由且输入完全不可由用户控制。

## 已知插件踩坑记录

安装插件后如果 WebUI 无法访问，先测试：

```bash
sudo docker exec -it astrbot_clean python -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1', 6185)))"
```

如果输出：

```text
111
```

说明 AstrBot WebUI 没有监听，可能是插件阻塞或崩溃。

临时移走插件目录：

```bash
sudo docker stop astrbot_clean
mkdir -p ~/astrbot_clean/plugin_backup
sudo mv ~/astrbot_clean/data/plugins ~/astrbot_clean/plugin_backup/plugins_bak
sudo mkdir -p ~/astrbot_clean/data/plugins
sudo docker start astrbot_clean
```

不要只把插件目录改名为 `.disabled` 后继续留在 `data/plugins/` 内，AstrBot 仍可能扫描它。

