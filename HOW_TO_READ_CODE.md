# 📖 Hướng Dẫn Đọc Code - Roadmap Chi Tiết

## 🎯 Mục Tiêu
Hiểu hệ thống Text-to-SQL Multi-Agent từ cơ bản đến nâng cao, theo trình tự logic.

---

## 🗺️ Lộ Trình Đọc Code (Recommended Order)

### **GIAI ĐOẠN 1: FOUNDATION - Hiểu Cấu Trúc Cơ Bản** ⭐

#### **Bước 1.1: Đọc README.md**
📁 `README.md`

**Mục đích:** Hiểu tổng quan về dự án
- Hệ thống làm gì?
- Có những components nào?
- Công nghệ gì được dùng?
- Cách chạy project?

**Thời gian:** 5-10 phút

---

#### **Bước 1.2: Xem Dependencies**
📁 `requirements.txt`

**Mục đích:** Biết các thư viện quan trọng
```python
fastapi         # Web framework
uvicorn         # ASGI server
streamlit       # UI framework
langchain       # LLM framework
langgraph       # Agent orchestration
chromadb        # Vector database
langchain-google-genai  # LLM provider
```

**Key takeaways:**
- Hệ thống dùng LangChain/LangGraph
- Vector DB là Chroma
- LLM là Google Gemini
- Frontend là Streamlit

**Thời gian:** 2-3 phút

---

#### **Bước 1.3: Hiểu Config**
📁 `app/config.py`

```python
# File này cực kỳ đơn giản nhưng quan trọng
llm = ChatGoogleGenerativeAI(model='gemini-2.0-flash')
embeddings = HuggingFaceEmbeddings(model_name="...")
```

**Key takeaways:**
- 2 global objects: `llm` và `embeddings`
- Tất cả agents đều dùng chung 2 objects này
- Dễ thay đổi LLM provider (chỉ cần đổi ở đây)

**Thời gian:** 2 phút

---

### **GIAI ĐOẠN 2: DATA MODEL - Hiểu State & Data Flow** ⭐⭐

#### **Bước 2.1: GlobalState - Trái Tim Của Hệ Thống**
📁 `app/state/agent_state.py`

**Tại sao đọc đầu tiên?**
- GlobalState là data structure được truyền qua TẤT CẢ agents
- Hiểu state = hiểu 50% hệ thống

**Cách đọc:**
```python
class GlobalState(TypedDict, total=False):
    # --- Core Session Info ---
    user_id: str
    session_id: str
    status: str  # "query_rewritten", "sql_generated", etc.

    # --- Query Handling ---
    original_query: str       # ← Input từ user
    rewritten_query: str      # ← Sau Query Rewriter

    # --- Schema & Context ---
    schema_context: str       # ← Từ Schema Agent
    relevant_tables: List[str]

    # --- SQL Generation ---
    generated_sql: str        # ← Từ Generation Agent
    validated_sql: str        # ← Sau Validation

    # --- Execution ---
    execution_result: Dict    # ← Kết quả query

    # --- Explanation ---
    natural_language_explanation: str  # ← Output cuối cùng

    # --- Histories (Audit Trail) ---
    rewrite_history: List[Dict]
    generation_history: List[Dict]
    validation_history: List[Dict]
    execution_history: List[Dict]
```

**Mental Model:**
```
GlobalState như một "bản ghi y tế" của câu hỏi
- Mỗi agent đọc state
- Làm việc của mình
- Cập nhật state
- Pass cho agent tiếp theo
```

**Thời gian:** 10 phút

---

### **GIAI ĐOẠN 3: WORKFLOW - Hiểu Orchestration** ⭐⭐⭐

#### **Bước 3.1: LangGraph Workflow**
📁 `app/graph/text_to_sql_graph.py`

**Đây là file QUAN TRỌNG NHẤT để hiểu flow!**

**Cách đọc từng phần:**

##### **Phần 1: Graph Definition**
```python
def build_text_to_sql_graph():
    graph = StateGraph(GlobalState)  # ← State type là GlobalState

    # Đăng ký các nodes (mỗi node là 1 agent function)
    graph.add_node("query_rewriter_node", query_rewriter_node)
    graph.add_node("schema_agent_node", schema_agent_node)
    graph.add_node("query_generation_node", query_generation_node)
    graph.add_node("validation_node", validation_node)
    graph.add_node("query_execution_node", query_execution_node)
    graph.add_node("visualization_node", visualization_node)
    graph.add_node("explainability_node", explainability_node)
```

**Hiểu:**
- Graph có 7 nodes
- Mỗi node là một function nhận `GlobalState` và return `GlobalState`

##### **Phần 2: Sequential Edges**
```python
# Định nghĩa thứ tự chạy
graph.add_edge("query_rewriter_node", "schema_agent_node")
graph.add_edge("schema_agent_node", "query_generation_node")
graph.add_edge("query_generation_node", "validation_node")
graph.add_edge("validation_node", "query_execution_node")
graph.add_edge("query_execution_node", "visualization_node")
graph.add_edge("visualization_node", "explainability_node")
graph.add_edge("explainability_node", END)
```

**Visualize:**
```
START → Query Rewriter → Schema Agent → Query Generation
      → Validation → Execution → Visualization → Explainability → END
```

##### **Phần 3: Entry Point & Compilation**
```python
graph.set_entry_point("query_rewriter_node")  # Bắt đầu từ đây
memory = MemorySaver()  # Lưu state history
compiled_graph = graph.compile(checkpointer=memory)
```

**Thời gian:** 15 phút

**✅ Checkpoint:** Sau bước này bạn đã hiểu:
- Có 7 agents chạy tuần tự
- State được pass qua từng agent
- Entry point là Query Rewriter
- Exit point là Explainability

---

### **GIAI ĐOẠN 4: AGENTS - Hiểu Chi Tiết Từng Agent** ⭐⭐⭐⭐

**Đọc theo thứ tự execution (như trong graph):**

---

#### **Agent 1: Query Rewriter**
📁 `app/agents/query_rewriter_agent.py`

**Đọc theo sections:**

##### **Section 1: Output Schema**
```python
class QueryRewriteOutput(BaseModel):
    rewritten_query: str
    explanation: str
    metadata: dict
```
**Hiểu:** Agent này output 3 fields

##### **Section 2: Prompt Template**
```python
rewriter_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an expert query rewriter..."),
    ("human", "Original: {query}")
])
```
**Hiểu:** Prompt engineering - instructions cho LLM

##### **Section 3: Agent Function**
```python
async def query_rewriter_node(state: GlobalState) -> GlobalState:
    query = state.get("original_query")  # ← Đọc từ state

    parser = PydanticOutputParser(pydantic_object=QueryRewriteOutput)
    chain = rewriter_prompt | llm | parser

    result = await chain.ainvoke({...})  # ← Gọi LLM

    new_state = state.copy()
    new_state.update({
        "rewritten_query": result.rewritten_query,  # ← Cập nhật state
        "rewrite_explanation": result.explanation,
        "status": "query_rewritten"
    })

    return new_state  # ← Return state mới
```

**Pattern chung của TẤT CẢ agents:**
```
1. Đọc data từ state
2. Xử lý (LLM call, validation, execution, etc.)
3. Copy state
4. Update với kết quả
5. Return state mới
```

**Thời gian:** 10 phút

---

#### **Agent 2: Schema Agent (RAG Core)**
📁 `app/agents/schema_agent.py`

**Đây là agent PHỨC TẠP NHẤT - có RAG pipeline**

**Đọc theo sections:**

##### **Section 1: Chroma Setup**
```python
CHROMA_PATH = os.path.join(BASE_DIR, "chroma_schema_index")
_vectorstore = None  # Global singleton
```

##### **Section 2: Build Index Function**
```python
def build_schema_index(schema_docs):
    vectorstore = Chroma.from_documents(
        documents=schema_docs,
        embedding=embeddings,
        persist_directory=CHROMA_PATH
    )
    return vectorstore
```
**Hiểu:**
- Nhận schema documents (từ `utils.py`)
- Tạo embeddings
- Lưu vào Chroma

##### **Section 3: Schema Agent Function**
```python
async def schema_agent_node(state: GlobalState):
    query = state.get("rewritten_query")

    # Load hoặc rebuild vectorstore
    if _vectorstore is None:
        _vectorstore = Chroma(persist_directory=CHROMA_PATH, ...)

    # RAG Retrieval: Tìm top-k relevant tables
    retriever = _vectorstore.as_retriever(search_kwargs={"k": 3})
    results = retriever.invoke(query)  # ← Vector search

    # Normalize documents
    rag_docs = []
    for doc in results:
        rag_docs.append({
            "text": doc.page_content,
            "metadata": {"table_name": doc.metadata.get("table_name")}
        })

    schema_context = "\n".join([doc["text"] for doc in rag_docs])

    # LLM Summarization
    parser = PydanticOutputParser(pydantic_object=SchemaSummary)
    chain = schema_prompt | llm | parser
    summary = await chain.ainvoke({
        "query": query,
        "schema_context": schema_context
    })

    # Update state
    new_state = state.copy()
    new_state.update({
        "schema_context": schema_context,
        "relevant_tables": relevant_tables,
        "rag_docs": rag_docs,
        "schema_summary": summary.summary_text
    })

    return new_state
```

**RAG Flow:**
```
Query → Vector Search → Top-K Documents → LLM Summarization → Schema Summary
```

**Thời gian:** 20 phút (file phức tạp nhất)

---

#### **Agent 3: Query Generation**
📁 `app/agents/query_generation_agent.py`

**Đơn giản hơn - follow pattern chuẩn:**

```python
class SQLGenerationOutput(BaseModel):
    sql: str
    explanation: str

async def query_generation_node(state: GlobalState):
    query = state.get("rewritten_query")
    schema_context = state.get("schema_context")  # ← Từ Schema Agent

    parser = PydanticOutputParser(pydantic_object=SQLGenerationOutput)
    chain = generation_prompt | llm | parser

    result = await chain.ainvoke({
        "query": query,
        "schema_context": schema_context,
        "schema_summary": schema_summary
    })

    sql_query = result.sql.strip()

    # Safety checks
    if not sql_query.lower().startswith("select"):
        sql_query = "SELECT " + sql_query

    new_state.update({
        "generated_sql": sql_query,
        "sql_explanation": explanation
    })

    return new_state
```

**Key Points:**
- Input: rewritten_query + schema_context
- Output: generated_sql
- Có safety checks (enforce SELECT)

**Thời gian:** 10 phút

---

#### **Agent 4: Validation**
📁 `app/agents/validation_agent.py`

**Không dùng LLM - pure logic:**

```python
async def validation_node(state: GlobalState):
    sql_query = state.get("generated_sql")

    # 1. Clean SQL
    sql_query = re.sub(r"```(?:sql)?|```", "", sql_query)

    # 2. Security Validation
    forbidden = ["delete", "drop", "update", "insert", "alter", "truncate"]
    for keyword in forbidden:
        if re.search(rf"\b{keyword}\b", sql_query.lower()):
            return {
                "validation_passed": False,
                "validation_explanation": "Unsafe SQL detected"
            }

    # 3. Syntax Validation (Dry Run)
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor.execute(f"EXPLAIN {sql_query}")  # ← Không thực thi, chỉ check syntax
        explanation = "SQL syntax is valid"
        validation_passed = True
    except sqlite3.Error as e:
        explanation = f"SQL validation failed: {e}"
        validation_passed = False

    new_state.update({
        "validation_passed": validation_passed,
        "validation_explanation": explanation
    })

    return new_state
```

**3 Layers:**
1. SQL Cleaning (remove markdown artifacts)
2. Security Check (regex forbidden keywords)
3. Syntax Check (SQLite EXPLAIN)

**Thời gian:** 10 phút

---

#### **Agent 5: Execution**
📁 `app/agents/execution_agent.py`

**Straightforward - execute SQL:**

```python
def execute_sql_on_replica(sql_query, db_path):
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row

    start = time.time()
    cursor.execute(sql_query)
    rows = cursor.fetchall()
    execution_time = time.time() - start

    columns = [desc[0] for desc in cursor.description]
    result = [dict(row) for row in rows]

    return {
        "success": True,
        "columns": columns,
        "rows": result,
        "row_count": len(result),
        "execution_time": execution_time
    }

async def query_execution_node(state: GlobalState):
    sql_query = state.get("validated_sql") or state.get("generated_sql")

    result = execute_sql_on_replica(sql_query, DB_PATH)

    new_state.update({
        "execution_result": result,
        "status": "query_executed" if result["success"] else "execution_failed"
    })

    return new_state
```

**Thời gian:** 8 phút

---

#### **Agent 6: Visualization**
📁 `app/agents/visualization_agent.py`

**Note:** File này hiện tại là placeholder (chưa được implement đầy đủ)

**Thời gian:** 2 phút (skip qua)

---

#### **Agent 7: Explainability**
📁 `app/agents/explainability_agent.py`

**Follow pattern quen thuộc:**

```python
class ExplanationOutput(BaseModel):
    query_purpose: str
    data_insights: str
    summary: str

async def explainability_node(state: GlobalState):
    sql_query = state.get("validated_sql")
    execution_result = state.get("execution_result")

    sample_rows = execution_result.get("rows", [])[:5]  # Chỉ lấy 5 rows đầu

    parser = PydanticOutputParser(pydantic_object=ExplanationOutput)
    chain = explain_prompt | llm | parser

    result = await chain.ainvoke({
        "sql_query": sql_query,
        "sample_rows": sample_rows
    })

    explanation_text = f"""
    ### What the Query Does:
    {result.query_purpose}

    ### Key Insights:
    {result.data_insights}

    ### Summary:
    {result.summary}
    """

    new_state.update({
        "natural_language_explanation": explanation_text
    })

    return new_state
```

**Thời gian:** 10 phút

---

### **GIAI ĐOẠN 5: API LAYER - Hiểu Request/Response Flow** ⭐⭐⭐

#### **Bước 5.1: API Routes**
📁 `app/api/routes.py`

**Đọc theo sections:**

##### **Section 1: Format Helper**
```python
def format_agent_update(node_name: str, state: GlobalState) -> dict:
    """Tạo message cho từng agent để gửi về frontend"""

    if node_name == "query_rewriter_node":
        msg = f"🔁 Rewriting query...\n✅ Rewritten: {state['rewritten_query']}"

    elif node_name == "schema_agent_node":
        tables = state.get("relevant_tables", [])
        msg = f"📚 Schema retrieved.\n✅ Tables: {', '.join(tables)}"

    # ... các agents khác

    return {
        "stage": node_name,
        "message": msg,
        "state": {
            "generated_sql": state.get("generated_sql"),
            "execution_result": state.get("execution_result"),
            "natural_language_explanation": state.get("natural_language_explanation")
        }
    }
```

**Hiểu:** Function này format state thành user-friendly messages

##### **Section 2: Streaming Endpoint**
```python
@router.post("/query/stream")
async def query_stream(request: Request):
    payload = await request.json()
    user_query = payload["query"]
    session_id = payload["session_id"]

    async def event_stream():
        # Initialize state
        state = GlobalState(
            original_query=user_query,
            session_id=session_id
        )

        # Build graph
        graph = build_text_to_sql_graph()

        # Stream through agents
        async for event in graph.astream(state, config={...}):
            for node_name, node_output in event.items():
                state.update(node_output)  # ← Update state

                update = format_agent_update(node_name, state)
                yield json.dumps(update) + "\n"  # ← SSE format
                await asyncio.sleep(0.3)

        # Final result
        yield json.dumps({
            "event": "complete",
            "rows": state.get("execution_result", {}).get("rows", []),
            "explanation": state.get("natural_language_explanation")
        }) + "\n"

    return StreamingResponse(event_stream(), media_type="application/json")
```

**Flow:**
```
POST /query/stream
    ↓
Initialize GlobalState
    ↓
Build LangGraph
    ↓
For each agent in graph:
    - Agent executes
    - State updates
    - Format message
    - Yield to client (SSE)
    ↓
Final result
```

**Thời gian:** 15 phút

---

#### **Bước 5.2: FastAPI Main**
📁 `app/main.py`

**Siêu đơn giản:**
```python
from fastapi import FastAPI
from app.api.routes import router

app = FastAPI()
app.include_router(router)
```

**Thời gian:** 1 phút

---

### **GIAI ĐOẠN 6: FRONTEND - Hiểu UI** ⭐⭐

#### **Streamlit UI**
📁 `app/ui/stream_ui.py`

**Đọc theo flow:**

##### **Part 1: Setup**
```python
st.title("🧠 Text-to-SQL Multi-Agent System")
query = st.text_input("Enter your query:")
run_button = st.button("Run Query")
```

##### **Part 2: Streaming Request**
```python
if run_button and query:
    url = "http://localhost:8000/query/stream"

    # Placeholders cho dynamic updates
    progress_placeholder = st.empty()
    sql_placeholder = st.empty()
    result_placeholder = st.empty()
    insight_placeholder = st.empty()

    with requests.post(url, json={"query": query}, stream=True) as response:
        for line in response.iter_lines():
            data = json.loads(line)

            # Update progress
            stream_output += f"\n\n### {stage}\n{message}"
            progress_placeholder.markdown(stream_output)

            # Capture SQL
            if "generated_sql" in state:
                sql_text = state["generated_sql"]

            # Capture results
            if "execution_result" in state:
                result_df = pd.DataFrame(state["execution_result"]["rows"])

            # Capture explanation
            if "natural_language_explanation" in state:
                explanation_text = state["natural_language_explanation"]
```

##### **Part 3: Final Display**
```python
    # Display SQL
    if sql_text:
        st.subheader("🧮 Generated SQL")
        st.code(sql_text, language="sql")

    # Display results
    if result_df is not None:
        st.subheader("📊 Results")
        st.dataframe(result_df)

    # Display explanation
    if explanation_text:
        st.subheader("💡 Insights")
        st.markdown(explanation_text)
```

**Thời gian:** 15 phút

---

### **GIAI ĐOẠN 7: UTILITIES - Hiểu Helper Functions** ⭐

#### **Utils - Schema Extraction**
📁 `app/utils.py`

```python
def extract_schema_from_db(db_path: str):
    """Trích xuất schema từ SQLite database"""
    conn = sqlite3.connect(db_path)

    # Get all tables (exclude system tables)
    tables = cursor.execute("""
        SELECT name FROM sqlite_master
        WHERE type='table' AND name NOT LIKE 'sqlite_%'
    """).fetchall()

    docs = []
    for (table_name,) in tables:
        # Get column info
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

**Được dùng bởi:** Schema Agent để build Chroma index

**Thời gian:** 8 phút

---

## 📊 Tổng Kết Roadmap

### **Thứ Tự Đọc Tối Ưu:**

```
1. README.md (5 min)
2. requirements.txt (3 min)
3. app/config.py (2 min)
          ↓
4. app/state/agent_state.py (10 min) ← QUAN TRỌNG
          ↓
5. app/graph/text_to_sql_graph.py (15 min) ← QUAN TRỌNG NHẤT
          ↓
6. Agents (theo thứ tự execution):
   - query_rewriter_agent.py (10 min)
   - schema_agent.py (20 min) ← PHỨC TẠP NHẤT
   - query_generation_agent.py (10 min)
   - validation_agent.py (10 min)
   - execution_agent.py (8 min)
   - explainability_agent.py (10 min)
          ↓
7. app/api/routes.py (15 min)
8. app/main.py (1 min)
          ↓
9. app/ui/stream_ui.py (15 min)
10. app/utils.py (8 min)
```

**Tổng thời gian:** ~2.5 - 3 giờ (đọc kỹ và hiểu)

---

## 🎯 Tips Khi Đọc Code

### **1. Xác định pattern chung của agents:**
```python
async def agent_node(state: GlobalState) -> GlobalState:
    # 1. Extract data từ state
    data = state.get("some_field")

    # 2. Process (LLM call, validation, execution, etc.)
    result = do_something(data)

    # 3. Update state
    new_state = state.copy()
    new_state.update({"new_field": result})

    # 4. Return
    return new_state
```

### **2. Trace data flow:**
```
original_query (user input)
    → rewritten_query (Query Rewriter)
    → schema_context (Schema Agent)
    → generated_sql (Query Generation)
    → validated_sql (Validation)
    → execution_result (Execution)
    → natural_language_explanation (Explainability)
    → (return to user)
```

### **3. Chú ý Pydantic schemas:**
Mỗi agent có output schema riêng:
- `QueryRewriteOutput`
- `SchemaSummary`
- `SQLGenerationOutput`
- `ExplanationOutput`

### **4. LangChain LCEL syntax:**
```python
chain = prompt | llm | parser
result = await chain.ainvoke({...})
```

Đây là cú pháp của LangChain Expression Language (LCEL)

---

## 🔍 Deep Dive Topics (Nâng Cao)

Sau khi đọc hết roadmap chính, có thể deep dive vào:

### **Topic 1: RAG Implementation**
- Schema extraction (`utils.py`)
- Chroma vector store setup (`schema_agent.py`)
- Embedding generation
- Retrieval pipeline
- LLM-based summarization

### **Topic 2: LangGraph Orchestration**
- StateGraph mechanics
- Edge definitions
- Entry/exit points
- Checkpointing with MemorySaver
- Async streaming

### **Topic 3: Security Best Practices**
- Multi-layer validation
- Read-only enforcement
- SQL injection prevention
- Safe execution patterns

### **Topic 4: Streaming Architecture**
- FastAPI SSE (Server-Sent Events)
- Async generators
- StreamingResponse
- Client-side handling (Streamlit)

---

## ✅ Checklist - Bạn Đã Hiểu Khi:

- [ ] Biết hệ thống có 7 agents nào
- [ ] Hiểu thứ tự execution của agents
- [ ] Biết GlobalState chứa gì và vai trò của nó
- [ ] Hiểu cách mỗi agent nhận input và tạo output
- [ ] Biết RAG được dùng ở đâu và như thế nào
- [ ] Hiểu security validation có những layers nào
- [ ] Biết API streaming hoạt động ra sao
- [ ] Hiểu UI cập nhật real-time như thế nào
- [ ] Có thể trace được data flow từ đầu đến cuối
- [ ] Biết cách thêm một agent mới vào hệ thống

---

## 🚀 Next Steps

Sau khi đọc xong:

1. **Run the system:**
   ```bash
   uvicorn app.main:app --reload
   streamlit run app/ui/stream_ui.py
   ```

2. **Thử debug:**
   - Thêm breakpoints
   - Print state ở mỗi agent
   - Xem logs

3. **Modify code:**
   - Thay đổi prompts
   - Thêm validation rules
   - Customize output format

4. **Extend:**
   - Thêm agent mới
   - Support multi-database
   - Add conversation memory

---

## 📚 Tài Liệu Tham Khảo

- **LangChain Docs:** https://python.langchain.com/docs/
- **LangGraph Docs:** https://langchain-ai.github.io/langgraph/
- **FastAPI Docs:** https://fastapi.tiangolo.com/
- **Streamlit Docs:** https://docs.streamlit.io/
- **Chroma Docs:** https://docs.trychroma.com/

---

**Happy Reading! 📖✨**
