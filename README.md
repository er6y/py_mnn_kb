# py_mnn_kb — MNN Knowledge Base
本地向量知识库，与 Android OfflineAI RAG 实现完全兼容。可作为 **OpenClaw Skill** 使用，让 AI Agent 自动管理和查询私有知识库。
---
## 特性
- **跨平台数据库兼容**：`knowledge_graph.db` 格式与 Android 端完全一致，PC 构建的知识库可直接放手机使用，反之亦然
- **自动下载模型**：首次运行自动从 ModelScope 下载 `Qwen3-Embedding-0.6B-MNN-int4`，无需手动配置
- **GraphRAG 召回**：向量相似度 + BM25 关键词 + 知识图谱实体关系三路融合召回
- **增量追加**：`build` 命令对已有知识库只追加新文件，不重建
- **JSON 结构化解析**：自动识别 Alpaca/CoT/DPO/Conversation 等训练集格式，按条目索引
---
## 目录结构
```
py_mnn_kb/
├── py_mnn_kb.py            # 核心库 + CLI 入口（单文件）
├── config.json             # 配置（路径、RAG参数、LLM API）
├── requirements.txt        # Python 依赖
├── example_stop.json       # 停用词表（可自定义）
├── example_terms.json      # 专业术语示例
├── README.md
└── knowledge_bases/        # 知识库存储目录（自动创建）
    └── <kb_name>/
        ├── knowledge_graph.db
        └── metadata.json
```
---
## 快速开始
### 1. 安装依赖
```bash
pip install -r requirements.txt
```
`requirements.txt` 说明：
| 包 | 用途 | 必须 |
|---|---|---|
| `MNN>=3.0.0` | MNN Python 推理后端（嵌入向量计算） | ✅ |
| `jieba>=0.42.1` | 中文分词（NER 实体提取） | ✅ |
| `openai>=1.0.0` | LLM API 调用（query 命令的 LLM 生成） | query+LLM |
| `pdfplumber` | PDF 解析 | 按需 |
| `python-docx` | .docx 解析 | 按需 |
| `python-pptx` | .pptx 解析 | 按需 |
| `openpyxl` | .xlsx 解析 | 按需 |
| `beautifulsoup4` | HTML 解析 | 按需 |
### 2. 配置 config.json
```json
{
  "embedding": {
    "model_dir": "Qwen3-Embedding-0.6B-MNN-int4"
  },
  "llm_api": {
    "base_url": "https://api.deepseek.com/v1",
    "api_key": "YOUR_API_KEY",
    "model": "deepseek-chat"
  }
}
```
- `model_dir`：嵌入模型目录，**首次运行自动从 ModelScope 下载**，无需手动操作
- `llm_api`：仅 `query` 命令需要，使用 `--no-llm` 则不需要
### 3. 运行
```bash
# 构建知识库（首次运行会自动下载模型）
python py_mnn_kb.py build ./my_docs/ --kb myproject
# 追加笔记
python py_mnn_kb.py note "今天会议决定..." --kb myproject
# 查询（含 LLM 回答）
python py_mnn_kb.py query "部署流程是什么？" --kb myproject
# 纯召回，不调 LLM
python py_mnn_kb.py query "部署流程是什么？" --kb myproject --no-llm
```
---
## CLI 命令参考
### `build` — 构建/追加知识库
```
python py_mnn_kb.py build <dir> [--kb <name>] [--config <path>]
```
| 参数 | 说明 |
|---|---|
| `dir` | 要索引的目录（递归扫描所有支持的文件） |
| `--kb` | 知识库名称（默认：config.json 中的 `default_name`） |
| `--config` | 配置文件路径（默认：与 py_mnn_kb.py 同目录的 config.json） |
**支持的文件格式**：`.txt` `.md` `.pdf` `.docx` `.pptx` `.xlsx` `.csv` `.html` `.json` `.jsonl`
**示例**：
```bash
python py_mnn_kb.py build ../docs/ --kb company_kb
python py_mnn_kb.py build /tmp/new_files/ --kb company_kb   # 追加，不覆盖老数据
```
---
### `note` — 添加文本笔记
```
python py_mnn_kb.py note "<text>" [--kb <name>] [--title <title>]
```
| 参数 | 说明 |
|---|---|
| `text` | 要添加的文本内容 |
| `--kb` | 知识库名称 |
| `--title` | 可选标题，写入 metadata |
**示例**：
```bash
python py_mnn_kb.py note "2025年Q1产品路线图：重点推进A和B两个模块..." --kb company_kb
python py_mnn_kb.py note "$(cat meeting_notes.txt)" --kb company_kb --title "周会记录"
```
---
### `query` — RAG 查询
```
python py_mnn_kb.py query "<prompt>" [--kb <name>] [--no-llm] [--output json]
```
| 参数 | 说明 |
|---|---|
| `prompt` | 查询问题 |
| `--kb` | 知识库名称 |
| `--no-llm` | 只返回召回的原始文档片段，不调 LLM |
| `--output json` | 以 JSON 格式输出完整结果（供 Agent 解析） |
**示例**：
```bash
python py_mnn_kb.py query "STAR1200的技术规格是什么？" --kb company_kb
python py_mnn_kb.py query "DDR sorting原理" --kb company_kb --no-llm
python py_mnn_kb.py --output json query "产品路线图" --kb company_kb
```
**JSON 输出格式**：
```json
{
  "status": "ok",
  "command": "query",
  "kb_name": "company_kb",
  "prompt": "产品路线图",
  "llm_used": true,
  "context": "Document1 [ID:42]:\n...",
  "context_chars": 4800,
  "answer": "根据知识库内容...",
  "answer_chars": 512
}
```
---
## Python API（作为库使用）
```python
from py_mnn_kb import KnowledgeBase
kb = KnowledgeBase("path/to/config.json")
# 1. 构建知识库
stats = kb.build_kb(files=["doc1.pdf", "manual.docx"], kb_name="myproject")
# → {'status': 'ok', 'kb_name': 'myproject', 'chunks_added': 86, 'elapsed_sec': 45.2}
# 2. 添加笔记
stats = kb.add_note(text="关键发现：...", kb_name="myproject", title="会议纪要")
# → {'status': 'ok', 'chunks_added': 1, 'elapsed_sec': 1.2}
# 3. RAG 召回（返回拼接后的上下文字符串）
context = kb.query_kb(prompt="部署流程是什么？", kb_name="myproject")
# → "Document1 [ID:42 source:manual.docx]:\n部署分三步..."
```
---
## 作为 OpenClaw Skill 使用
### Skill 定义
```python
from py_mnn_kb import KnowledgeBase
kb = KnowledgeBase()
SKILLS = {
    "kb_build": {
        "description": "从本地目录或文件构建/追加向量知识库。支持 PDF/Word/PPT/Excel/HTML/TXT/JSON 等格式，JSON 训练集自动按结构化条目索引。",
        "function": lambda dir_path, kb_name="default": kb.build_kb(
            files=[str(f) for f in Path(dir_path).rglob("*") if f.is_file()],
            kb_name=kb_name
        ),
        "parameters": {
            "dir_path": "要索引的目录路径",
            "kb_name": "知识库名称（可选，默认 'default'）"
        }
    },
    "kb_note": {
        "description": "将一段文本（笔记、摘要、聊天片段）直接插入知识库，立即可被召回。适合用户口述或粘贴的零散知识点。",
        "function": lambda text, kb_name="default", title="": kb.add_note(text, kb_name, title=title),
        "parameters": {
            "text": "要存入的文本内容",
            "kb_name": "知识库名称（可选）",
            "title": "标题（可选）"
        }
    },
    "kb_query": {
        "description": "从知识库召回与问题最相关的文档片段，返回上下文字符串。Agent 可直接将此上下文拼入 prompt 让 LLM 回答。",
        "function": lambda prompt, kb_name="default": kb.query_kb(prompt, kb_name),
        "parameters": {
            "prompt": "查询问题或关键词",
            "kb_name": "知识库名称（可选）"
        }
    }
}
```
### Agent 使用示例
**场景 A：用户上传文件 → 自动入库**
```
用户："把这个 PDF 加入知识库"
Agent → 把 PDF 存到临时目录 → 调用 kb_build(dir_path=tmp_dir, kb_name="user_kb")
      → 显示进度："已添加 32 个知识片段到 user_kb"
```
**场景 B：用户指定文本内容 → 插入笔记**
```
用户："记住这个信息：STAR2000 在低温下写入性能提升 8%"
Agent → 调用 kb_note(text="...", kb_name="user_kb", title="技术发现")
      → "已保存到知识库 user_kb"
```
**场景 C：用户提问 → 知识库辅助回答**
```
用户："NAND 筛选的核心流程是什么？"
Agent → 调用 kb_query(prompt="NAND筛选核心流程", kb_name="user_kb")
      → 获取 context → 拼入 LLM prompt → 生成回答
```
**场景 D：总结网页/文章 → 存入知识库**
```
用户："总结这个网页并保存"
Agent → 爬取网页内容 → LLM 总结成知识点格式 → 存为临时 alpaca JSON
      → 调用 kb_build(dir_path=tmp_dir, kb_name="user_kb")
```
### OpenClaw Skill 提示词建议
```
你有以下知识库工具可以使用：
1. kb_build(dir_path, kb_name) - 将目录中的文件批量导入知识库
   - 用户说"把这些文件加入知识库"、"索引这个目录"时调用
   - 支持格式：PDF/Word/PPT/Excel/HTML/TXT/JSON
   - 文件先保存到临时目录，再调用此函数
2. kb_note(text, kb_name, title) - 将文本片段直接插入知识库
   - 用户说"记住这个"、"保存这段话"、"添加笔记"时调用
   - 也用于将网页摘要、会议记录等零散信息入库
3. kb_query(prompt, kb_name) - 从知识库召回相关内容
   - 用户提问时，先调用此函数获取 context，再生成回答
   - 返回的 context 直接拼入你的 prompt："根据以下资料：{context}\n\n{question}"
注意：
- kb_name 默认用 "default"，用户指定不同项目时用不同名称
- build 是增量追加，不会覆盖已有数据
- query 返回原始上下文，你负责理解和生成最终回答
```
---
## config.json 参数说明
### `chunking` — 文本分块
| 参数 | 默认值 | 说明 |
|---|---|---|
| `chunk_size` | 500 | 分块大小（字符数），对齐 Android 默认值 |
| `chunk_overlap` | 100 | 相邻分块重叠，保证上下文连续性 |
| `min_chunk_size` | 10 | 最小分块，过滤噪声片段 |
| `enable_json_dataset_opt` | true | JSON 训练集按条目整体索引，不再二次切分 |
### `retrieval` — 召回参数
| 参数 | 默认值 | 说明 |
|---|---|---|
| `search_depth` | 10 | 最终返回的文档数 |
| `bm25_enabled` | true | 启用 BM25 关键词召回（与向量召回 RRF 融合） |
| `graph_rag_enabled` | true | 启用知识图谱实体关系增强 |
| `graph_rag_weight_preset` | 1 | 融合权重：0=向量优先, 1=均衡, 2=图谱增强 |
| `graph_rag_vector_expand` | 20 | 向量粗召回数量（再经图谱重排） |
### `graph_ner` — 实体识别
| 参数 | 说明 |
|---|---|
| `custom_dict_path` | 专业术语词典 JSON（提升领域实体识别准确率） |
| `stopwords_path` | 停用词表 JSON（过滤高频噪声词） |
| `enabled` | 是否启用 Graph NER |
---
## 数据库格式（与 Android 完全兼容）
SQLite 表结构：
```
documents      - 文本分块 + 向量 (BLOB, little-endian float32 × 1024)
entities       - NER 实体 + 频率统计
entity_edges   - 实体共现关系图（带权重）
chunk_entities - chunk-实体多对多映射
metadata       - 库元信息（model、dim、chunk 参数等）
```
向量序列化与 Android 完全一致：
```python
# Python
struct.pack(f"<{len(vec)}f", *vec)
# Java
ByteBuffer.allocate(n*4).order(ByteOrder.LITTLE_ENDIAN).putFloat(v)...
```