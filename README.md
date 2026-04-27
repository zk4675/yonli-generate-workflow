核心能力
1. 自动解析业务需求

系统接收业务需求、用户故事或变更说明后，会自动提取：

业务目标
核心实体
风险关键词
P0 / P1 关注点
改写后的独立需求描述

改写后的需求会用于后续知识检索和测试用例生成，提升检索准确性和生成稳定性。

2. 双步交叉验证机制

系统同时使用两类知识进行测试设计：

测试规范库

用于约束首轮测试用例生成，保证用例结构规范、覆盖基础主流程和常规边界场景。

典型内容包括：

测试设计规范
P0 / P1 标准
通用测试模板
边界值设计规则
权限、兼容性、幂等性检查清单
历史缺陷库

用于在首轮用例生成后做二次反思，自动补齐异常、高风险、历史易漏测场景。

典型内容包括：

线上事故复盘
回归缺陷
高频漏测点
权限问题
并发问题
时间边界问题
缓存一致性问题
3. 防漏测二次反思

系统不会只依赖一次大模型生成结果，而是采用两阶段生成策略：

第一阶段：基于测试规范生成主流程和基础边界用例
第二阶段：基于历史缺陷进行二次反思和异常场景补全

二次反思会输出：

已检查的漏测点
补充策略
关联缺陷模式
缺失风险场景
审查清单
信心说明
4. 严格 JSON 输出约束

系统要求大模型最终输出标准 JSON，并使用 Pydantic 进行结构校验。

如果模型输出格式异常，会依次触发：

JSON Repair Prompt 修复
本地 Normalize 兜底
再次 Schema 校验

确保最终结果可被后续测试平台、导出工具或自动化流程消费。

5. Redis 多轮会话管理

系统使用 Redis 保存多轮会话历史，支持连续追问和上下文关联。

默认策略：

保留最近 10 轮对话
每轮包含用户消息 + 助手消息
因此最大保留 20 条消息
默认 TTL 为 3600 秒

如果 Redis 不可用，系统会自动降级为内存态，方便本地开发和容灾。

6. 流式生成支持

系统支持通过接口获取生成进度和最终结果，便于前端页面展示 Agent 执行过程。

系统架构

系统整体分为四层：

接口层
Agent 编排层
检索与会话层
生成与约束层
接口层

基于 FastAPI 对外提供接口：

测试用例生成
兼容旧查询入口
流式结果返回
会话历史查询
会话历史删除
Agent 编排层

使用 LangGraph 编排完整测试用例生成流程。

主流程包括：

node_intent_parse
↓
node_search_test_knowledge
↓
node_generate_primary_cases
↓
node_leakage_review
↓
node_json_guard
↓
node_finalize_output
检索与会话层

使用：

Milvus 存储测试规范和历史缺陷知识
BGE-M3 进行稠密 + 稀疏混合检索
Redis 管理多轮会话和 TTL
生成与约束层

使用：

LangChain ChatOpenAI
Prompt Engineering
JSON Mode
Pydantic Schema
JSON Repair Prompt
本地模板兜底
项目目录结构
app/
├── testcase_agent/
│   ├── main_graph.py
│   ├── schemas.py
│   └── nodes/
│       ├── node_intent_parse.py
│       ├── node_search_test_knowledge.py
│       ├── node_generate_primary_cases.py
│       ├── node_leakage_review.py
│       ├── node_json_guard.py
│       └── node_finalize_output.py
│
├── clients/
│   └── redis_history_utils.py
│
├── conf/
│   └── redis_config.py
│
└── query_process/
    └── api/
        └── query_service.py

prompts/
├── testcase_intent_parse.prompt
├── testcase_primary_generation.prompt
├── testcase_leakage_review.prompt
└── testcase_json_repair.prompt

test/
├── 06-testcase-agent-schema.py
└── 07-redis-history-fallback.py
核心流程说明
1. 需求意图解析

实现文件：

app/testcase_agent/nodes/node_intent_parse.py

该节点负责解析用户输入的业务需求，并提取结构化信息。

输出内容包括：

business_goal
core_entities
risk_keywords
p0_p1_focus
rewritten_requirement

如果模型调用失败，则使用本地规则进行兜底解析。

2. 测试知识检索

实现文件：

app/testcase_agent/nodes/node_search_test_knowledge.py

该节点会根据改写后的需求，同时检索两类知识：

TESTCASE_STANDARDS_COLLECTION
TESTCASE_DEFECTS_COLLECTION

如果 Milvus 不可用，则自动回退到内置知识种子。

3. 首轮测试用例生成

实现文件：

app/testcase_agent/nodes/node_generate_primary_cases.py

首轮生成主要关注：

正向主流程
核心业务校验
基础边界输入

标准测试用例字段包括：

case_id
title
objective
preconditions
test_data
steps
priority
case_type
coverage_tags
risk_source
4. 防漏测二次反思

实现文件：

app/testcase_agent/nodes/node_leakage_review.py

该节点读取首轮生成结果和历史缺陷知识，自动补齐异常场景和高风险场景。

输出内容包括：

standardized_cases
missing_risk_scenarios
review_checklist
self_reflection
5. JSON 格式校验

实现文件：

app/testcase_agent/nodes/node_json_guard.py
app/testcase_agent/schemas.py

系统使用 Pydantic 定义结构模型：

TestStep
TestCaseItem
SelfReflection
TestCaseAgentResponse

如果输出不符合 Schema，会触发 JSON 修复逻辑。

6. 多轮会话管理

实现文件：

app/clients/redis_history_utils.py
app/conf/redis_config.py

Redis 会话管理策略：

使用 Redis List 保存消息
使用 RPUSH 写入消息
使用 LTRIM 保留最近窗口
使用 EXPIRE 设置 TTL
Redis 不可用时自动降级为内存态
接口说明
1. 生成测试用例
POST /testcase/generate

请求体：

{
  "requirement": "审批流程新增撤回能力，需要输出标准化测试方案",
  "session_id": "optional-session-id",
  "is_stream": false
}

返回体：

{
  "message": "测试用例生成完成",
  "session_id": "xxx",
  "result_json": "...",
  "result_markdown": "..."
}
2. 兼容旧查询入口
POST /query

该接口兼容原有查询入口，功能与 /testcase/generate 基本一致。

3. 流式获取生成结果
GET /stream/{session_id}

用于获取指定会话的流式生成进度和最终结果。

4. 查询会话历史
GET /history/{session_id}

用于查询指定会话 ID 的历史消息。

5. 删除会话历史
DELETE /history/{session_id}

用于清空指定会话 ID 的历史记录。

环境变量配置

请在项目根目录创建 .env 文件，或通过系统环境变量进行配置。

# OpenAI / 大模型配置
OPENAI_API_KEY=
OPENAI_BASE_URL=
LLM_DEFAULT_MODEL=
LLM_DEFAULT_TEMPERATURE=0.2

# Redis 配置
REDIS_URL=redis://127.0.0.1:6379/0
REDIS_KEY_PREFIX=testcase-agent
REDIS_SESSION_TTL_SECONDS=3600
REDIS_SESSION_WINDOW_ROUNDS=10

# Milvus 配置
MILVUS_URL=
TESTCASE_STANDARDS_COLLECTION=testcase_standards
TESTCASE_DEFECTS_COLLECTION=testcase_defects
安装方式
1. 克隆项目
git clone https://github.com/your-username/testcase-agent.git
cd testcase-agent
2. 创建虚拟环境
python -m venv .venv

Linux / macOS：

source .venv/bin/activate

Windows：

.venv\Scripts\activate
3. 安装依赖
pip install -r requirements.txt

如果项目暂未提供 requirements.txt，可参考以下依赖进行安装：

pip install fastapi uvicorn langchain langgraph pydantic redis pymilvus python-dotenv

如需使用 OpenAI 兼容模型接口，可安装：

pip install openai
启动方式
1. 启动 Redis

如果本地使用 Docker：

docker run -d --name testcase-agent-redis -p 6379:6379 redis:latest

如果 Redis 已安装在本机，确保服务正常运行即可。

2. 启动 Milvus

如果需要完整启用知识检索能力，请提前启动 Milvus，并创建以下集合：

testcase_standards
testcase_defects

如果未配置 Milvus，系统会自动使用内置知识种子进行降级生成。

3. 启动 FastAPI 服务
uvicorn app.query_process.api.query_service:app --host 127.0.0.1 --port 8001

启动成功后，可以访问：

http://127.0.0.1:8001

接口文档地址：

http://127.0.0.1:8001/docs
使用示例
生成审批撤回功能测试用例

请求：

curl -X POST "http://127.0.0.1:8001/testcase/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "requirement": "审批流程新增撤回能力，需要输出标准化测试方案",
    "session_id": "demo-session-001",
    "is_stream": false
  }'

返回示例：

{
  "message": "测试用例生成完成",
  "session_id": "demo-session-001",
  "result_json": "{...}",
  "result_markdown": "## 测试方案 ..."
}
输出 JSON 示例
{
  "requirement_summary": "审批流程新增撤回能力",
  "standardized_cases": [
    {
      "case_id": "TC-001",
      "title": "审批流程中申请人成功撤回审批单",
      "objective": "验证申请人在审批未完成前可以正常撤回审批单",
      "preconditions": [
        "用户已登录",
        "存在审批中的审批单",
        "当前用户为审批单申请人"
      ],
      "test_data": {
        "approval_status": "审批中",
        "operator_role": "申请人"
      },
      "steps": [
        {
          "step_no": 1,
          "action": "进入审批详情页",
          "expected_result": "页面展示撤回按钮"
        },
        {
          "step_no": 2,
          "action": "点击撤回按钮并确认",
          "expected_result": "审批单撤回成功，状态变为已撤回"
        }
      ],
      "priority": "P0",
      "case_type": "positive",
      "coverage_tags": [
        "主流程",
        "状态流转"
      ],
      "risk_source": "测试规范"
    }
  ],
  "missing_risk_scenarios": [
    "重复撤回",
    "审批完成后撤回",
    "无权限用户撤回他人审批单"
  ],
  "review_checklist": [
    "是否覆盖主流程",
    "是否覆盖权限校验",
    "是否覆盖状态流转",
    "是否覆盖历史缺陷相似场景"
  ],
  "self_reflection": {
    "checked_risks": [
      "权限绕过",
      "重复操作",
      "状态一致性"
    ],
    "supplement_strategy": "基于历史缺陷补充异常和边界场景",
    "related_defect_patterns": [
      "审批状态未回滚",
      "重复提交导致状态异常"
    ],
    "confidence": "medium"
  }
}
测试脚本
1. JSON Schema 校验测试
python test/06-testcase-agent-schema.py

该脚本用于验证测试用例输出是否符合 Pydantic Schema。

2. Redis 会话降级测试
python test/07-redis-history-fallback.py

该脚本用于验证 Redis 不可用时，系统是否能够自动降级为内存态。

推荐知识库数据

为了让 Agent 具备更强的防漏测能力，建议向 Milvus 中沉淀以下数据。

测试规范类
测试用例设计规范
P0 / P1 分级标准
边界值设计规则
异常值设计规则
权限测试清单
幂等性测试清单
兼容性测试规范
回归测试规范
历史缺陷类
线上事故复盘
回归缺陷记录
高频漏测点
权限绕过问题
并发一致性问题
时间边界问题
缓存一致性问题
状态流转异常问题
项目亮点
1. Agent 化测试设计流程

系统不是一次性调用大模型，而是将测试设计流程拆分为多个可编排节点：

需求解析 → 知识检索 → 首轮生成 → 二次反思 → JSON 校验
2. 历史缺陷驱动防漏测

通过引入历史缺陷库，系统可以基于真实发生过的问题补充异常场景，提高高风险场景覆盖率。

3. 标准化结构输出

所有测试用例都按照固定字段输出，方便后续接入测试管理平台、导出 Excel、生成 XMind 或进行自动化处理。

4. 多级容错机制

系统内置多种 fallback：

意图解析失败时回退本地规则
Milvus 不可用时回退内置知识种子
模型生成失败时回退模板用例
JSON 校验失败时触发修复 Prompt
Redis 不可用时回退内存态
5. 支持多轮上下文

基于 Redis 保存会话历史，使用户可以连续追问、补充测试范围、调整优先级或导出格式。

后续规划

后续可继续增强以下能力：

支持测试用例导出为 Excel
支持测试用例导出为 XMind
接入 Jira、禅道、Tapd 等缺陷管理系统
自动识别需求变更影响范围
自动推荐回归测试范围
增加支付、审批、风控、库存、账号权限等领域插件
增加风险评分器，根据业务关键度和历史缺陷密度调整 P0 / P1 用例比例
与测试管理平台打通，实现自动入库和执行状态追踪
适用场景

本项目适用于以下场景：

需求评审后快速生成测试方案
测试工程师编写标准化测试用例
历史缺陷驱动的回归测试补充
高风险业务场景测试设计
多轮交互式测试方案完善
测试平台自动化接入前的结构化数据生成
技术栈
技术	作用
FastAPI	提供 HTTP API 服务
LangGraph	编排 Agent 工作流
LangChain	调用大模型与构建链路
OpenAI Compatible API	大模型生成能力
Milvus	向量知识库
BGE-M3	稠密 + 稀疏混合向量检索
Redis	多轮会话管理
Pydantic	JSON Schema 校验
JSON Mode	约束模型结构化输出
Prompt Engineering	控制不同阶段模型行为
常见问题
1. 没有 Milvus 可以运行吗？

可以。

如果未配置 Milvus，系统会自动回退到内置知识种子，仍然可以生成测试用例。但推荐在生产环境中接入 Milvus，以获得更强的企业知识检索和历史缺陷补全能力。

2. 没有 Redis 可以运行吗？

可以。

如果 Redis 不可用，系统会自动降级为内存态。但生产环境建议使用 Redis，避免服务重启后会话丢失。

3. 为什么要做二次反思？

因为一次性生成测试用例容易遗漏异常、边界和历史高风险场景。

本项目采用：

首轮主流程生成 + 历史缺陷反向补漏

让测试用例覆盖更接近真实测试工程师的设计过程。

4. 为什么要强制 JSON 输出？

因为测试用例最终需要被平台消费、导出或落库。

自然语言输出虽然可读，但不稳定，不利于自动化处理。JSON 结构可以保证字段统一、格式可校验、后续易扩展。

License

本项目仅用于学习、研究和内部测试效率提升场景。

如需商用，请根据实际情况补充开源协议或企业内部使用说明。
