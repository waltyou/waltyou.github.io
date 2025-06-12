---
layout: post
title: Build a production-ready RAG system
date: 2025-06-12 13:39:04
author: admin
comments: true
categories: [GenAI]
tags: [RAG, Milvus, Elasticsearch, langchain]
---

简单介绍一个结合 **Milvus**（向量数据库）、**Elasticsearch**（全文检索/BM25），并通过 **LangChain** 构建的 Retrieval 方案。

<!-- more -->

---

* Outline
{:toc}
---

# 0. RAG 最佳实践、关键见解和流行工具/库 in 2025

| 方面                     | 最佳实践要点                                                                                              | 关键见解                                                                                                                  | 流行工具 / 库                                                                                                      |
|--------------------------|---------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| 数据准备 (Data Preparation) | 自动化 pipelines、过滤、分段、metadata 提取、持续更新                                                   | 数据质量和新鲜度至关重要；将文档分割成可管理的小块；自动化更新避免信息陈旧                                               | RAGFlow（深度文档理解、PDF parsing）、LangChain（pipeline 构建）、自定义 ETL 工具                                |
| 向量数据库 (Vector Databases) | 使用 semantic embeddings、高维 vector search 实现精准检索                                              | 向量数据库存储语义嵌入，实现超越关键词匹配的 semantic search，是可扩展 RAG 的关键组件                                     | Pinecone、Chroma、Weaviate、FAISS、Milvus                                                                         |
| 评估 (Evaluation)          | 多指标评估、human-in-the-loop、持续 fine-tuning                                                        | 结合 precision/recall 与人工反馈；监控 hallucination；迭代优化平衡检索相关性和生成质量                                   | OpenAI Evals（定制 LLM 评估）、Humanloop、Hugging Face Evaluate                                                  |
| 检索技术 (Retrieval Techniques) | 混合检索（keyword + semantic + graph）、adaptive retrieval                                            | 结合 vector retrieval（embedding）与 BM25 keyword search，实现精准且语义覆盖广泛；rank fusion 提升效果                  | Elasticsearch（BM25）、Hugging Face Transformers（重排序模型如 bge-ranker-base）、RAGFlow、LightRAG               |
| 多模态检索 (Multimodal Retrieval) | 融合 images、videos、audio、structured data                                                          | 利用 multimodal embeddings 统一不同数据类型，实现更丰富的 context 理解和更优决策                                         | RAGFlow（支持 GraphRAG）、Hugging Face Transformers



# 1. 系统架构与原理

### **目标**
- **兼顾关键词检索（BM25）与语义检索（Embedding/向量）**，兼容不同用户提问方式，召回率和相关性俱佳。
- **高效扩展**，支持海量文档的低延迟检索。

### **核心组件**
- **Elasticsearch**：倒排索引，支持BM25/TF-IDF关键词检索、结构化过滤，适合精确查询。
- **Milvus**：高性能向量数据库，支持海量向量的ANN检索，适合语义相似度召回。
- **LangChain**：作为Orchestrator，整合两种检索结果，并与LLM/RAG链集成。

---

# 2. 检索流程方案

## **方案A：级联检索（Cascade Retrieval，推荐）**

1. **第一步：Elasticsearch 预筛（BM25）**
   - 用关键词查询在ES召回Top N（比如500~2000，N可调）。
   - 适合短关键词、精确提问、性能瓶颈前移。

2. **第二步：Milvus 精检（Vector Search）**
   - 对Elasticsearch召回的Top N文档做embedding，送入Milvus做相似度Top K（比如K=10）。
   - 只需对小批量文档做embedding和向量检索，节省算力，召回更相关内容。

3. **第三步：结果融合与排序**
   - 可以根据BM25分数和向量分数加权重融合。
   - 也可根据业务场景优先某一方向。

## **方案B：混合检索（Hybrid Retrieval）**

- 并行调用ES和Milvus，各自召回Top N，然后在业务层或LangChain用重排序策略（如加权、去重、排序）融合。
- 推荐用LangChain的`EnsembleRetriever`或自定义融合逻辑。

---

# 3. LangChain 集成实现要点

## **a. ES Retriever**

LangChain 已内置 `ElasticsearchRetriever`，直接用即可。

## **b. Milvus Retriever**

LangChain 已内置 `MilvusRetriever`，需要先将文档embedding写入Milvus。

## **c. 组合检索器（混合/级联）**

- 可用 `EnsembleRetriever`，或自定义一个“级联Retriever”。
- 级联实现要点：先用ES检索，取Top N文档内容做embedding，批量查Milvus。

---

# 4. 伪代码/示例

```python
from langchain.retrievers import ElasticsearchRetriever, MilvusRetriever, EnsembleRetriever
from langchain.embeddings import HuggingFaceEmbeddings

# ES配置
es_retriever = ElasticsearchRetriever(
    es_url="http://localhost:9200",
    index_name="your-index",
    search_type="bm25"
)

# 向量配置
embedding_model = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")
milvus_retriever = MilvusRetriever(
    embedding=embedding_model,
    collection_name="your_collection",
    connection_args={"host": "localhost", "port": "19530"}
)

# 混合检索（并行召回+融合）
hybrid_retriever = EnsembleRetriever(
    retrievers=[es_retriever, milvus_retriever],
    weights=[0.5, 0.5]  # 可调
)

# 级联检索（自定义逻辑）
def cascade_retrieval(query):
    # 1. ES召回Top N
    es_results = es_retriever.get_relevant_documents(query, top_n=500)
    # 2. 拿到文本内容批量做embedding
    docs_texts = [doc.page_content for doc in es_results]
    doc_embeddings = embedding_model.embed_documents(docs_texts)
    # 3. 批量查Milvus
    milvus_results = milvus_retriever.similarity_search_by_vector_batch(doc_embeddings, top_k=10)
    # 4. 结果融合
    # ...自定义排序逻辑...
    return milvus_results

# QA链集成
from langchain.chains import RetrievalQA
from langchain.llms import OpenAI

llm = OpenAI()
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=hybrid_retriever  # 或自定义cascade_retrieval
)
```

---

# 5. 工程建议与注意事项

- **ES与Milvus均为分布式，支持横向扩展，适合大规模数据。**
- **级联检索对性能友好，embedding只对少量候选做即可。**
- **不要把所有文本embedding全存内存，Milvus负责管理向量。**
- **LangChain支持自定义Retriever，方便扩展和重排序。**
- **如果业务问句多为语义类，Milvus权重可调高；如多为精确查找，ES权重调高。**
- **要保证ES和Milvus索引/collection一致性。**

---

# 6. 可选优化

- ES召回结果带BM25分数，Milvus带相似度分数，可规范化后融合排序。
- 支持多字段（如标签、分类、时间等）filter过滤。
- 向量写入Milvus和ES建索引可异步分批进行。

---

# 7. 相关参考

- [LangChain官方：Elasticsearch Retriever](https://python.langchain.com/docs/integrations/retrievers/elasticsearch)
- [LangChain官方：Milvus Retriever](https://python.langchain.com/docs/integrations/vectorstores/milvus)
- [LangChain官方：混合检索](https://python.langchain.com/docs/modules/data_connection/retrievers/ensemble)
- [Elasticsearch Hybrid Search官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html)
- [Milvus官方文档](https://milvus.io/docs)

---