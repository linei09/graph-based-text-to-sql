# 🤔 Tại Sao Cần Bước Tóm Tắt Schema?

## ❓ Câu Hỏi

**"Tại sao phải có bước Schema Agent tóm tắt schema, mà không đưa toàn bộ schema vào Query Generation Agent luôn?"**

---

## 📊 So Sánh 2 Approaches

### **Approach 1: Đưa Toàn Bộ Schema (Naive Approach)**
```python
# Schema Agent: Chỉ lấy schema, không tóm tắt
schema_context = get_entire_database_schema()  # Toàn bộ schema

# Query Generation Agent
prompt = f"""
Schema:
{schema_context}  # ← Có thể rất dài!

User Query: {user_query}

Generate SQL.
"""
```

### **Approach 2: RAG + Summarization (Current Approach)**
```python
# Schema Agent: RAG retrieval + Summarization
relevant_tables = vector_search(user_query, k=3)  # Chỉ lấy 3 tables liên quan
schema_context = get_schema_for_tables(relevant_tables)
schema_summary = llm_summarize(schema_context)  # Tóm tắt ngắn gọn

# Query Generation Agent
prompt = f"""
Schema Context (relevant tables only):
{schema_context}  # ← Ngắn gọn, focused

Schema Summary:
{schema_summary}  # ← High-level overview

User Query: {user_query}

Generate SQL.
"""
```

---

## 🎯 Lý Do Chi Tiết

### **1. Context Window Limitations (Giới Hạn Token)**

**Vấn đề:**
- LLMs có giới hạn context window (ví dụ: 128k tokens cho Gemini)
- Schema của database lớn có thể rất dài

**Ví dụ thực tế:**
```sql
-- Database có 50 tables, mỗi table 20 columns
-- Mỗi table definition ~500 tokens
-- Tổng: 50 × 500 = 25,000 tokens chỉ cho schema!

-- Nếu database có 200 tables → 100,000 tokens
-- Không còn chỗ cho query, instructions, examples!
```

**Với RAG + Summarization:**
```sql
-- Chỉ lấy 3 tables relevant: 3 × 500 = 1,500 tokens
-- Schema summary: ~200 tokens
-- Tổng: 1,700 tokens (giảm 93%!)
```

**Benefit:** Tiết kiệm context window cho các phần khác (instructions, few-shot examples, etc.)

---

### **2. Relevance & Focus (Độ Liên Quan)**

**Vấn đề với toàn bộ schema:**
```
User query: "Đếm số users đăng ký tháng này"

Schema gồm 50 tables:
✓ users (RELEVANT)
✗ products
✗ orders
✗ inventory
✗ shipments
✗ payments
✗ reviews
✗ categories
... 42 tables nữa (IRRELEVANT)
```

**LLM có thể bị "confused" bởi quá nhiều thông tin không liên quan:**
- "Có phải user muốn query về orders không nhỉ?"
- "Có cần JOIN với products không?"
- **→ Tăng khả năng sinh SQL sai!**

**Với RAG + Summarization:**
```
Vector search → Chỉ lấy: users

Schema Context:
Table: users
Columns:
  - id (INTEGER)
  - email (TEXT)
  - created_at (DATETIME)

Schema Summary:
"This query requires the 'users' table.
The key column is 'created_at' for filtering by month."
```

**Benefit:** LLM focus vào đúng context → Accuracy cao hơn

---

### **3. Cost Optimization (Tối Ưu Chi Phí)**

**LLM APIs tính phí theo tokens:**

| Approach | Input Tokens | Cost (ví dụ @$0.001/1k tokens) |
|----------|-------------|-------------------------------|
| Full Schema (50 tables) | 25,000 | $0.025 / request |
| RAG (3 tables) | 1,700 | $0.0017 / request |

**Với 10,000 requests/tháng:**
- Full Schema: $250/month
- RAG: $17/month
- **Tiết kiệm: $233/month (93%)**

**Benefit:** Giảm chi phí đáng kể khi scale

---

### **4. Latency Reduction (Giảm Độ Trễ)**

**Số tokens ảnh hưởng trực tiếp đến latency:**

```
Full Schema (25k tokens):
  LLM processing time: ~3-5 seconds

RAG (1.7k tokens):
  Vector search: ~0.1 seconds
  LLM processing: ~0.5 seconds
  Total: ~0.6 seconds

→ Nhanh hơn 5-8x!
```

**Benefit:** Better UX, faster response time

---

### **5. Improved Output Quality (Chất Lượng Tốt Hơn)**

**Nghiên cứu cho thấy:**
- LLMs perform better với focused context
- Quá nhiều context → "lost in the middle" problem
- Schema summary giúp LLM hiểu "big picture"

**Ví dụ:**

**Không có summary:**
```
Schema: [50 table definitions...]

User: "Tổng revenue tháng này"

LLM: "Không chắc revenue ở table nào...
      Có thể là orders.total?
      Hay payments.amount?
      Hay invoices.revenue?"
```

**Có summary:**
```
Schema Context: [orders, payments tables]

Schema Summary:
"Revenue data is in 'orders.total_amount'.
Payment status is tracked in 'payments.status'.
Join via orders.id = payments.order_id"

User: "Tổng revenue tháng này"

LLM: "Rõ ràng! Query từ orders.total_amount,
      filter by created_at và payment_status = 'completed'"
```

**Benefit:** SQL chính xác hơn, ít lỗi hơn

---

### **6. Scalability (Khả Năng Mở Rộng)**

**Khi database phát triển:**

| Database Size | Full Schema Approach | RAG Approach |
|--------------|---------------------|--------------|
| 10 tables | ✅ OK | ✅ OK |
| 50 tables | ⚠️ Slow, expensive | ✅ OK |
| 200 tables | ❌ Không khả thi | ✅ OK |
| 1000+ tables | ❌ Impossible | ✅ OK (với tuning) |

**Benefit:** Hệ thống có thể scale với database lớn

---

### **7. Multi-Database Support (Tương Lai)**

**Nếu muốn support nhiều databases:**

```python
# Không có RAG:
full_schema = (
    postgres_schema +    # 100 tables
    mysql_schema +       # 80 tables
    mongodb_schema +     # 50 collections
    redis_schema         # 20 key patterns
)
# → Không khả thi!

# Với RAG:
# Vector search tự động tìm đúng database + tables
relevant_docs = vector_search(query)  # Có thể từ bất kỳ DB nào
# → Scalable!
```

---

## 🔬 Deep Dive: Tại Sao Cần Cả RAG VÀ Summarization?

### **Tại sao không chỉ RAG (không summarize)?**

```python
# Chỉ RAG, không summarize
schema_context = retrieve_relevant_tables(query)  # 3 tables

prompt = f"Schema:\n{schema_context}\n\nQuery: {user_query}"
```

**Vấn đề:**
- LLM vẫn phải đọc raw schema definitions
- Không có "big picture" về relationships
- Có thể miss implicit joins

**Với Summarization:**
```python
schema_summary = """
Key Tables: users, orders
Key Columns: users.id, orders.user_id, orders.total
Relationships: users.id = orders.user_id (one-to-many)
Context: Orders belong to users. Use LEFT JOIN for users without orders.
"""
```

**Benefit:** LLM hiểu context nhanh hơn, generate SQL chính xác hơn

---

### **Tại sao không chỉ Summarization (không RAG)?**

```python
# Tóm tắt toàn bộ schema
schema_summary = summarize_entire_database()
```

**Vấn đề:**
- Summary của 50 tables vẫn dài
- Mất details quan trọng (column names, data types)
- SQL generation cần exact column names

**Benefit của RAG + Summarization:**
- RAG: Lấy exact schema details cho relevant tables
- Summarization: Cung cấp high-level context
- **→ Best of both worlds!**

---

## 📈 Performance Comparison

### **Thực Nghiệm:**

| Metric | Full Schema | RAG Only | RAG + Summary |
|--------|------------|----------|---------------|
| **Accuracy** | 72% | 85% | **92%** ✅ |
| **Avg Latency** | 4.2s | 1.1s | **0.8s** ✅ |
| **Cost/1k queries** | $25 | $2.5 | **$1.7** ✅ |
| **Token Usage** | 25k | 2k | **1.7k** ✅ |
| **Scalability** | ❌ | ✅ | ✅✅ |

*(Số liệu minh họa, thực tế phụ thuộc database size & query complexity)*

---

## 🎓 Best Practices

### **Khi Nào Cần Schema Summarization?**

✅ **CẦN khi:**
- Database có > 10 tables
- Schema phức tạp với nhiều relationships
- Cần optimize cost & latency
- Production system với high traffic

❌ **KHÔNG CẦN khi:**
- Database rất nhỏ (< 5 tables)
- Prototype/POC
- Schema cực kỳ đơn giản
- Development/testing environment

---

### **Cách Optimize Schema Summarization:**

**1. Tune số lượng tables retrieved (k value):**
```python
# Ít tables → Faster, nhưng có thể miss context
retriever = vectorstore.as_retriever(search_kwargs={"k": 2})

# Nhiều tables → Slower, nhưng đầy đủ hơn
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# Default: k=3 (good balance)
```

**2. Caching:**
```python
# Cache schema summaries cho queries phổ biến
@lru_cache(maxsize=100)
def get_schema_summary(query_pattern):
    ...
```

**3. Incremental Updates:**
```python
# Chỉ rebuild index khi schema thay đổi
if schema_changed():
    rebuild_vector_index()
```

---

## 🔄 Alternative Approaches

### **Approach A: Hybrid (Dynamic Decision)**
```python
if database.table_count < 10:
    # Small DB → Use full schema
    return full_schema
else:
    # Large DB → Use RAG + Summarization
    return rag_and_summarize()
```

### **Approach B: Hierarchical Summarization**
```python
# Level 1: Database-level summary
db_summary = "E-commerce DB with users, products, orders"

# Level 2: Table-level summary (RAG)
table_summary = "orders table tracks purchases"

# Level 3: Column-level details
column_details = "orders.total (DECIMAL), orders.created_at (TIMESTAMP)"
```

### **Approach C: Query-Specific Schemas**
```python
# Pre-define schema subsets cho common query types
schemas = {
    "user_analytics": ["users", "sessions", "events"],
    "sales_reports": ["orders", "products", "payments"],
    "inventory": ["products", "stock", "warehouses"]
}

query_type = classify_query(user_query)
relevant_schema = schemas[query_type]
```

---

## 💡 Kết Luận

### **Tại Sao Cần Schema Summarization:**

1. **Technical:** Context window limits, token costs
2. **Quality:** Better focus → Higher accuracy
3. **Performance:** Faster inference, lower latency
4. **Economics:** Significant cost savings at scale
5. **Scalability:** Works with large, complex databases

### **Trade-off:**

**Pros:**
- ✅ Efficient
- ✅ Scalable
- ✅ Cost-effective
- ✅ Better accuracy

**Cons:**
- ❌ Thêm 1 bước (RAG retrieval + LLM summarization)
- ❌ Có thể miss relevant tables nếu vector search không tốt
- ❌ Cần maintain vector index

**Verdict:** Với production systems và large databases, benefits >> costs

---

## 🚀 Trong Code Hiện Tại

**File:** `app/agents/schema_agent.py`

```python
async def schema_agent_node(state: GlobalState):
    # 1. RAG Retrieval
    retriever = _vectorstore.as_retriever(search_kwargs={"k": 3})
    results = retriever.invoke(query)  # Vector search

    # 2. Extract schema context
    schema_context = "\n".join([doc.page_content for doc in results])

    # 3. LLM Summarization
    summary: SchemaSummary = await chain.ainvoke({
        "query": query,
        "schema_context": schema_context
    })

    # 4. Return both raw context + summary
    return {
        "schema_context": schema_context,      # ← Raw schema (for SQL generation)
        "schema_summary": summary.summary_text  # ← High-level summary (for LLM understanding)
    }
```

**→ Query Generation Agent nhận CẢ HAI:**
- `schema_context`: Exact table/column definitions
- `schema_summary`: High-level understanding

**→ Best of both worlds! ✨**

---

## 📚 Further Reading

- [RAG for Structured Data](https://www.pinecone.io/learn/rag-structured-data/)
- [Optimizing LLM Context Windows](https://www.anthropic.com/index/claude-2-1-prompting)
- [Lost in the Middle Problem](https://arxiv.org/abs/2307.03172)
- [Text-to-SQL Benchmarks](https://yale-lily.github.io/spider)
