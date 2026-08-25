# 商汤 SenseNova 网关限流误报配额 + DSH 重试配置（踩坑记录）

> 日期：2026-08-25
> 场景：DSH web 通过 `llm-pi-ai` 配置的 `sensenova` 网关（`https://token.sensenova.cn/v1`）调用模型，频繁出现「本轮运行失败 429」。
> 结论速览：**商汤把 RPM 限流误报成 `insufficient_quota` / `quota_exceeded_error`，DSH 默认把它当终止性 QUOTA 错误不重试 → 需要把 `QUOTA` 纳入 `retryPolicy.retryableCodes`。**

---

## 一、现象

- 会话中模型调用报错：

  ```
  本轮运行失败 429: {"message":"Allocated quota exceeded, please increase your quota limit.","type":"invalid_request_error","code":"insufficient_quota"}
  ```

- 但用户在商汤控制台看到各模型配额 **96%–100% 剩余**（deepseek-v4-flash 96.20%、glm-5.2 99.33%、其它 100%），**配额明明充足**。

## 二、排查过程

### 1. 直接 curl 复现（绕过 DSH）

```bash
curl -s -X POST "https://token.sensenova.cn/v1/chat/completions" \
  -H "Authorization: Bearer $SENSENOVA_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

返回与 DSH 所见完全一致：`insufficient_quota`。

### 2. 逐模型批量测试（发现差异）

| 模型 | 结果 |
|---|---|
| `deepseek-v4-flash` | ⚠️ **间歇性**：同参数连续调用时好时坏 |
| `sensenova-6.7-flash-lite` / `sensenova-6.8-flash-lite` | ✅ 稳定成功 |
| `glm-5.2` | ❌ `Workspace allocated quota exceeded`（工作空间措辞） |
| `sensenova-u1-fast` / `sensenova-u1.5-lite` | ❌ `model is not found`（/v1/models 列表里有但调不通） |

### 3. 并发测试（定位真相）

8 个并发请求打 `deepseek-v4-flash`：**4 成功 / 4 失败**，失败响应原文：

```json
{"error":{"message":"rpm exhausted","type":"quota_exceeded_error","code":"8"}}
```

**`rpm exhausted` = Requests Per Minute（每分钟请求数）用尽 → 这是速率限制，不是配额不足！**

连续打时返回 HTTP 429 + `rpm exhausted`，触发后窗口期内持续被拒（连续 4 次全失败），窗口过了自动恢复（串行又成功）——典型的按分钟配额限流特征。

## 三、根因

1. **商汤网关把 RPM 限流误报为配额类错误**：`rpm exhausted`（code 8）/ `Allocated quota exceeded`（code insufficient_quota）本质都是其限流保护，与真实账户配额无关。
2. **DSH 的错误分类把这类 429 判成终止性 QUOTA**：`llm/src/error.ts` 的 `isQuotaExceededError()` 正则命中 `insufficient quota`、`quota ... exceeded` 等文案 → 归为 `QUOTA` 码。
3. **默认重试策略不含 QUOTA**：`llm/src/retry-policy.ts` 的默认 `retryableCodes` 只有 `EMPTY_RESPONSE / RATE_LIMIT / SERVER / TIMEOUT / TRANSPORT`，**不含 `QUOTA`** → llm-retry 插件对 QUOTA 直接放弃 →「本轮运行失败」卡死。

> DSH 这样设计本身是对的（真配额用尽重试无意义），但商汤的误报让「可恢复的限流」被误判为「不可恢复的配额」，属于网关侧错误分类 + 客户端默认策略的叠加坑。

## 四、修复方案（settings.yaml）

在 `llm-pi-ai.providers.<供应商>` 下显式配置 `retryPolicy`，**把 `QUOTA` 加入 `retryableCodes`** 并加大退避：

```yaml
llm-pi-ai:
  providers:
    sensenova:
      apiKeyEnv: SENSENOVA_API_KEY
      api: openai-completions
      baseURL: https://token.sensenova.cn/v1
      retryPolicy:
        mode: normal
        maxRetries: 4                     # 窗口恢复前最多重试 4 次
        retryableCodes:
          - EMPTY_RESPONSE
          - RATE_LIMIT
          - QUOTA                           # ← 关键：商汤限流误报成 QUOTA，必须放行重试
          - SERVER
          - TIMEOUT
          - TRANSPORT
        backoff:
          initialDelayMs: 2000              # 首次等待 2s
          maxDelayMs: 60000                 # 最长等 60s（上限 MAX_TIMER_DELAY_MS = 2^31-1）
          jitterRatio: 0.25                 # ±25% 抖动，避免同步风暴
```

生效步骤：

```bash
# 1) 校验 YAML
python3 -c "import yaml; yaml.safe_load(open('/root/.dsh/settings.yaml'))"
# 2) 重启后端（systemd 托管）
systemctl restart dsh-web
# 3) 确认无配置报错
journalctl -u dsh-web -n 50 --no-pager | grep -iE "retryPolicy|invalid"
```

重试执行逻辑（`llm-retry/src/index.ts`）：退避 = `min(initialDelayMs × 2^(retry-1), maxDelayMs) × (1 ± jitterRatio)`，每轮重试写入会话事件 `llm/retry`，尊重服务商 `Retry-After`，超过 `maxRetries` 才放弃。

## 五、验证

- YAML 解析通过，`QUOTA in retryableCodes == True`；
- 服务重启后 `systemctl is-active dsh-web` 为 active，日志无 retryPolicy 报错；
- 被限流的请求现在会走「等待退避 → 重试」路径，窗口恢复后自动成功。

## 六、顺带发现的其它坑（同网关）

1. **`sensenova-u1-fast` / `sensenova-u1.5-lite`**：`/v1/models` 列表存在，但 `chat/completions` 恒报 `model is not found`（code 5）→ 商汤侧接入问题，重试无效，**别选这两个模型**。
2. **`glm-5.2`**：报 `Workspace allocated quota exceeded`（工作空间配额措辞，另一套池子）；若控制台显示剩余充足，同样属误报，上述重试配置可救；持续失败则需找商汤确认工作空间。
3. **sensenova 的模型没有推理强度调节**：sensenova 不在 pi-ai 内置 provider 目录，`models` 手写条目无 `reasoningEfforts` 时 `reasoning` 默认为 false（`llm-pi-ai/src/catalog.ts` 的 `resolveModelReasoning`），UI 不显示推理强度选项。要开启需在模型条目里手写：
   ```yaml
   - id: deepseek-v4-flash
     reasoningEfforts: { off: null, low: 'low', high: 'high', max: 'max' }
   ```

## 七、背景补充：pi-ai 升级才能让新模型有推理能力

- `glm-5.3` 此前无推理强度：因为 pi-ai 0.82.1 内置目录没有它 → 升级 `@earendil-works/pi-ai` 到 **0.84.3**（npm 最新，2026-08-24 发布）后内置 zai-coding-cn 目录新增 `glm-5.3`（reasoning: true, low/high/max 三级）。
- **升级陷阱**：`dsh` 的 `dsh-llm-pi-ai` 声明 `^0.82.1`，npm 会在 `node_modules/@deepseek-ai/dsh-llm-pi-ai/node_modules/@earendil-works/pi-ai` **嵌套装一份 0.82.1**；运行时 ESM 解析优先命中嵌套旧版 → 顶层升到 0.84.3 也不生效。**删除嵌套旧版**后 ESM 会向上解析到顶层 0.84.3：
  ```bash
  rm -rf /root/.npm-global/lib/node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai/dsh-llm-pi-ai/node_modules/@earendil-works
  systemctl restart dsh-web
  ```
- 验证：
  ```bash
  node --input-type=module -e "
  import { getBuiltinModels } from /* 顶层路径 */ '@earendil-works/pi-ai/providers/all';
  console.log(getBuiltinModels('zai-coding-cn').map(m => m.id));
  "
  ```