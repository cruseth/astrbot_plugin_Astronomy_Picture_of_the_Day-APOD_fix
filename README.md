# astrbot_plugin_Astronomy_Picture_of_the_Day

AstrBot 的 NASA Astronomy Picture of the Day 插件。

插件会通过 NASA APOD API 获取当天的每日天文图片，并按配置返回图片、标题、日期和说明文本；也支持调用 AstrBot 已配置的 LLM 提供商将标题和说明翻译为简体中文。

## 功能简介

- 通过 `apod` 指令获取 NASA 每日天文图片
- 支持返回图片、标题、日期、说明
- 支持将标题和说明翻译为简体中文
- 支持图片和文本分开发送，标题、日期、说明会合并为一个文本段
- 支持按 UMO 列表定向自动推送到指定群聊/会话
- 自动推送会遵循 `is_divided` 配置，和 `/apod` 命令保持相同发送逻辑
- 主动推送使用 AstrBot `MessageChain`，兼容 AstrBot v4.25+
- 支持为 NASA APOD API 请求配置 HTTP 代理
- 内置 APOD 数据缓存，减少重复请求 NASA API
- 内置翻译结果缓存，避免重复翻译相同文本
- 内置按目标会话记录的推送状态与推送内容缓存，实现“拉取一次，发送多次”
- 支持超时和失败重试配置

## 适用版本

- AstrBot `>= 4.16`

## 支持平台

- `aiocqhttp`

## 指令

### 获取今日 APOD

```text
/apod
```

### 获取当前会话 UMO（用于自动推送配置）

```text
/sid
```

### 手动执行一次自动推送检查（管理员）

```text
/apod_push_now
```

## 配置说明

插件配置项定义见 `_conf_schema.json`。下面是各字段的实际含义：

### `token`

NASA API Token，用于访问 APOD API。

- 类型：`string`
- 必填：是
- 申请地址：<https://api.nasa.gov/>

### `image`

是否发送图片。

- 类型：`bool`
- 默认值：`true`

### `explanation`

图片说明配置。

- 类型：`object`

子项：

- `is_show`：是否发送 NASA 官方说明，类型为 `bool`，默认 `true`
- `is_translate`：是否将说明翻译为简体中文，类型为 `bool`，默认 `true`

### `title`

标题配置。

- 类型：`object`

子项：

- `is_show`：是否发送标题，类型为 `bool`，默认 `true`
- `is_translate`：是否翻译标题，类型为 `bool`，默认 `true`

### `provider`

用于翻译的 LLM 提供商 ID。

- 类型：`string`
- 特殊类型：`select_provider`

说明：

- 只有在开启翻译时才需要配置
- 如果开启了标题或说明翻译，但没有配置 `provider`，插件会跳过翻译并发送 NASA 原文

### `date`

日期配置。

- 类型：`object`

子项：

- `is_show`：是否发送日期，类型为 `bool`，默认 `true`

### `is_divided`

是否将图片和文本分开发送。

- 类型：`bool`
- 默认值：`true`

说明：

- `true`：图片单独发送，标题、日期、说明合并为一个文本段发送
- `false`：图片和文本会组合为一条消息链发送，文本部分仍为一个完整文本段
- 该配置同时影响 `/apod` 命令和自动推送

### `timeout`

请求 NASA APOD 数据的超时时间，单位为秒。

- 类型：`int`
- 默认值：`120`

### `proxy`

NASA APOD API 请求代理。

- 类型：`string`
- 默认值：`""`

说明：

- 留空时不使用代理
- 示例：`http://127.0.0.1:7890`
- 如果 AstrBot 运行在 Docker 容器中，`127.0.0.1` 指容器内部，通常需要填写宿主机局域网 IP 或 Docker 可访问的代理地址
- 该代理只用于拉取 NASA APOD API 数据；图片 URL 发送仍由 AstrBot/平台适配器处理

### `retry_count`

NASA API 临时错误时的重试次数。

- 类型：`int`
- 默认值：`2`

### `push`

自动推送配置。

- 类型：`object`

子项：

- `enabled`：是否启用自动推送，类型为 `bool`，默认 `true`
- `target_unified_msg_origins`：目标会话 UMO 列表，类型为 `list`，默认 `[]`
- `daily_push_time`：每天自动推送时间（本机时区，24 小时制 `HH:MM`），类型为 `string`，默认 `"09:00"`
- `max_groups_per_round`：单轮最多推送数量，类型为 `int`，默认 `0`（不限制）

说明：

- UMO 可通过 `/sid` 获取
- 到达 `daily_push_time` 后执行一次推送，并按目标会话记录 APOD 日期，避免重复发送且不影响失败目标后续重试
- `daily_push_time` 使用运行 AstrBot 机器的本地时区

## 配置示例

```json
{
  "token": "YOUR_NASA_API_TOKEN",
  "image": true,
  "explanation": {
    "is_show": true,
    "is_translate": true
  },
  "title": {
    "is_show": true,
    "is_translate": true
  },
  "provider": "your_provider_id",
  "date": {
    "is_show": true
  },
  "is_divided": true,
  "timeout": 120,
  "proxy": "",
  "retry_count": 2,
  "push": {
    "enabled": true,
    "target_unified_msg_origins": [
      "aiocqhttp:GroupMessage:123456789"
    ],
    "daily_push_time": "09:00",
    "max_groups_per_round": 0
  }
}
```

## 返回行为

根据配置不同，插件会返回以下内容中的一部分：

- 图片：优先使用 `hdurl`，若不存在则回退到 `url`
- 标题：原文或翻译后的中文标题
- 日期：APOD 对应日期
- 说明：原文或翻译后的中文说明

如果当天 APOD 不是图片类型，而你又开启了 `image`，插件会直接返回错误提示，而不是继续发送文本内容。

## 缓存机制

插件包含三类缓存：

### APOD 数据缓存

- 会缓存最近获取到的 APOD 数据
- 当缓存仍然有效时，优先使用缓存，避免频繁请求 NASA API
- 缓存过期后会重新拉取最新数据

### 翻译结果缓存

- 标题和说明翻译结果会被缓存
- 相同文本再次出现时，优先使用缓存翻译结果
- 可以减少模型调用次数并提升响应速度

### 自动推送缓存

- `apod_push:target_sent:<target_hash>`：按目标会话记录最近一次已推送的 APOD 日期
- `apod_push:last_sent_date`：记录最近一次至少有一个目标成功推送的 APOD 日期，仅用于日志兼容
- `apod_push:last_payload:<date>`：记录某天 APOD 的已组装推送内容
- 一轮定时推送中只拉取一次 APOD，然后复用同一份内容推送给多个目标会话

## 翻译说明

当以下条件同时满足时，插件会自动调用 LLM 进行翻译：

- `title.is_translate` 或 `explanation.is_translate` 为 `true`
- 对应内容开启显示
- `provider` 已正确配置

翻译目标语言为简体中文，提示词偏向准确直译，不附加额外解释。

## 错误处理

插件对以下情况做了处理：

- NASA API 限流
- Token 无效或未授权
- 网络请求超时
- 临时服务错误，如 `502`、`503`、`504`
- 一般网络异常

在可重试错误下，插件会按照 `retry_count` 进行有限次重试。

## 使用建议

- 建议始终配置有效的 NASA API Token，避免公共额度带来的限制
- 如果你不需要中文翻译，可以关闭翻译选项并留空 `provider`
- 如果你希望聊天体验更自然，建议开启 `is_divided`
- 如果你更希望一次性返回完整内容，可以关闭 `is_divided`

## 项目地址

- Repository: <https://github.com/cruseth/astrbot_plugin_Astronomy_Picture_of_the_Day-APOD_fix>

## 致谢

- [NASA APOD API](https://api.nasa.gov/)
- [AstrBot](https://github.com/AstrBotDevs/AstrBot)
