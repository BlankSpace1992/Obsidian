---
title: RAG 落地实战：Spring AI + Milvus 企业级智能问答系统
author: 程序员波特
source: https://mp.weixin.qq.com/s/bYenweqqPDy5qHP9s8kA_A
date: 2025-05-14
tags:
  - RAG
  - Spring-AI
  - Milvus
  - 向量数据库
  - Java
  - 智能问答
  - 大模型
---

# RAG 落地实战：Spring AI + Milvus 企业级智能问答系统

> 原文链接：[程序员波特](https://mp.weixin.qq.com/s/bYenweqqPDy5qHP9s8kA_A)

---

## 一、为什么 RAG 是企业 AI 落地的必选方案？

2025 年是 RAG 技术爆发的一年，超过 70% 的企业 AI 应用都采用了 RAG 架构。

大模型的三个致命问题：
- **知识滞后**：训练数据有截止日期，不知道你公司的新产品信息
- **幻觉问题**：一本正经地胡说八道
- **数据安全**：把公司机密数据喂给公有大模型？老板不同意

### RAG 核心价值

**RAG（Retrieval-Augmented Generation，检索增强生成）** 不是重新训练模型，而是给大模型外挂一个"企业专属图书馆"。

**工作原理三步走：**
1. 🔍 **检索（Retrieval）**：从企业知识库中找到最相关的文档片段
2. ⚡ **增强（Augmented）**：把检索到的知识和用户问题组成高质量 Prompt
3. ✨ **生成（Generation）**：大模型基于真实知识生成准确、可信的答案

### 企业级应用场景

| 应用场景 | 业务价值 | 典型客户 |
|---------|---------|---------|
| 智能客服 | 响应效率提升 80%，准确率达 95%+ | 电商、金融、运营商 |
| 内部知识库 | 查询效率提升 3 倍，培训周期减半 | 中大型企业、政府机构 |
| 文档问答 | 合同、财报、技术文档秒级解读 | 法律、咨询、研发部门 |
| 产品助手 | 7x24 小时咨询，转化率提升 40% | SaaS、硬件厂商 |

---

## 二、Spring AI 框架解析

**Spring AI** 是 Pivotal 官方推出的 AI 应用开发框架，专为 Java 开发者设计，用熟悉的 Spring 模型快速构建 AI 应用。

> 核心理念：**抽象而非实现** — 一套 API 兼容所有大模型，切换 OpenAI / 通义千问 / 文心一言只需改配置！

### Spring AI 核心架构

```
┌─────────────────────────────────────────────────────────┐
│                    Spring AI 抽象层                      │
├─────────────┬─────────────┬─────────────┬───────────────┤
│  ChatClient │  Embedding   │ VectorStore │   Document   │
│   接口       │   Client    │   接口      │   处理器       │
├─────────────┼─────────────┼─────────────┼───────────────┤
│  OpenAI     │  OpenAI     │  Milvus     │   PDF/Word    │
│  通义千问    │  通义千问   │  Pinecone   │   Excel/HTML   │
│  文心一言    │  文心一言   │  Chroma     │   Markdown    │
│  Ollama     │  Ollama     │  PGVector   │   纯文本      │
└─────────────┴─────────────┴─────────────┴───────────────┘
```
### Spring AI vs LangChain4j

| 特性 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 官方支持 | Spring 官方出品，长期维护 | 社区驱动 |
| 学习曲线 | 极低，Spring 开发者零成本 | 中等 |
| 生态集成 | 完美集成 Spring Boot | 需自行适配 |
| 生产就绪 | 1.0 GA 已发布 | 已稳定 |

> 实战建议：Java 后端团队优先选择 Spring AI，开发效率提升 50% 以上！

---

## 三、环境准备：阿里云百炼 API

### 3.1 注册 & 开通

1. 注册阿里云账号：[https://account.aliyun.com](https://account.aliyun.com)
2. 开通百炼服务：[百炼控制台](https://bailian.console.aliyun.com)
3. 获取 API Key：百炼控制台 → 密钥管理 → 创建 API Key

### 3.2 模型选型建议

| 模型 | 适用场景 | 价格（千 tokens） | 推荐 |
|------|---------|-------------------|------|
| qwen-plus | 日常问答、RAG 检索 | 输入 ¥0.002，输出 ¥0.006 | ⭐⭐⭐⭐⭐ |
| qwen-max | 复杂推理、长文档 | 输入 ¥0.02，输出 ¥0.06 | ⭐⭐⭐⭐ |
| qwen-turbo | 简单问答、高并发 | 输入 ¥0.0005，输出 ¥0.002 | ⭐⭐⭐ |
| text-embedding-v4 | 向量生成 | ¥0.0005 / 千 tokens | ⭐⭐⭐⭐⭐ |

> 💡 90% 的 RAG 场景用 `qwen-plus` + `text-embedding-v4` 就够了，性价比最高！

---

## 四、向量数据库：RAG 的核心引擎

### 什么是向量数据库？

向量数据库存的是**语义向量**——把文本、图片、音频都变成高维空间中的点，语义相近的数据距离更近。

核心概念：
- **向量嵌入（Embedding）**：把"我想买手机"变成 1024 维浮点数数组
- **相似度搜索**：找到向量空间中距离最近的 Top-K 个文档
- **ANN 算法**：近似最近邻搜索，毫秒级从百万数据中找到结果

### 主流向量数据库选型

| 数据库 | 部署方式 | 推荐场景 |
|--------|---------|---------|
| **Milvus** | 独立部署 / Docker | 中大型企业生产环境 |
| Qdrant | 独立部署 / Docker | 中小团队快速落地 |
| PGVector | PostgreSQL 扩展 | 已有 PG 栈 |
| Chroma | 嵌入式 | 本地开发、原型验证 |

> 💡 生产环境首选 **Milvus**，云原生架构支持水平扩展，单集群可支持百亿级向量！

### Milvus 部署（Docker）

```bash
# 参考：https://milvus.io/docs/install_standalone-docker.md
# 部署完成后访问 UI：http://127.0.0.1:9091/webui/
```

---

## 五、工程搭建

### 5.1 核心依赖（pom.xml）

```xml
<properties>
    <spring-ai.version>1.0.0-M1</spring-ai.version>
</properties>

<!-- Spring Boot Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring AI Ollama -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>

<!-- Milvus Java SDK -->
<dependency>
    <groupId>io.milvus</groupId>
    <artifactId>milvus-sdk-java</artifactId>
    <version>2.4.0</version>
</dependency>

<!-- PDF 解析 -->
<dependency>
    <groupId>org.apache.pdfbox</groupId>
    <artifactId>pdfbox</artifactId>
    <version>2.0.30</version>
</dependency>
```

### 5.2 配置文件

```yaml
server:
  port: 8080

milvus:
  uri: http://localhost:19530
  token: root:Milvus
  collection-name: pdf_rag_collection
  vector-dim: 1024  # Embedding 向量维度
```

### 5.3 PDF 读取工具类

```java
public class PdfReaderUtil {
    public static String readPdfToString(String pdfPath) throws IOException {
        try (PDDocument document = PDDocument.load(new File(pdfPath))) {
            PDFTextStripper stripper = new PDFTextStripper();
            stripper.setSortByPosition(true);
            return stripper.getText(document).trim();
        }
    }
}
```

### 5.4 文本分块工具类

```java
public class TextSplitterUtil {
    /**
     * 文本分块（固定长度，无重叠）
     * @param chunkSize 每块大小，默认 512 字符
     */
    public static List<String> splitText(String text, int chunkSize) {
        List<String> chunks = new ArrayList<>();
        int start = 0;
        while (start < text.length()) {
            int end = Math.min(start + chunkSize, text.length());
            chunks.add(text.substring(start, end).trim());
            start = end;
        }
        return chunks;
    }
}
```

### 5.5 千问 API 工具类

```java
@Component
public class QianWenUtil {
    private static final String API_KEY = "xxx";
    private static final String BASE_URL = "https://dashscope.aliyuncs.com/compatible-mode/v1";
    private static final String LLM_MODEL = "qwen-plus";

    OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey(API_KEY)
            .baseUrl(BASE_URL)
            .build();

    // 文本向量化
    public List<Float> textToEmbedding(String text) {
        EmbeddingCreateParams params = EmbeddingCreateParams.builder()
                .model("text-embedding-v4")
                .input(EmbeddingCreateParams.Input.ofString(text))
                .dimensions(vectorDim)
                .build();
        CreateEmbeddingResponse response = client.embeddings().create(params);
        // ... 提取向量
    }

    // 大模型问答
    public String llmChat(String prompt) {
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
                .addSystemMessage(SYSTEM_PROMPT)
                .addUserMessage(prompt)
                .model(LLM_MODEL)
                .build();
        ChatCompletion chatCompletion = client.chat().completions().create(params);
        return chatCompletion.choices().get(0).message().content().orElse("");
    }
}
```

### 5.6 Milvus 操作类

```java
@Component
public class MilvusUtil {
    private MilvusClientV2 milvusClient;

    @PostConstruct
    public void init() {
        ConnectConfig config = ConnectConfig.builder()
                .uri(uri).token(token).build();
        milvusClient = new MilvusClientV2(config);
        createCollection();
    }

    // 创建集合：id(主键) + text(文本) + vector(向量)
    private void createCollection() { /* ... */ }

    // 插入文本+向量
    public void insertData(String text, List<Float> vector) { /* ... */ }

    // 相似度检索（返回最相似的 3 条文本）
    public List<String> searchSimilarText(List<Float> queryVector) {
        SearchReq searchReq = SearchReq.builder()
                .collectionName(collectionName)
                .data(Collections.singletonList(queryVector))
                .annsField("vector")
                .topK(3)
                .outputFields(Collections.singletonList("text"))
                .build();
        // ... 搜索并返回结果
    }
}
```

### 5.7 RAG 核心服务

```java
@Service
@RequiredArgsConstructor
public class RagService {
    private final QianWenUtil qianWenUtil;
    private final MilvusUtil milvusUtil;

    // 上传 PDF 并向量化入库
    public void uploadPdfAndInitData(String pdfPath) throws Exception {
        String pdfText = PdfReaderUtil.readPdfToString(pdfPath);       // 1. 读取 PDF
        List<String> chunks = TextSplitterUtil.splitText(pdfText);     // 2. 文本分块
        for (String chunk : chunks) {
            List<Float> vector = qianWenUtil.textToEmbedding(chunk);   // 3. 向量化
            milvusUtil.insertData(chunk, vector);                      // 4. 存入 Milvus
        }
    }

    // RAG 问答核心逻辑
    public String ragChat(String question) throws Exception {
        List<Float> queryVector = qianWenUtil.textToEmbedding(question);     // 1. 问题向量化
        List<String> similarTexts = milvusUtil.searchSimilarText(queryVector); // 2. 检索相似文本
        String prompt = "基于以下文档内容回答问题，不要编造答案：\n"
                + "文档内容：" + String.join("\n", similarTexts)
                + "\n用户问题：" + question;                                   // 3. 拼接 Prompt
        return qianWenUtil.llmChat(prompt);                                    // 4. 调用大模型
    }
}
```

### 5.8 Controller

```java
@RestController
@RequestMapping("/api/rag")
@RequiredArgsConstructor
public class RagController {
    private final RagService ragService;

    // 初始化 PDF 数据
    @GetMapping("/import")
    public String importDocument(@RequestParam("path") String pdfPath) {
        ragService.uploadPdfAndInitData(pdfPath);
        return "PDF 初始化完成，文本已向量化存入 Milvus！";
    }

    // RAG 问答
    @RequestMapping(value = "/chat", method = {RequestMethod.GET, RequestMethod.POST})
    public Map<String, String> chat(@RequestBody Map<String, String> request) {
        String answer = ragService.ragChat(request.get("question"));
        return Map.of("answer", answer);
    }
}
```

---

## 六、RAG 流程总结

```
PDF 文档 → 文本提取 → 分块(512字) → 向量化(Embedding) → 存入 Milvus
                                                          ↓
用户提问 → 问题向量化 → Milvus 相似度检索(Top-3) → 拼接 Prompt → 大模型生成回答
```

---

## 七、代码地址

源码已开源，包含完整前后端实现。
