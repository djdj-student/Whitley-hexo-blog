---
title: "Building a PDF RAG Bot with Docker and DeepSeek: A Full Development Lifecycle"
date: 2026-03-15 08:40:00
tags: [RAG, Docker, DeepSeek, PDF_Analysis]
categories: [AI_Engineering]
mathjax: false
comments: true
top_img: /img/rag-pdf-bg.jpg
---

## 1. Background & Environment Setup

### 1.1 Why RAG + PDF Research Papers?
In the AI era, while LLMs are powerful, they still suffer from "hallucinations" when dealing with specific or real-time knowledge—such as the latest scientific papers. **RAG (Retrieval-Augmented Generation)** acts as a dynamic, real-time knowledge base attached to the AI.

The core challenge of this experiment lies in **PDF Parsing**. Research papers often feature double-column layouts and complex formulas, requiring a system that doesn't just "read" text, but truly "understands" the document structure.

### 1.2 Core Architecture: Docker + DeepSeek
To achieve high performance, low cost, and high portability, I chose a lightweight, deployable, and full-stack architecture:

- **DeepSeek-R1**: The foundational LLM for reasoning and generation.
- **LangChain**: Orchestrating the RAG pipeline (Loading → Splitting → Embedding → Retrieval → Generation).
- **Sentence-Transformers**: Local open-source embedding models.
- **FAISS**: A lightweight vector database.
- **Streamlit**: For rapid Web interface development.
- **Docker + Docker Compose**: Containerizing the entire environment.

### 1.3 Containerized Deployment in Action
I developed a streamlined `docker-compose.yml` to drive the system, focusing on **development efficiency** and **security isolation**:

```yaml
services:
  app:
    build: .
    # Port mapping: Access Streamlit via localhost:7887
    ports:
      - "7887:8501" 
    
    # Security: Manage DeepSeek API Key via env file
    env_file:
      - .env
    
    # Efficiency: Mount current directory for "hot-reloading"
    volumes:
      - .:/app
    
    # Interactive mode: For real-time debugging of PDF parsing errors
    stdin_open: true
    tty: true
```

## 2. PDF Parsing & Knowledge Base Construction
This section is the core essence of RAG. The goal: How to make DeepSeek "read" the static text trapped in PDFs?

### 2.1 The "Deep Water" of PDF Parsing
Scientific papers are rarely simple text. Their double-column layouts and complex formatting mean that a "brute-force" text extraction will often scramble the semantic meaning.

### 2.2 Practice: Extracting Content with Python
In this experiment, I used PyMuPDF (fitz). It’s incredibly fast and does a great job of preserving the document's logical structure.
import fitz  # PyMuPDF library

```python
def extract_text_from_pdf(pdf_path):
    # Open the PDF file
    doc = fitz.open(pdf_path)
    full_text = ""
    
    for page in doc:
        # Extract text from the current page
        full_text += page.get_text()
        
    return full_text
```

> Pro Tip: For double-column papers, consider using page.get_text("blocks"). This reads content by physical blocks, preventing the text from left and right columns from getting mixed up.
> 
## 3. LLM Integration: Connecting the DeepSeek Engine
With structured text blocks ready, the next step is building the bridge to DeepSeek-R1. Here, we don't just send a query; we feed the "retrieved PDF snippets" as context to the model.

### 3.1 Why DeepSeek-R1?
DeepSeek’s Chain of Thought (CoT) capability is exceptional for complex academic papers. It identifies logical correlations within experimental data rather than just repeating words.

### 3.2 Implementation Python
Here is the core function running inside the container, utilizing the API Key configured in our .env file:

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"), 
    base_url="[https://api.deepseek.com](https://api.deepseek.com)"
)

def ask_deepseek(context, query):
    # Constructing the Prompt: Forcing the model to answer based on Context
    prompt = f"Answer the question based on the following PDF content:\n{context}\n\nQuestion: {query}"
    
    response = client.chat.completions.create(
        model="deepseek-chat",
        messages=[
            {"role": "system", "content": "You are a professional academic assistant. Provide accurate answers based on the provided document."},
            {"role": "user", "content": prompt},
        ],
        stream=False
    )
    return response.choices[0].message.content
```

### 3.3 Prompt Engineering: Avoiding Hallucinations
A common issue in RAG is the AI "making things up" beyond the document. To prevent this, I added a strict constraint to the System Prompt.
Core Strategy: "If the information is not present in the document, simply answer 'Sorry, this content is not mentioned in the document.' Strictly avoid hallucinations."

## 4. Hard-earned Lessons: Networking & Versioning Issues
The biggest challenge wasn't the algorithm, but the "war" against networking issues and dependency conflicts. Here are the essential fixes:

### 4.1 Pip Timeouts & Mirror Acceleration
The Pain Point: pip install in Docker often triggers ReadTimeoutError due to network instability, causing the build to fail.
The Fix: Use a local mirror (like Tsinghua) and increase timeout settings in your Dockerfile:
Accelerate build with a mirror and disable cache for stability

```bash
RUN pip install --no-cache-dir \
    -i [https://pypi.tuna.tsinghua.edu.cn/simple](https://pypi.tuna.tsinghua.edu.cn/simple) \
    -r requirements.txt
```

### 4.2 LangChain "Version Hell"
The Pain Point: LangChain evolves rapidly, and APIs are often not backward compatible. A simple pip install langchain can break your code with ImportError.
The Fix: Lock your versions strictly in requirements.txt. Here is my "Golden Combination":

```text
langchain==0.1.12
langchain-community==0.0.28
langchain-openai==0.0.8
python-dotenv==1.0.1
PyMuPDF==1.23.26
```

### 4.3 Missing System Dependencies
The Pain Point: Libraries like fitz won't run on lightweight images (like python:3.9-slim) because they lack underlying GUI/graphics libraries.
The Fix: Install system-level dependencies before the pip install step:

```bash
RUN apt-get update && apt-get install -y \
    libgl1-mesa-glx \
    && rm -rf /var/lib/apt/lists/*
```

## 5. 中文版

1.实验背景与环境构建

1.1 为什么选择 RAG + PDF 论文？
在AI时代，大模型虽然强大，但对于特定、实时的知识仍存在“幻觉”,比如说如最新的科研论文。**RAG ：Retrieval-Augmented Generation** 就像是给AI挂载了一个实时更新的知识库。

本次实验的挑战在于 **PDF 解析**。科研论文通常包含双栏排版和复杂的公式，这要求系统不仅要能“读”到文字，更要能“理解”文档结构。

1.2 核心架构：Docker + DeepSeek
为了实现高性能、低成本且易于迁移的开发环境，我选择了以下组合：
本项目采用轻量化、可部署、可移植的全栈架构：
- **DeepSeek-R1**：作为基座大模型，负责理解与生成
- **LangChain**：串联RAG全流程（加载 → 分割 → 向量化 → 检索 → 生成）
- **Sentence-Transformers**：本地开源Embedding模型
- **FAISS**：轻量级向量数据库
- **Streamlit**：快速搭建 Web 交互界面
- **Docker + Docker Compose**：环境容器化

1.3 容器化部署实战
我编写了一个精简高效的 `docker-compose.yml` 来驱动整个系统。其核心逻辑在于**开发效率**与**安全隔离**：

``` yaml
services:
  app:
    build: .
    # 端口映射：浏览器输入 localhost:7887 即可访问 Streamlit 界面
    ports:
      - "7887:8501" 
    
    # 安全性：通过环境变量文件管理 DeepSeek API Key
    env_file:
      - .env
    
    # 开发效率：挂载当前目录，实现代码修改后的“热更新”
    volumes:
      - .:/app
    
    # 交互模式：方便在终端实时调试 PDF 解析报错
    stdin_open: true
    tty: true
```

2.PDF 解析与知识库构建

这一部分是RAG的灵魂。我要解决的是：**如何让 DeepSeek 读懂那些“死”在 PDF 里的文字？**

2.1 PDF 解析的“深水区”
科研论文通常不是简单的长文本，它们存在双栏布局和复杂的排版。如果直接暴力读取，语义会被打乱。

2.2 实战：使用 Python 提取核心内容
在我的实验中，我使用了 `PyMuPDF` (fitz)。它的优势在于速度极快，且能较好地保留文档的逻辑结构。

```python
import fitz  # PyMuPDF 库

def extract_text_from_pdf(pdf_path):
    # 打开 PDF 文件
    doc = fitz.open(pdf_path)
    full_text = ""
    
    for page in doc:
        # 提取当前页面的文本
        full_text += page.get_text()
        
    return full_text
```
实验心得：对于双栏论文，建议使用 page.get_text("blocks"),这样可以按物理区块读取，避免左右两栏文字混在一起。

3.核心交互：接入 DeepSeek 推理引擎 LLM Integration

有了结构化的文本块，下一步就是建立与 **DeepSeek-R1** 的对话桥梁。在这一步中，我们不仅要发送问题，还要把“检索到的 PDF 片段”作为背景知识一并塞给模型。

3.1 为什么选择 DeepSeek-R1？
在处理复杂的学术论文时，DeepSeek 的**思维链**能力非常出众。它能识别出论文实验数据之间的逻辑关联，而不是简单地复述文字。

3.2 调用代码实战 
以下是我在容器中运行的核心调用函数。注意我们使用了 `.env` 文件中配置的 API Key：

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

# 加载环境变量
load_dotenv()

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"), 
    base_url="[https://api.deepseek.com](https://api.deepseek.com)"
)

def ask_deepseek(context, query):
    # 构建 Prompt：强制模型基于 Context 回答
    prompt = f"根据以下 PDF 内容回答问题：\n{context}\n\n问题：{query}"
    
    response = client.chat.completions.create(
        model="deepseek-chat",
        messages=[
            {"role": "system", "content": "你是一个专业的学术助手，请基于提供的文档内容给出准确回答。"},
            {"role": "user", "content": prompt},
        ],
        stream=False
    )
    return response.choices[0].message.content
```

3.3 避坑指南：Prompt 的艺术
在RAG实验中，最容易出现的问题是AI“脱离文档瞎编”。为了防止这种情况，我在System Prompt中加入了强约束，这能显著降低模型的“幻觉”。

核心策略： "如果文档中没有相关信息，请直接回答‘抱歉，文档中未提及此内容’，严禁幻觉。"

4.环境构建“血泪史”：网络与版本的双重毒打

在搭建 RAG 开发环境时，我遇到的最大挑战不是算法，而是网络和依赖库的版本冲突。以下是两个必须“抄作业”的解决方案：

4.1 Pip 下载超时与镜像源加速
**痛点：** 在 Docker 中执行 `pip install` 时，默认源经常因为网络抖动导致 `ReadTimeoutError`，整个镜像构建会直接崩溃。

**关键行修正：**
在 `Dockerfile` 中，务必使用国内镜像源（如清华源）并增加超时设置：

```bash
# 使用镜像源加速并忽略缓存，确保构建稳定
RUN pip install --no-cache-dir \
    -i [https://pypi.tuna.tsinghua.edu.cn/simple](https://pypi.tuna.tsinghua.edu.cn/simple) \
    -r requirements.txt
```

4.2 LangChain 的“版本地狱”
痛点：LangChain更新极快，新旧版本API不兼容。如果直接 pip install langchain，下载的最早或最新版本往往会导致程序运行报 ImportError。
关键行修正：
不要在 Dockerfile 里直接写包名，而是在 requirements.txt 中严格锁定版本。这是我实验成功的“黄金组合”：

```text
langchain==0.1.12
langchain-community==0.0.28
langchain-openai==0.0.8
python-dotenv==1.0.1
PyMuPDF==1.23.26
```

4.3 缺失的系统依赖库
痛点： 很多 PDF 解析库如 fitz在轻量级 Docker 镜像如 python:3.9-slim中无法直接运行，因为缺少底层的图形处理库。

关键行修正：在 pip install 之前，必须在 Dockerfile 里先安装系统级依赖。

```bash
RUN apt-get update && apt-get install -y \
    libgl1-mesa-glx \
    && rm -rf /var/lib/apt/lists/*
```






