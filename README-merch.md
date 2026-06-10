# 🚀 商户智能管理平台 - RuoYi AI + DeerFlow 集成方案

> **核心思路**：在 RuoYi AI 商户管理系统中增加**AI聊天气泡**，调用 DeerFlow 的 Agent 能力，DeerFlow 通过 **MCP 协议**调用 RuoYi AI 提供的业务工具。

---

## 一、架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                   商户管理系统 (RuoYi AI)                    │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │  Vue3 前端    │   │  聊天气泡 💬  │   │  MCP Server   │  │
│  │  (商户管理)   │──▶│  (新增组件)   │◀─┤  (10个工具)    │  │
│  └──────────────┘   └──────┬───────┘   └───────┬────────┘  │
│                            │                   │            │
└────────────────────────────┼───────────────────┼────────────┘
                     HTTP/SSE │                   │ MCP Protocol
                              ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                    DeerFlow (AI 大脑)                        │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │  Lead Agent  │   │  Intent      │   │  MCP Client    │  │
│  │  (LangGraph) │──▶│  Classification│─▶│  (调用工具)     │  │
│  └──────────────┘   └──────────────┘   └────────────────┘  │
│                                                             │
│  端口: 8001 (或 Docker: 2026)                               │
└─────────────────────────────────────────────────────────────┘
```

### 数据流

```
用户在聊天气泡输入: "查询商户 M001234567 的交易记录"
        ↓
① RuoYi AI 前端 → POST /api/chat (DeerFlow)
        ↓
② DeerFlow Lead Agent 意图识别 → "交易查询"
        ↓
③ DeerFlow MCP Client → 调用 RuoYi AI 的 transaction_query 工具
        ↓
④ RuoYi AI 执行 SQL 查询 → 返回数据
        ↓
⑤ DeerFlow LLM 生成自然语言回答
        ↓
⑥ SSE 流式返回给前端聊天气泡显示
```

---

## 二、核心集成点

### 2.1 RuoYi AI 前端：新增聊天气泡组件

**位置**: `ruoyi-ai-java/ruoyi-admin/src/views/`

```vue
<!-- MerchantChatBubble.vue -->
<template>
  <div class="chat-container">
    <div class="messages" ref="messagesContainer">
      <div v-for="msg in messages" :key="msg.id" :class="['message', msg.role]">
        <div class="avatar">{{ msg.role === 'user' ? '👤' : '🤖' }}</div>
        <div class="content" v-html="msg.content"></div>
      </div>
    </div>

    <div class="input-area">
      <el-input
        v-model="inputText"
        type="textarea"
        placeholder="输入问题，如：查询商户交易记录、进件审核状态..."
        @keyup.enter="sendMessage"
      />
      <el-button type="primary" @click="sendMessage">发送</el-button>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import { ElMessage } from 'element-plus'

const messages = ref([])
const inputText = ref('')
const messagesContainer = ref(null)

const DEERFLOW_API = import.meta.env.VITE_DEERFLOW_API || 'http://localhost:2026/api'

async function sendMessage() {
  if (!inputText.value.trim()) return

  const userMessage = {
    id: Date.now(),
    role: 'user',
    content: inputText.value
  }
  messages.value.push(userMessage)
  const query = inputText.value
  inputText.value = ''

  await scrollToBottom()

  const assistantMessage = {
    id: Date.now() + 1,
    role: 'assistant',
    content: ''
  }
  messages.value.push(assistantMessage)

  try {
    const response = await fetch(`${DEERFLOW_API}/chat/stream`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getToken()}`
      },
      body: JSON.stringify({
        message: query,
        thread_id: `merchant-${getCurrentMerchantId()}`,
        metadata: {
          source: 'ruoyi-admin',
          merchant_no: getCurrentMerchantId()
        }
      })
    })

    if (!response.ok) throw new Error('网络错误')

    const reader = response.body.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      const text = decoder.decode(value, { stream: true })
      const lines = text.split('\n').filter(line => line.startsWith('data:'))

      for (const line of lines) {
        const data = JSON.parse(line.slice(5))
        if (data.type === 'token') {
          assistantMessage.content += data.content
          await scrollToBottom()
        }
      }
    }
  } catch (error) {
    console.error('Chat error:', error)
    assistantMessage.content = '抱歉，AI助手暂时无法响应，请稍后重试。'
    ElMessage.error('连接AI服务失败')
  }
}

function scrollToBottom() {
  return nextTick(() => {
    if (messagesContainer.value) {
      messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
    }
  })
}

function getCurrentMerchantId() {
  return useRoute().params.merchantNo || 'default'
}
</script>

<style scoped>
.chat-container {
  height: 600px;
  display: flex;
  flex-direction: column;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.message {
  display: flex;
  margin-bottom: 16px;
  gap: 8px;
}

.message.user {
  flex-direction: row-reverse;
}

.avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f0f2f5;
}

.content {
  max-width: 70%;
  padding: 10px 14px;
  border-radius: 8px;
  background: #f0f2f5;
  line-height: 1.6;
}

.message.user .content {
  background: #409eff;
  color: white;
}

.input-area {
  padding: 12px;
  border-top: 1px solid #e4e7ed;
  display: flex;
  gap: 8px;
}
</style>
```

### 2.2 RuoYi AI 后端：提供 MCP 工具接口

**位置**: `ruoyi-ai-java/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/controller/mcp/`

```java
@RestController
@RequestMapping("/mcp")
public class McpToolController {

    @Autowired
    private MerchantService merchantService;

    @Autowired
    private TransactionService transactionService;

    @Autowired
    private RiskControlService riskControlService;

    @PostMapping("/tools/list")
    public List<McpTool> listTools() {
        return Arrays.asList(
            McpTool.builder()
                .name("merchant_query")
                .description("查询商户信息，支持按商户号、手机号、名称查询")
                .parameters(Map.of(
                    "merchantNo", Map.of("type", "string", "description", "商户编号"),
                    "phone", Map.of("type", "string", "description", "手机号"),
                    "name", Map.of("type", "string", "description", "商户名称")
                ))
                .build(),

            McpTool.builder()
                .name("transaction_query")
                .description("查询商户交易流水，支持时间范围筛选")
                .parameters(Map.of(
                    "merchantNo", Map.of("type", "string", "required", true),
                    "startDate", Map.of("type", "string", "format", "yyyy-MM-dd"),
                    "endDate", Map.of("type", "string", "format", "yyyy-MM-dd"),
                    "pageSize", Map.of("type", "integer", "default", 20)
                ))
                .build(),

            McpTool.builder()
                .name("merchant_risk_check")
                .description("风控检查，返回风险评分和建议")
                .parameters(Map.of(
                    "merchantNo", Map.of("type", "string", "required", true),
                    "sceneType", Map.of("type", "string", "enum", Arrays.asList("entry", "transaction")),
                    "amount", Map.of("type", "number", "description", "交易金额")
                ))
                .build(),

            McpTool.builder()
                .name("knowledge_retrieval")
                .description("RAG知识库检索，用于回答FAQ类问题")
                .parameters(Map.of(
                    "query", Map.of("type", "string", "required", true),
                    "topK", Map.of("type", "integer", "default", 5),
                    "knowledgeBaseId", Map.of("type", "string", "default", "faq")
                ))
                .build(),

            McpTool.builder()
                .name("ticket_create")
                .description("创建工单，转人工处理")
                .parameters(Map.of(
                    "title", Map.of("type", "string", "required", true),
                    "type", Map.of("type", "string", "enum", Arrays.asList("complaint", "consult", "technical")),
                    "content", Map.of("type", "string", "required", true),
                    "merchantNo", Map.of("type", "string")
                ))
                .build()
        );
    }

    @PostMapping("/tools/call")
    public McpToolResult callTool(@RequestBody McpToolCallRequest request) {
        switch (request.getName()) {
            case "merchant_query":
                return merchantQuery(request.getArguments());
            case "transaction_query":
                return transactionQuery(request.getArguments());
            case "merchant_risk_check":
                return riskCheck(request.getArguments());
            case "knowledge_retrieval":
                return knowledgeRetrieval(request.getArguments());
            case "ticket_create":
                return ticketCreate(request.getArguments());
            default:
                throw new IllegalArgumentException("Unknown tool: " + request.getName());
        }
    }

    private McpToolResult merchantQuery(Map<String, Object> args) {
        String merchantNo = (String) args.get("merchantNo");
        String phone = (String) args.get("phone");

        MerchantDTO merchant = StringUtils.hasText(merchantNo)
            ? merchantService.getByMerchantNo(merchantNo)
            : merchantService.getByPhone(phone);

        return McpToolResult.success(merchant);
    }

    private McpToolResult transactionQuery(Map<String, Object> args) {
        String merchantNo = (String) args.get("merchantNo");
        String startDate = (String) args.getOrDefault("startDate", getYesterday());
        String endDate = (String) args.getOrDefault("endDate", getToday());
        int pageSize = (Integer) args.getOrDefault("pageSize", 20);

        List<TransactionDTO> transactions =
            transactionService.queryByMerchantNo(merchantNo, startDate, endDate, pageSize);

        return McpToolResult.success(transactions);
    }

    // ... 其他工具实现类似
}
```

### 2.3 DeerFlow 配置：连接 RuoYi AI MCP Server

**配置文件**: `deer-flow-ai-python/config.yaml`

```yaml
models:
  - name: deepseek-chat
    use: deerflow.models.patched_deepseek:PatchedChatDeepSeek
    api_base: https://api.deepseek.com/v1
    api_key: $DEEPSEEK_API_KEY

mcp:
  servers:
    - name: "ruoyi-mcp"
      url: "http://localhost:8080/mcp"
      auth:
        type: "jwt"
        token: "${RUOYI_MCP_TOKEN}"
      tools:
        - merchant_query
        - transaction_query
        - merchant_risk_check
        - knowledge_retrieval
        - ticket_create
        - blacklist_check
        - settlement_query
        - ocr_recognize
        - audit_decision
        - report_generate

gateway:
  host: "0.0.0.0"
  port: 2026

memory:
  enabled: true
  storage_type: local
```

### 2.4 DeerFlow 后端：Chat API 对接

**已内置支持**，无需额外开发：

```python
# backend/app/gateway/routers/chat.py (伪代码)

@router.post("/api/chat/stream")
async def chat_stream(request: ChatRequest):
    async def event_generator():
        client = DeerFlowClient()

        async for event in client.stream(
            message=request.message,
            thread_id=request.thread_id,
            metadata=request.metadata
        ):
            if event.type == "token":
                yield f"data: {json.dumps({'type': 'token', 'content': event.content})}\n\n"
            elif event.type == "tool_call":
                yield f"data: {json.dumps({'type': 'tool_call', 'tool': event.tool_name})}\n\n"

        yield "data: [DONE]\n\n"

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive"
        }
    )
```

---

## 三、MCP 工具清单（10个）

| # | 工具名 | 功能 | 适用场景 | 核心参数 |
|---|--------|------|---------|---------|
| 1 | `merchant_query` | 商户信息查询 | 所有场景 | merchantNo / phone |
| 2 | `transaction_query` | 交易流水查询 | 客服/风控/分析 | merchantNo, dateRange |
| 3 | `merchant_risk_check` | 风控评分 | 进件/风控 | merchantNo, sceneType, amount |
| 4 | `knowledge_retrieval` | RAG知识检索 | 客服FAQ | query, topK, knowledgeBaseId |
| 5 | `ticket_create` | 创建工单 | 客服转人工 | title, type, content |
| 6 | `blacklist_check` | 黑名单校验 | 风控/进件 | listType, value |
| 7 | `settlement_query` | 结算查询 | 分析/客服 | merchantNo, dateRange |
| 8 | `ocr_recognize` | OCR证件识别 | 进件审核 | documentId, docType |
| 9 | `audit_decision` | 审核决策记录 | 进件审核 | merchantNo, decision, reason |
| 10 | `report_generate` | 报告生成 | 数据分析 | type(daily/weekly/monthly), date |

---

## 四、快速实施步骤（5步）

### Step 1: 启动 RuoYi AI（1小时）

```bash
cd ruoyi-ai-java
docker-compose -f docs/docker/ruoyi-ai/docker-compose-all.yaml up -d

# 验证
curl http://localhost:8080/mcp/tools/list
# 返回10个MCP工具定义 ✅
```

### Step 2: 启动 DeerFlow（30分钟）

```bash
cd deer-flow-ai-python
cp config.example.yaml config.yaml
make setup          # 配置LLM API Key、MCP地址等
make docker-start

# 验证
curl http://localhost:2026/docs
# Swagger文档可访问 ✅
```

### Step 3: 集成聊天气泡到 RuoYi AI 前端（2小时）

```bash
# 复制上面 2.1 的 Vue 组件到项目
cp MerchantChatBubble.vue ruoyi-ai-java/ruoyi-admin/src/views/merchant/components/

# 在商户详情页引入
# ruoyi-ai-java/ruoyi-admin/src/views/merchant/detail.vue
import MerchantChatBubble from './components/MerchantChatBubble.vue'
```

### Step 4: 配置环境变量（15分钟）

```bash
# ruoyi-ai-java/.env
VITE_DEERFLOW_API=http://localhost:2026/api
RUOYI_MCP_TOKEN=your-jwt-token-here

# deer-flow-ai-python/.env
DEEPSEEK_API_KEY=sk-xxx
RUOYI_MCP_URL=http://host.docker.internal:8080/mcp
```

### Step 5: 测试验证（30分钟）

**测试用例**：

```bash
# Test 1: FAQ问答
输入: "如何提现？费率是多少？"
预期: 从知识库检索答案并返回 ✅

# Test 2: 数据查询
输入: "查询商户 M001234567 最近7天的交易记录"
预期: 调用 transaction_query 并格式化展示 ✅

# Test 3: 风控评估
输入: "评估商户 M001234567 的风险等级"
预期: 调用 merchant_risk_check 并给出评分 ✅

# Test 4: 工单创建
输入: "我要投诉，商户 M001234567 结算延迟"
预期: 调用 ticket_create 并确认工单号 ✅
```

---

## 五、典型对话示例

### 示例 1：智能客服（FAQ + 知识库）

```
用户: 如何提现？手续费多少？
AI:   👤 用户提问...

🤖 AI正在思考... [调用 knowledge_retrieval]

🤖 根据商户平台规则：

✅ **提现流程**：
1. 登录商户后台 → 资金管理 → 提现申请
2. 选择提现银行卡（需先绑定）
3. 输入提现金额（最低100元）
4. 确认提交，等待审核（T+1到账）

💰 **手续费标准**：
- 普通提现：0.1%，最低2元，最高50元
- 加急提现：0.3%，最低5元，无上限
- VIP商户：免手续费（月交易额>100万）

📞 如有疑问，可拨打客服热线：400-xxx-xxxx
   或点击【转人工】为您服务
```

### 示例 2：数据查询（MCP工具调用）

```
用户: 查询商户 TEST001 最近一个月的交易统计
AI:   👤 用户提问...

🤖 正在查询数据... [调用 transaction_query]

📊 **商户 TEST001 交易概览**（2026-05-01 ~ 2026-05-29）

| 指标 | 数值 |
|------|------|
| 总交易笔数 | 1,234 笔 |
| 总交易金额 | ¥89,567.80 |
| 平均客单价 | ¥72.58 |
| 成功率 | 98.5% |

📈 **趋势分析**：
- 日均交易：42笔，较上月增长12%
- 高峰时段：11:00-13:00（午餐）、18:00-20:00（晚餐）
- 异常提醒：5月15日交易量骤降40%（可能是系统维护日）

💡 建议：该商户经营状况良好，可考虑提升授信额度
```

### 示例 3：复杂任务（多工具协作）

```
用户: 帮我审核新商户 NEW_MERCHANT，并给出建议
AI:   👤 用户收到审核请求...

🤖 正在执行审核流程... [多步骤任务]

Step 1️⃣: 查询商户基本信息 [merchant_query]
✅ 商户新鲜水果店，个体工商户，联系人：张三

Step 2️⃣: OCR识别证件 [ocr_recognize]
✅ 营业执照：有效（至2028年）
✅ 身份证：匹配
⚠️ 银行卡：姓名一致，但为二类账户

Step 3️⃣: 黑名单校验 [blacklist_check]
✅ 未命中黑名单

Step 4️⃣: 风控评分 [merchant_risk_check]
📊 综合评分：35分（低风险）

📋 **审核结论**：

| 维度 | 得分 | 说明 |
|------|------|------|
| 资料完整度 | 90/100 | 证件齐全，清晰度高 |
| 证件真实性 | 95/100 | OCR验证通过 |
| 行业风险 | 70/100 | 餐饮行业中等风险 |
| 银行账户 | 60/100 | 二类账户，限额较低 |

✅ **建议操作**：**自动通过**
- 风险等级：低
- 授信额度建议：¥50,000
- 需关注：引导升级为一类银行账户

是否执行【通过】操作？或【转人工】复核？
```

---

## 六、技术要点

### 6.1 认证鉴权

```java
// RuoYi AI MCP Server 使用 JWT 鉴权
@Configuration
public class McpSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/mcp/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/mcp/tools/list").permitAll()  // 工具列表公开
                .requestMatchers("/mcp/tools/call").authenticated()  // 工具调用需认证
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}));

        return http.build();
    }
}
```

### 6.2 错误处理

```javascript
// 前端错误降级策略
try {
  await sendMessage()
} catch (error) {
  if (error.code === 'ECONNREFUSED') {
    assistantMessage.content = '⚠️ AI服务暂时不可用，请稍后重试或联系管理员'
  } else if (error.response?.status === 401) {
    assistantMessage.content = '⚠️ 登录已过期，请刷新页面重新登录'
    await refreshToken()
  } else {
    assistantMessage.content = '❌ 未知错误：' + error.message
  }
}
```

### 6.3 性能优化

```yaml
# DeerFlow config.yaml 优化配置
models:
  - name: deepseek-chat  # 成本敏感场景使用 DeepSeek
    use: deerflow.models.patched_deepseek:PatchedChatDeepSeek
    # 比GPT-4o便宜10倍，中文效果好

cache:
  enabled: true
  ttl: 3600  # 缓存常见FAQ答案1小时

rate_limiting:
  enabled: true
  requests_per_minute: 30  # 单用户限流
```

---

## 七、扩展方向

### Phase 2（可选）：更多渠道接入

DeerFlow 已内置6+IM渠道，可直接复用：

```yaml
# config.yaml 新增渠道配置
channels:
  wechat:
    enabled: true
    app_id: $WECHAT_APP_ID
    app_secret: $WECHAT_APP_SECRET

  dingtalk:
    enabled: true
    app_key: $DINGTALK_APP_KEY
    app_secret: $DINGTALK_APP_SECRET

  feishu:
    enabled: true
    app_id: $FEISHU_APP_ID
    app_secret: $FEISHU_APP_SECRET
```

**效果**：商户不仅可以在Web后台聊天，还能通过微信/钉钉/飞书与AI助手交互。

---

## 八、项目结构（精简版）

```
agent-flow/
├── deer-flow-ai-python/           # DeerFlow AI引擎
│   ├── backend/
│   │   ├── app/gateway/           # FastAPI网关（含Chat API）
│   │   └── packages/harness/deerflow/
│   │       ├── mcp/client.py      # MCP客户端（调用RuoYi工具）
│   │       └── agents/lead_agent/ # 主Agent编排
│   └── config.yaml                # 配置文件
│
├── ruoyi-ai-java/                 # RuoYi AI 商户管理系统
│   ├── ruoyi-admin/               # Vue3前端
│   │   └── src/views/merchant/
│   │       └── components/
│   │           └── MerchantChatBubble.vue  # ← 新增聊天气泡
│   └── ruoyi-modules/ruoyi-chat/
│       └── src/main/java/.../controller/mcp/
│           └── McpToolController.java     # ← MCP工具接口
│
└── README-merch.md                # 本文档
```

---

## 九、快速参考

| 服务 | 地址 | 用途 |
|------|------|------|
| RuoYi Admin | http://localhost:25666 | 商户管理后台 |
| RuoYi MCP | http://localhost:8080/mcp | MCP工具接口 |
| DeerFlow API | http://localhost:2026/docs | AI聊天API文档 |
| DeerFlow Chat | http://localhost:2026/api/chat/stream | SSE流式聊天端点 |

---

**最后更新**: 2026-05-29  
**版本**: 2.0 (精简版 - 聚焦聊天气泡+MCP 集成)

---

## 🔒 安全与隐私提示

### 敏感信息保护

本文档中涉及的以下信息属于敏感配置，**严禁**提交到代码仓库：

| 类型 | 示例变量 | 说明 |
|------|----------|------|
| LLM API Key | `DEEPSEEK_API_KEY` | DeepSeek、OpenAI 等模型的 API 密钥 |
| MCP Token | `RUOYI_MCP_TOKEN` | MCP 服务认证 JWT Token |
| 渠道凭证 | `WECHAT_APP_ID`、`WECHAT_APP_SECRET` | 微信/钉钉/飞书应用凭证 |
| 数据库连接 | `DB_PASSWORD` | 数据库密码 |

### 正确做法

✅ **使用环境变量**：
```bash
# .env 文件（加入 .gitignore）
DEEPSEEK_API_KEY=sk-your-actual-key
RUOYI_MCP_TOKEN=your-jwt-token
```

✅ **使用配置管理工具**：
- Docker Secrets
- Kubernetes Secrets
- HashiCorp Vault
- AWS Secrets Manager

✅ **代码审查**：
- 提交前检查 `git diff`，确保无敏感信息
- 使用预提交钩子（如 `git-secrets`、`detect-secrets`）

### 示例数据说明

文档中的示例数据（如商户号 `M001234567`、手机号等）均为**虚构数据**，仅用于演示目的。

生产环境中请使用真实数据，并遵守 GDPR、个人信息保护法等相关法规。
