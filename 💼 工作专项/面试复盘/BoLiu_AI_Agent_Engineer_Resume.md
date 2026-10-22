# Bo Liu

**Phone:** +86 18124537262
**Email:** [liukiol2020@gmail.com](mailto:13764743157@163.com)
**Experience:** 12 Years
**Target Roles:** AI Agent Engineer / AI Application Engineer / LLM Engineer
**Expected Salary:** Negotiable
**Education:** Bachelor’s Degree

------

## Professional Summary

Experienced engineer specializing in AI Agent systems and LLM-based applications, with end-to-end delivery capability from requirement analysis to production deployment. Proven expertise in RAG pipelines, knowledge graphs, multi-agent orchestration, and LLM optimization. Strong background in building scalable, production-grade systems with measurable performance improvements.

------

## Core Skills

- **AI Stack:** FastAPI, RAG, Agent, LangGraph, Knowledge Graph (Neo4j)
- **Retrieval Systems:** Elasticsearch, Milvus, Hybrid Retrieval (RRF + Rerank)
- **LLM Engineering:** Qwen2.5, LoRA Fine-tuning, Function Calling, Tool Integration
- **Data Processing:** PDF Parsing, OCR, Structured Extraction, Embedding
- **Infrastructure:** Docker Compose, vLLM, GPU Deployment
- **Testing & QA:** Automation Frameworks, Performance Testing, Fault Analysis

------

## GitHub

- https://github.com/lobokiol/InitialArchitectureForMedicalAgent1

------

## Work Experience

### Chinasoft International Technology Co., Ltd.

**AI Agent Engineer**
Jul 2024 – Present

#### Project 1: Intelligent Medical Triage Agent System

**Overview**
Developed a hospital triage system based on RAG, Agent architecture, Knowledge Graph, and hybrid retrieval. The system supports multi-turn diagnosis, symptom extraction, and department recommendation.

**Key Responsibilities**

- Designed a multi-turn dialogue state machine using LangGraph, supporting intent recognition, slot filling, dynamic questioning, and answer validation
- Implemented knowledge graph reasoning pipeline: symptom → disease → department using Jaccard similarity and score aggregation
- Built hybrid retrieval system combining Elasticsearch and Milvus, with RRF fusion and reranking
- Developed LLM-based modules for symptom extraction, semantic normalization, and response generation using Qwen2.5
- Designed risk control mechanisms including negation detection and emergency rules (100+ rules)
- Implemented confidence evaluation combining KG and RAG scores with threshold-based decision logic
- Built PDF ingestion pipeline: parsing, OCR, structured extraction, chunking, and validation
- Deployed system locally using Docker Compose, vLLM, and GPU-based inference
- Designed modular Skills layer including symptom processing, KG reasoning, RAG retrieval, and follow-up generation
- Led multi-modal extension (image + text) using structured engineering workflow (SPEC → PLAN → BUILD → TEST → REVIEW → SHIP)

**Tech Stack**
LangGraph, Neo4j, Elasticsearch, Milvus, Qwen2.5 7B/72B, bge-large-zh, vLLM, FastAPI

**Achievements**

- Covered ~80% of common triage scenarios, reducing manual workload
- Improved retrieval accuracy by 15%–20%, with Top-5 recall ~75%
- Controlled average dialogue turns to 3–5
- Completed 10,000+ automated test interactions, with <5% system error rate

------

#### Project 2: Medical Intent Classification (LoRA Fine-tuning)

**Overview**
Developed a high-accuracy intent classification module for triage entry, supporting structured JSON output.

**Key Responsibilities**

- Designed classification schema: symptom / process / mixed / non-medical / emergency
- Fine-tuned Qwen2.5-7B using LoRA for structured extraction
- Built training dataset with 2,000 labeled samples including hard cases
- Configured training pipeline on L40S GPU

**Training Configuration**

- LoRA: r=16, alpha=32
- Target modules: full projection layers
- Learning rate: 1e-4
- Batch size: 16
- Epochs: 3

**Achievements**

- Reduced GPU memory usage by >70% using LoRA
- Achieved >95% classification accuracy
- Improved robustness on negation and implicit expressions by ~30%

------

### Chinasoft International Technology Co., Ltd.

**Software Test Engineer (Cloud Storage SaaS)**
Oct 2022 – Jun 2024

- Rebuilt core automation framework, improving stability by ~60%
- Designed failure analysis and one-click retry system
- Led P0/P1 defect resolution and quality assurance processes

------

### Kr Network Technology Co., Ltd.

**Software Test Engineer**
Jun 2017 – Jun 2022

- Developed automated testing systems using Python, Selenium, Pytest
- Conducted performance testing using JMeter and Locust
- Covered functional and API testing

------

### Shanghai Dasheng Mould Co., Ltd.

**Electrical Engineer**
Jul 2014 – Jun 2017

- Developed PLC programs and HMI interfaces for industrial automation
- Designed communication protocols between control systems
- Performed system debugging, fault diagnosis, and optimization

------

## Education

**Yuncheng University**
Bachelor’s Degree in Electronic Science and Technology
2010 – 2014

------

## Additional Notes

- Strong experience in production-grade AI systems
- Capable of independent system architecture and delivery
- Focus on reliability, scalability, and cost optimization