# 概念 · MCP 工具调用

> 考点：工具为何独立、Schema、权限、失败恢复、和 Agent 节点的关系。

## 一句话

把医院/业务能力做成 MCP Tool（值班、介绍、路线、KG 查询），Agent 只做意图路由与参数组装，工具侧可控、可测、可降级。

## 标准答（可背）

- 导诊：Stdio MCP → `get_oncall` / 科室介绍 / 路线；KG 也可经 MCP 调 Neo4j  
- 图纸：`extract_order_params` / `parse_drawing` / `compare_params` —— Schema 比「万能函数」重要  
- 原则：结构化返回、只读权限、超时重试、幂等、失败转人工  

## 链接到证据

- [[面试复盘]]（MCP 调 Neo4j；Tools 列表段）
- [[面试讲解稿]]（mcp_followup）
- [[简历]]（MCP 集成）
- [[零部件图纸审核智能体-项目文档]]
- [[智能体工程师-AI工程师岗位要求]]（工具维度）

## 相关概念

- [[概念-LangGraph状态机]]
- [[概念-知识图谱Neo4j]]

#概念 #MCP #工具调用 #面试
