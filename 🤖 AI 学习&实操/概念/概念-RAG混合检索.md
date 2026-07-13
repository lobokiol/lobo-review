# 概念 · RAG 混合检索

> 考点：为什么要 hybrid、RRF、rerank、俗语归一、否定症状、幻觉控制。

## 一句话

向量解决「像不像」，关键词（ES/BM25）解决「是不是」；RRF 融合后再用 rerank 压到 topK，最终才交给 LLM。

## 标准答（可背）

- 召回：向量 top50（BGE）+ ES/BM25 top50 → **RRF** 融合  
- 精排：BGE reranker → top20 → top5 → 入 LLM  
- 失败兜底：embedding 挂了回退纯关键词；无证据则拒答，不硬编  

## 链接到证据

- [[面试复盘]]（1STEP.AI / 零食有鸣 / RAG 专题段）
- [[面试讲解稿]]（OpenSearch BM25 + kNN）
- [[100个面试问题]]（RAG 优化章节）
- [[简历]]（Hybrid RAG / Harness）
- [[TodoList导诊]]（hybrid pipeline 缺口）

## 相关概念

- [[概念-知识图谱Neo4j]]（俗语→术语归一）
- [[概念-Golden评测]]（Recall@5 / 幻觉判定）
- [[概念-LangGraph状态机]]（rag 节点与拒答边）

#概念 #RAG #面试
