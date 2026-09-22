# 目录收录台账

> 判据：能被**扫描/收录**（技术门槛） × **有多少开发者真的从这里找 MCP server**（流量） × 成本。
> 结论只出三种：`已提交 / 待条件 / 归档`。**每次只推进一个，做完再下一个。**
> 事实来源都标注了核对日期；机制会变，改前重核。

## 当前状态

| 目录 | 状态 | 阻塞条件 |
|---|---|---|
| 官方 MCP Registry | **待条件** | 需要用户在 `turingcorp.net` 加 DNS TXT（域名鉴权） |
| Smithery | **待条件** | 需要 Smithery 账号（用户）；服务侧已就绪 |
| mcp.so | 待办 | 需给 `chatmcp/mcpso` 提交 README PR |
| awesome-mcp-servers 等清单 | 待办 | 需仓转 public |
| Glama | 归档（最低优先） | 自动爬 GitHub，且**要求仓内有本地 stdio 入口**；要新建适配器 + 长期维护 + 多一个泄密面 |

## 1. 官方 MCP Registry（标准来源）

- 提交 `server.json`（本仓根已起草：`net.turingcorp/decider`）。
- **命名空间鉴权**：GitHub 型（`io.github.*`）或**域名型（`net.turingcorp/*`）**。我们用域名型
  ⇒ 需要在 `turingcorp.net` 加一条 **DNS TXT**（由 `mcp-publisher` 生成公私钥后给出具体值）。
  **只有用户能加**（zone 在 CF）。
- 远端格式**原生支持静态 Bearer**：`remotes[].headers[]` 可声明 `isRequired` / `isSecret`
  ⇒ **不需要 OAuth**。
- `version` 必须语义化且不接受范围写法；`name` 反向域名 + 恰好一个 `/`。
- **注意**：官方明确说 registry 仍处 **preview**，可能 break change 或数据重置。

## 2. Smithery（**第一优先**：URL 发布最顺、开发者第一入口）

核对日期 2026-09-22（smithery.ai/docs/build/publish）。

**发布路径**：`smithery.ai/new` → 填**公开 HTTPS URL** → 走完流程（Smithery Gateway 做反向代理）。

**硬性要求**：
- **Streamable HTTP** 传输 ✅（我们是）
- **OAuth support (if auth required)** ⚠️ —— 我们用的是**静态 Bearer（Agent Pass）**，不是 OAuth。
  Smithery 靠**未鉴权请求返回 401**（而非 403）来触发 OAuth discovery
  ⇒ 我们**实测是 401 + `WWW-Authenticate: Bearer realm="turingcorp-mcp"`**，形态正确。
  **但"静态 Bearer 能否过它的发布流程"没有文档保证 —— 这是本次提交要实测的第一件事。**

**扫描机制**：
- 公开 server → 自动扫描；需要认证的 server → 会提示你去认证后完成扫描。
- 扫不到时的兜底 = **静态 server card** `/.well-known/mcp/server-card.json`
  ⇒ **我们已按官方格式实现并上线**（`serverInfo{name,version}` +
  `authentication{required:boolean, schemes:string[]}` + `tools[]/resources[]/prompts[]`）。
- 罚则提醒：403 通常是 WAF/`Bot Fight Mode` 拦了 `SmitheryBot/1.0`。
  **内核方实测我们的 zone 对该 UA 返回 200** ✅（Bot Fight Mode 只拦 stock `Python-urllib`）。

### ✅ 扫描预演（2026-09-22 实测，用它自己的 UA 打我们的线上端点）

提交前先把"扫描器会遇到什么"演一遍，避免撞了才知道：

| 扫描动作 | 实测 | 判读 |
|---|---|---|
| 未鉴权 `initialize`（**legacy 代次** `2025-06-18`） | **200** | 旧代次握手能过 |
| 未鉴权 `initialize`（**现代** `2026-07-28` + initialize body） | **400** `-32020` | **这是对的**：现代代次无 initialize 握手；官方 handler 明确报"heads/body 不一致" |
| 未鉴权 `tools/list` | **200**，含 `"name":"decide"` | **工具清单可读** —— 扫描器能拿到全部能力 |
| 未鉴权 `tools/call` | **401** | ✅ Smithery 明确要求 401（非 403）|
| 响应头 | `www-authenticate: Bearer realm="turingcorp-mcp"` | 401 语义正确 |
| UA `SmitheryBot/1.0` 是否被 WAF 拦 | 未被拦（上述均正常返回） | ✅ |

**结论**：**发现面与鉴权面都满足 Smithery 的公开扫描要求。** 剩下的唯一不确定点是
"静态 Bearer（非 OAuth）能否过发布流程" —— 这条**只能实际提交才知道**，
文档写了"OAuth support (if auth required)"但没说不支持静态 Bearer。

**CLI 备选**：`smithery mcp publish "https://mcp.turingcorp.net/mcp" -n @<org>/<server> --config-schema '{...}'`

**发布后**：`Settings → Verification` 有官方厂商验证清单。

## 3. mcp.so

给 `chatmcp/mcpso` 仓提一个 README PR、加一行。**几乎零维护成本**（无验证、无质检信号 ⇒ 流量质量也低）。

## 4. awesome-mcp-servers 等 GitHub 清单

长期 SEO 复利，但**需要仓转 public**。

## 5. Glama（最后做）

自动爬**公开 GitHub 仓**，且**仓内必须有本地 stdio 入口**（明确拒绝纯 URL）。
⇒ 要建独立公开仓 + stdio 适配器（同一套 tool schema 走 stdio、handler 转发到托管端点）
+ npm 发布 + 长期同步 + 多一个泄密面。**性价比最低，放最后。**

## 提交前检查清单（每次提交前跑）

- [ ] 线上冒烟全绿（尤其：未鉴权调用返 **401** 而非 403、免鉴权 `tools/list` 返 200）
- [ ] 落地页与 `llms.txt` 的**价格口径**与 Poe 渠道页逐字一致
- [ ] 无内部实现细节（架构、模型路由、供应商、成本口径）
- [ ] 无 benchmark 题面与参考答案、无竞品评价、无营销夸大词
- [ ] 无 SLA、无退款政策、无自动化暗示（`auto-accept` / `unattended` 等）
- [ ] `server.json` 过了官方 schema；server-card 的官方字段齐全
- [ ] 提交后**读回目录页**，确认抓到的描述与我们的口径一致（不信"提交成功"）
