# feat: 跟随当前模型显示余额（DeepSeek 官方 / OpenAI 兼容中转站自动切换）

## 解决什么问题

内置余额模块永远显示 DeepSeek 官方余额（`{balance_ds}` 固定取官方接口）。
用户实际在用 OpenAI 兼容中转站（OneAPI / New API）的模型时，
泡泡里仍是官方余额 —— 显示的账户和正在用的账户对不上。

相关：issue #45（自动识别当前会话使用的模型）、issue #250（无法区分当前正在使用的账户）、
PR #174（服务端归属，未合并）、PR #164（open）。

## 做法

按 **provider** 而非 model id 判定，自动切换余额来源：

| 当前会话 provider | 显示 |
|---|---|
| `deepseek-official` / `deepseek-account` | DeepSeek 官方余额 |
| 其它（含中转站） | 该模型在「自定义 API 模型」里配的余额接口 |

### 为什么必须按 provider 判

只看 model id 会**静默错标**，三条实测证据：

1. 同一个 `deepseek-flash` 可以同时属于 `deepseek-official` 与 `deepseek-account`，只看名字分不开；
2. 中转站可以注册与官方同名的 model id（`cordis` 里 provider 列表是用户自定义的）；
3. 中转站会改写上游模型名 —— 实测 `source.model = deepseek-v4.1-flash-1017`
   而 `replayState.response.responseModel = deepseek-v4.1-flash`。

按名字分类的后果是把第三方中转流量误标成官方余额，且**不报错**。

## 数据源（两级，互为备份）

1. **宿主会话事件**（权威）：监听 `request/context` / `model/selection` /
   `request/header` / `assistant/message`，四者都带真实 `provider`，
   经 `/dsh-whale/last-turn.json` 新增的 `activeModel` 字段下发给前端。
2. **DOM 兜底**：读模型选择器的 `[data-provider-id][data-model-id]` 属性
   （只认 `triggerLabel` 容器内、排除 `[role="menu"]` 里的选项，避免被列表项或历史消息带偏）。
   宿主未更新时也能工作。

## 顺带修的两个观感问题

- **标题自动缩字号**：来源名变长后（如「OpenAI 兼容中转站 余额」）会超出泡泡内宽 `560u`
  并被折成两行。现在超宽时先缩字号（常规下限 60%，硬下限 40%），仍放不下才折行。
  **只对 `type:'text'` 生效** —— 随机语句/长句是用户故意写长的，应当折行而非缩小。
  放得下时一个像素都不动，官方模型下外观与改动前完全一致。
- **加载中不冒充官方**：正在用中转站但它的余额还没拉到时，显示 `…`（加载中）/ `—`（未配置），
  而不是回落成官方余额。另外支持 `{source}` 占位符。

## 兼容性

- 纯增量：新增 `activeModel` 字段、新增主菜单开关「跟随当前模型」（默认开，可关回旧行为）
- 不改账本结构、不改 `.dshw-api.json` 结构、不碰凭据
- 老宿主（无 `request/context` 事件）自动走 DOM 兜底

## 验证

- 18 条缩字号断言 + 10 条跟随逻辑断言全过
- `node --check` 通过；patch 对官方 0.4.2 干净 `git apply`，产出与本机改动逐字节一致
