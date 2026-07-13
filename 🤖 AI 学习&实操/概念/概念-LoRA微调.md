# 概念 · LoRA 微调

> 考点：为何微调、数据怎么构、指标看什么、和 Prompt 的边界。

## 一句话

Prompt 控不住的结构化输出 / 领域映射 / 硬边界，用小样本 LoRA 把 Schema 和行为钉死；推理时 Adapter + 置信度兜底。

## 标准答（可背）

- 典型任务：意图五分类、槽位 JSON、俗语→术语、急诊边界样本  
- 流程：真实对话清洗 → 模板增强 → 8:2 → r=16 / lr=5e-5 / 3 epoch → 导出 Adapter  
- 指标：整体准确率、**emergency 召回**、Frame Accuracy；错误多在 hard case  
- 不是替代 RAG：生成仍要检索证据；LoRA 主要稳住「入口理解」  

## 链接到证据

- [[面试复盘]]（LoRA 流程 / 零食有鸣 / 南昌万宜）
- [[100个面试问题]]（微调章节）
- [[Lora基本流程车险理赔]]（多模态+规则+回流训练的企业流程）
- [[深度学习本质与 Qwen3-8B 运行链路深度解析]]（模型侧基础）
- [[BoLiu_AI_Agent_Engineer_Resume]]（Medical Intent LoRA）

## 相关概念

- [[概念-Golden评测]]
- [[概念-RAG混合检索]]

#概念 #LoRA #微调 #面试
