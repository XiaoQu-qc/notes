```
用户自然语言
“帮我创建一个需求单”
        │
        ▼
企业微信回调 / 聊天调试台
        │
        ▼
Platform Server runOnmachine()//这个是关键函数，发送/api/agents/run到找到的绑定的机器上     runOnMachine在    platform-server/internal/api/wecom_agent_bridge.go下
- 识别当前用户
- 找到用户绑定的 Agent 机器
- 确定 agentId、模型、API 权限
- 不解析“创建需求单”这个业务意图
        │
        │ POST /api/agents/run
        │ {
        │   "agentId": "wecom-assistant",
        │   "prompt": "帮我创建一个需求单",
        │   ...
        │ }
        ▼
目标机器 Agent Server
- 加载 wecom-assistant 配置 //根据agentId"找到代码中wecom-assistant的config配置
- 准备 system prompt
- 物化 codelix-console skill
- 注入 MCP 配置和 API 凭证
        │
        ▼
启动 Claude/Codex
        │
        ├─ 用户消息：帮我创建一个需求单
        ├─ 系统提示词：你是企微助手，动手前读 codelix-console
        ├─ Skill：Codelix API 的接口说明和调用配方
        └─ MCP 工具：mcp__codelix_api__request
        │
        ▼
大模型理解用户意图
- 判断用户要创建需求
- 读取建单操作说明
- 判断信息是否完整
- 不完整则追问
- 完整则决定调用 API
        │
        ▼
调用 MCP 工具
mcp__codelix_api__request({
  method: "POST",
  path: "/api/tapd/tech-story",
  body: {...}
})
        │
        ▼
MCP codelix_api Server
        │ HTTP + X-Codelix-Run-Token
        ▼
目标机器本地 Agent Server
POST /api/tapd/tech-story
        │
        ▼
TechStoryCreate()
        │
        ▼
调用 TAPD stories_create
        │
        ▼
TAPD 返回创建结果
        │
        ▼
结果作为 tool_result 返回 Claude/Codex
        │
        ▼
Claude/Codex 生成自然语言回复
“需求已经创建，需求编号是……”
        │
        ▼
Agent Server 转成 SSE
        │
        ▼
Platform Server 流式转发
        │
        ├─ 正式环境：返回企业微信
        └─ 调试环境：返回聊天调试台
```
