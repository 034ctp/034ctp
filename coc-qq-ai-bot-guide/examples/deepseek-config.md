# DeepSeek Config Example

本示例记录 AstrBot 中接入 DeepSeek 的常用配置。

## 获取 API Key

访问 DeepSeek 平台：

```text
https://platform.deepseek.com/
```

创建 API Key。不要把 API Key 上传到 GitHub。

## AstrBot 服务提供商配置

在 AstrBot WebUI 中进入：

```text
服务提供商 / 模型提供商 -> 新增
```

如果有 DeepSeek 选项，直接选择 DeepSeek。

如果使用 OpenAI 兼容配置，可填写：

```text
名称：deepseek
类型：OpenAI Compatible
Base URL：https://api.deepseek.com/v1
API Key：sk-xxxxxxxxxxxxxxxx
模型：deepseek-chat
```

日常群聊和跑团建议先使用：

```text
deepseek-chat
```

需要更强推理时可另建：

```text
deepseek-reasoner
```

## 默认模型

在 AstrBot WebUI 中进入：

```text
配置 -> 普通配置
```

把默认对话模型设置为刚才创建的 DeepSeek 模型。

## 测试

先在 AstrBot WebUI 的聊天测试区发送：

```text
你好，请用一句话介绍你自己
```

如果 WebUI 内能回复，但 QQ 群不回复，说明 DeepSeek 配置基本正常，问题多半在 NapCat、OneBot 或群聊触发链路。

## 不要提交到 GitHub 的内容

```text
API Key
cmd_config.json
模型服务商完整配置截图
任何包含余额、密钥、账号信息的日志
```

