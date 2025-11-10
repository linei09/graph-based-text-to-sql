# 🧠 Giải Thích Chi Tiết Về Hệ Thống Text-to-SQL Multi-Agent

## 📋 Tổng Quan

Dự án này là một hệ thống **Text-to-SQL thông minh** sử dụng kiến trúc **Multi-Agent** được xây dựng trên **LangGraph**. Hệ thống có khả năng:
- Chuyển đổi câu hỏi bằng ngôn ngữ tự nhiên thành câu lệnh SQL
- Thực thi SQL trên database
- Giải thích kết quả bằng ngôn ngữ tự nhiên
- Streaming kết quả real-time đến người dùng

---

## 🏗️ Kiến Trúc Tổng Thể

### **Luồng Xử Lý (Pipeline)**

```
User Query (Câu hỏi tự nhiên)
    ↓
1. Query Rewriter Agent       → Viết lại câu hỏi cho rõ ràng hơn
    ↓
2. Schema Agent               → Tìm kiếm schema phù hợp (RAG với Chroma)
    ↓
3. Query Generation Agent     → Sinh câu lệnh SQL
    ↓
4. Validation Agent           → Kiểm tra tính hợp lệ và bảo mật của SQL
    ↓
5. Execution Agent            → Thực thi SQL trên database
    ↓
6. Visualization Agent        → Chuẩn bị dữ liệu cho visualization
    ↓
7. Explainability Agent       → Giải thích SQL và kết quả bằng ngôn ngữ tự nhiên
    ↓
Results + Explanation         → Trả về cho người dùng
```

### **Công Nghệ Sử Dụng**

| Thành Phần | Công Nghệ |
|-----------|-----------|
| **Backend Framework** | FastAPI |
| **Agent Orchestration** | LangGraph |
| **LLM Framework** | LangChain |
| **LLM Model** | Google Gemini 2.0 Flash |
| **Embeddings** | HuggingFace (sentence-transformers/all-MiniLM-L6-v2) |
| **Vector Database** | Chroma |
| **Database** | SQLite |
| **Frontend** | Streamlit |
| **Streaming** | Server-Sent Events (SSE) |

---

## 🔍 Giải Thích Chi Tiết Từng Agent

### 1️⃣ **Query Rewriter Agent** (`query_rewriter_agent.py`)

**Mục đích:** Cải thiện và làm rõ câu hỏi của người dùng trước khi xử lý.

**Cách hoạt động:**
- Nhận câu hỏi gốc từ người dùng
- Sử dụng LLM để viết lại câu hỏi sao cho SQL-friendly hơn
- Làm rõ các filter, aggregation, date range
- Trả về cả câu hỏi đã viết lại + lý do viết lại

**Ví dụ:**
```
Input:  "Có bao nhiêu khách hàng năm ngoái?"
Output: "Đếm số lượng khách hàng được tạo trong năm 2024"
```

**Output Structure:**
```python
{
    "rewritten_query": "Câu hỏi đã được viết lại",
    "explanation": "Lý do viết lại",
    "metadata": {}
}
```

---

### 2️⃣ **Schema Agent** (`schema_agent.py`)

**Mục đích:** Tìm kiếm và cung cấp schema database phù hợp với câu hỏi.

**Cách hoạt động (RAG Pipeline):**

1. **Index Building (Offline):**
   - Trích xuất schema từ SQLite database
   - Tạo embeddings cho mỗi table definition
   - Lưu vào Chroma vector database

2. **Schema Retrieval (Runtime):**
   - Nhận câu hỏi đã được viết lại
   - Tìm kiếm top-k (k=3) tables có liên quan nhất
   - Sử dụng semantic search với embeddings

3. **Schema Summarization:**
   - LLM tóm tắt schema thành structured output
   - Xác định: tables, columns, relationships

**Output Structure:**
```python
{
    "key_tables": ["users", "orders"],
    "key_columns": ["user_id", "created_at", "total"],
    "relationships": "users.id = orders.user_id",
    "summary_text": "Mô tả ngắn gọn về schema"
}
```

**Lợi ích của RAG:**
- Giảm context size cho LLM
- Chỉ lấy schema thực sự cần thiết
- Có thể scale với database lớn

---

### 3️⃣ **Query Generation Agent** (`query_generation_agent.py`)

**Mục đích:** Sinh câu lệnh SQL từ câu hỏi tự nhiên.

**Input:**
- Câu hỏi đã viết lại
- Schema context (từ Schema Agent)
- Schema summary

**Cách hoạt động:**
- Prompt engineering với strict guidelines:
  - Chỉ dùng SELECT (read-only)
  - Không được thêm columns không có trong schema
  - Xử lý date filtering đúng cách (STRFTIME)
- Sử dụng **Pydantic Structured Output** để đảm bảo format
- Trả về cả SQL + explanation

**Safety Measures:**
```python
# Đảm bảo chỉ SELECT
if not sql.startswith("select"):
    sql = "SELECT " + sql

# Thêm semicolon nếu thiếu
if not sql.endswith(";"):
    sql += ";"
```

**Output:**
```python
{
    "sql": "SELECT COUNT(*) FROM users WHERE created_at >= '2024-01-01';",
    "explanation": "Đếm số lượng users được tạo từ đầu năm 2024"
}
```

---

### 4️⃣ **Validation Agent** (`validation_agent.py`)

**Mục đích:** Kiểm tra tính hợp lệ, bảo mật, và syntax của SQL.

**3 Lớp Validation:**

1. **SQL Cleaning:**
   ```python
   # Loại bỏ markdown code blocks
   sql = re.sub(r"```(?:sql)?|```", "", sql)
   ```

2. **Security Validation:**
   ```python
   # Chặn các keyword nguy hiểm
   forbidden = ["delete", "drop", "update", "insert", "alter", "truncate"]

   # Đảm bảo read-only
   if not sql.lower().startswith("select"):
       return FAILED
   ```

3. **Syntax Validation:**
   ```python
   # Sử dụng SQLite EXPLAIN để check syntax
   cursor.execute(f"EXPLAIN {sql_query}")
   ```

**Output:**
```python
{
    "validation_passed": True/False,
    "validation_explanation": "Chi tiết lý do pass/fail",
    "validation_history": [...]
}
```

---

### 5️⃣ **Execution Agent** (`execution_agent.py`)

**Mục đích:** Thực thi SQL đã được validate trên database.

**Cách hoạt động:**
```python
def execute_sql_on_replica(sql_query, db_path):
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row  # Truy cập columns by name

    start = time.time()
    cursor.execute(sql_query)
    rows = cursor.fetchall()
    execution_time = time.time() - start

    return {
        "columns": [...],
        "rows": [...],
        "row_count": len(rows),
        "execution_time": execution_time
    }
```

**Output:**
```python
{
    "success": True,
    "columns": ["user_id", "name", "email"],
    "rows": [
        {"user_id": 1, "name": "John", "email": "john@example.com"},
        ...
    ],
    "row_count": 10,
    "execution_time": 0.0234
}
```

---

### 6️⃣ **Visualization Agent** (`visualization_agent.py`)

**Mục đích:** Chuẩn bị dữ liệu cho visualization (charts, tables).

*Note: Trong code hiện tại, agent này chưa được implement đầy đủ, chủ yếu là placeholder cho future enhancements.*

---

### 7️⃣ **Explainability Agent** (`explainability_agent.py`)

**Mục đích:** Giải thích SQL query và kết quả bằng ngôn ngữ tự nhiên.

**Cách hoạt động:**
- Nhận: SQL query + execution results (5 rows đầu tiên)
- LLM phân tích và tạo structured explanation:
  - **Query Purpose**: SQL đang làm gì?
  - **Data Insights**: Những insights chính từ kết quả
  - **Summary**: Tóm tắt tổng thể

**Output Format:**
```markdown
### 🧩 What the Query Does:
This query counts all users created in 2024

### 📊 Key Insights:
- Total of 150 users were registered in 2024
- Most registrations happened in Q1

### 📝 Summary:
The database shows significant user growth in early 2024
```

---

## 🔄 LangGraph Workflow (`text_to_sql_graph.py`)

### **Graph Definition**

```python
graph = StateGraph(GlobalState)

# Đăng ký các nodes (agents)
graph.add_node("query_rewriter_node", query_rewriter_node)
graph.add_node("schema_agent_node", schema_agent_node)
# ... các agents khác

# Định nghĩa sequential flow
graph.add_edge("query_rewriter_node", "schema_agent_node")
graph.add_edge("schema_agent_node", "query_generation_node")
graph.add_edge("query_generation_node", "validation_node")
# ... tiếp tục

# Entry point và end point
graph.set_entry_point("query_rewriter_node")
graph.add_edge("explainability_node", END)

# Compile với memory
compiled_graph = graph.compile(checkpointer=MemorySaver())
```

### **GlobalState - Shared Memory**

Tất cả agents chia sẻ một **GlobalState** object:

```python
class GlobalState(TypedDict):
    # Session info
    user_id: str
    session_id: str
    status: str

    # Query processing
    original_query: str
    rewritten_query: str

    # Schema context
    schema_context: str
    relevant_tables: List[str]
    rag_docs: List[Dict]

    # SQL generation & validation
    generated_sql: str
    validated_sql: str
    validation_passed: bool

    # Execution results
    execution_result: Dict

    # Explanation
    natural_language_explanation: str

    # Histories (audit trail)
    rewrite_history: List[Dict]
    generation_history: List[Dict]
    validation_history: List[Dict]
    execution_history: List[Dict]
    explanation_history: List[Dict]
```

**Tại sao dùng GlobalState?**
- Tất cả agents đều có thể đọc/ghi state
- Tracking history đầy đủ
- Dễ debug và monitor
- Support cho multi-turn conversations (future)

---

## 🌐 API Layer (`routes.py`)

### **Streaming Endpoint**

```python
@router.post("/query/stream")
async def query_stream(request: Request):
    # Parse request
    query = request.json()["query"]

    # Initialize state
    state = GlobalState(
        original_query=query,
        session_id=session_id
    )

    # Build graph
    graph = build_text_to_sql_graph()

    # Stream qua từng agent
    async for event in graph.astream(state):
        for node_name, node_output in event.items():
            state.update(node_output)

            # Format và gửi update
            update = format_agent_update(node_name, state)
            yield json.dumps(update) + "\n"

    # Final result
    yield json.dumps({"event": "complete", ...})
```

**Streaming Benefits:**
- Real-time feedback cho user
- User biết hệ thống đang làm gì
- Better UX so với blocking request

---

## 🎨 Frontend (`stream_ui.py`)

**Streamlit UI với Real-time Updates:**

```python
# Gửi request với streaming
with requests.post(url, json={"query": query}, stream=True) as response:
    for line in response.iter_lines():
        data = json.loads(line)

        # Update progress
        progress_placeholder.markdown(stream_output)

        # Capture SQL
        if "generated_sql" in state:
            sql_placeholder.code(sql)

        # Capture results
        if "execution_result" in state:
            result_placeholder.dataframe(df)

        # Capture explanation
        if "natural_language_explanation" in state:
            insight_placeholder.markdown(explanation)
```

**UI Components:**
1. Query input box
2. Real-time progress display
3. Generated SQL code block
4. Results dataframe
5. Natural language insights

---

## 🔐 Security Features

### **1. SQL Injection Prevention**
- Chỉ cho phép SELECT queries
- Block tất cả DML/DDL commands
- Syntax validation trước khi execute

### **2. Database Safety**
- Execute trên replica/sandbox database
- Read-only mode
- No modifications allowed

### **3. Validation Pipeline**
```
Generated SQL
    → Regex pattern matching (forbidden keywords)
    → Read-only check
    → Syntax validation (EXPLAIN)
    → Execute
```

---

## 📊 Database Schema Management

### **Schema Extraction** (`utils.py`)

```python
def extract_schema_from_db(db_path):
    # Get all user tables
    tables = cursor.execute("""
        SELECT name FROM sqlite_master
        WHERE type='table' AND name NOT LIKE 'sqlite_%'
    """)

    docs = []
    for table_name in tables:
        # Get columns info
        columns = cursor.execute(f"PRAGMA table_info({table_name})")

        # Format schema text
        schema_text = f"Table: {table_name}\nColumns:\n..."

        # Create LangChain Document
        docs.append(Document(
            page_content=schema_text,
            metadata={"source": table_name}
        ))

    return docs
```

### **Chroma Vector Index**
- Mỗi table definition được embed thành vector
- Semantic search để tìm relevant tables
- Tự động rebuild nếu schema thay đổi

---

## 🚀 Ưu Điểm Của Kiến Trúc Này

### **1. Modularity (Tính Mô-đun)**
- Mỗi agent là một module độc lập
- Dễ thêm/sửa/xóa agents
- Dễ test từng component

### **2. Scalability**
- RAG approach cho phép handle large schemas
- Streaming giảm latency
- Có thể chạy agents parallel (future)

### **3. Transparency**
- User thấy được từng bước xử lý
- Audit trail đầy đủ
- Dễ debug khi có lỗi

### **4. Safety First**
- Multi-layer validation
- Read-only by design
- Structured outputs (ít hallucination hơn)

### **5. Extensibility**
- Dễ thêm features mới:
  - Multi-database support
  - Conversation memory
  - Self-healing SQL correction
  - Advanced visualization

---

## 🎯 Use Cases

1. **Business Intelligence:**
   - "Revenue tháng này so với tháng trước?"
   - "Top 10 sản phẩm bán chạy nhất?"

2. **Data Analysis:**
   - "Average order value by customer segment?"
   - "Churn rate trong Q1 2024?"

3. **Reporting:**
   - "Tổng số users active hôm nay?"
   - "Inventory levels below threshold?"

---

## 🔮 Future Enhancements

### **Được đề cập trong README:**

1. **Memory for Multi-turn Conversations**
   - Lưu context của các câu hỏi trước
   - Support follow-up questions

2. **Multi-Database Support**
   - PostgreSQL, MySQL, MongoDB
   - Database-specific SQL dialects

3. **Query Caching**
   - Cache kết quả của queries phổ biến
   - Giảm tải database

4. **Self-Healing SQL**
   - Nếu SQL fail, tự động fix và retry
   - Learn from validation errors

5. **Smart Visualization**
   - Tự động suggest chart types
   - Generate plots/graphs

---

## 💡 Những Điểm Đáng Học Hỏi

### **1. Prompt Engineering**
Mỗi agent có prompts được thiết kế cẩn thận:
- Clear instructions
- Format specifications
- Safety guidelines

### **2. Structured Outputs**
Sử dụng Pydantic để enforce output format:
```python
class QueryRewriteOutput(BaseModel):
    rewritten_query: str = Field(...)
    explanation: str = Field(...)
```

### **3. Error Handling**
Graceful degradation ở mọi level:
```python
try:
    result = await chain.ainvoke(...)
except Exception as e:
    # Fallback behavior
    result = default_value
```

### **4. RAG Implementation**
Practical example của RAG:
- Document chunking (table definitions)
- Embedding generation
- Semantic retrieval
- LLM summarization

### **5. Agent Orchestration**
LangGraph patterns:
- Sequential flow
- Shared state
- Checkpointing
- Streaming

---

## 📝 Kết Luận

Đây là một **production-ready** implementation của Text-to-SQL với:
- ✅ Multi-agent architecture
- ✅ Real-time streaming
- ✅ RAG-based schema retrieval
- ✅ Comprehensive validation
- ✅ Natural language explanation
- ✅ Security-first design

**Core Innovation:**
Thay vì một LLM call duy nhất, hệ thống phân tách thành 7 specialized agents, mỗi agent có trách nhiệm rõ ràng, cùng làm việc qua LangGraph orchestration để tạo ra kết quả tốt hơn, safe hơn, và explainable hơn.
