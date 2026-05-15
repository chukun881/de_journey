# 📅 Day 61 — Monday, 14 July 2026
# 🔍 Portfolio Enhancement: Add AI-Powered Query to MakanExpress

---

## 🎯 Today's Goal

You've learned APIs and LLMs. Today you enhance your MakanExpress portfolio by adding an AI-powered natural language query feature. Instead of writing SQL, users can ask "What's the most popular dish in Singapore?" and get an answer.

This is a killer portfolio feature — it shows employers you can combine traditional data engineering (SQL, databases) with modern AI (LLMs, RAG).

---

## ☀️ Morning Block (2 hours): Build the AI Query Tool

### Architecture

```
User types: "What's the cheapest dish in KL?"
       │
       ▼
┌──────────────────┐
│ 1. Schema Lookup  │ ← Get table/column info from data catalog
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. SQL Generation  │ ← LLM converts question to SQL
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. Execute SQL     │ ← Run query on PostgreSQL
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Format Answer   │ ← LLM formats results in natural language
└────────┬─────────┘
         │
         ▼
"The cheapest dish in KL is Roti Canai at RM 4.00 
 from Roti Valentine."
```

### Implementation

```python
# src/ai_query/engine.py

import psycopg2
import json
from openai import OpenAI
import logging

logger = logging.getLogger(__name__)

class AIQueryEngine:
    """Natural language to SQL query engine."""
    
    def __init__(self, db_params, api_key):
        self.client = OpenAI(api_key=api_key)
        self.db_params = db_params
        
        self.schema_context = """
Tables in MakanExpress database:

1. orders (id, customer_id, restaurant_id, order_date, total_amount, status, delivery_city)
2. restaurants (id, name, cuisine, city, rating, avg_delivery_time_min)
3. customers (id, name, email, city, signup_date)
4. order_items (id, order_id, menu_item, quantity, unit_price)
5. menu_items_vectors (id, name, description, restaurant, price_sgd, cuisine, embedding)

Status values: pending, preparing, on_the_way, delivered, cancelled
Cities: Singapore, Kuala Lumpur, Penang, Johor Bahru
Currency: SGD for Singapore, MYR for others (but stored as numeric)
"""
    
    def _get_connection(self):
        return psycopg2.connect(**self.db_params)
    
    def _generate_sql(self, question):
        """Convert natural language to SQL."""
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {
                    "role": "system",
                    "content": f"""You are a SQL expert. Convert the user's question to a PostgreSQL query.

Database Schema:
{self.schema_context}

Rules:
- Return ONLY the SQL query, no explanation
- Use proper PostgreSQL syntax
- Limit results to 10 rows max
- Use COALESCE for potentially null values
- For date queries, use SGT timezone (UTC+8)"""
                },
                {"role": "user", "content": question}
            ],
            max_tokens=300,
            temperature=0.1
        )
        
        sql = response.choices[0].message.content.strip()
        if sql.startswith("```"):
            sql = sql.split("\n", 1)[1].rsplit("```", 1)[0].strip()
        return sql
    
    def _execute_sql(self, sql):
        """Execute SQL safely (read-only)."""
        # Safety: only allow SELECT statements
        if not sql.strip().upper().startswith("SELECT"):
            raise ValueError("Only SELECT queries are allowed")
        
        # Block dangerous operations
        dangerous = ["DROP", "DELETE", "UPDATE", "INSERT", "ALTER", "CREATE", "TRUNCATE"]
        for word in dangerous:
            if word in sql.upper():
                raise ValueError(f"Dangerous operation blocked: {word}")
        
        conn = self._get_connection()
        cur = conn.cursor()
        
        try:
            cur.execute(sql)
            columns = [desc[0] for desc in cur.description]
            rows = cur.fetchall()
            return {"columns": columns, "rows": rows, "row_count": len(rows)}
        finally:
            cur.close()
            conn.close()
    
    def _format_answer(self, question, sql, results):
        """Format query results as natural language."""
        if results["row_count"] == 0:
            return "No data found for your question."
        
        # Convert rows to readable format
        rows_text = []
        for row in results["rows"]:
            row_dict = dict(zip(results["columns"], row))
            rows_text.append(str(row_dict))
        
        data_text = "\n".join(rows_text[:10])  # Max 10 rows
        
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {
                    "role": "system",
                    "content": "Answer the user's question based on the SQL query results. Be concise and natural. Mention specific values from the data."
                },
                {
                    "role": "user",
                    "content": f"Question: {question}\n\nSQL: {sql}\n\nResults ({results['row_count']} rows):\n{data_text}"
                }
            ],
            max_tokens=200,
            temperature=0.3
        )
        
        return response.choices[0].message.content
    
    def query(self, question):
        """Full pipeline: question → SQL → execute → answer."""
        logger.info(f"Processing: {question}")
        
        # Generate SQL
        sql = self._generate_sql(question)
        logger.info(f"Generated SQL: {sql}")
        
        # Execute
        try:
            results = self._execute_sql(sql)
        except Exception as e:
            return {
                "question": question,
                "sql": sql,
                "answer": f"Query failed: {e}",
                "status": "error"
            }
        
        # Format answer
        answer = self._format_answer(question, sql, results)
        
        return {
            "question": question,
            "sql": sql,
            "answer": answer,
            "row_count": results["row_count"],
            "status": "success"
        }
```

### Usage

```python
# demo.py

db_params = {
    "host": "localhost", "port": "5432",
    "dbname": "makanexpress", "user": "postgres", "password": "postgres"
}

engine = AIQueryEngine(db_params, "YOUR_API_KEY")

# Ask questions!
questions = [
    "What are the top 5 restaurants by number of orders?",
    "What's the average order amount in Singapore vs KL?",
    "Which cuisine type is most popular on weekends?",
    "How many new customers signed up this month?",
    "What's the most expensive dish on the menu?"
]

for q in questions:
    result = engine.query(q)
    print(f"❓ {result['question']}")
    print(f"📝 SQL: {result['sql'][:80]}...")
    print(f"💬 {result['answer']}")
    print(f"📊 {result.get('row_count', 0)} rows returned")
    print("-" * 60)
```

---

### Morning Exercises

**Exercise 1:** Add a query history feature. Store each question, generated SQL, and answer in a `query_history` table. Write a function `get_popular_questions()` that returns the most-asked questions.

<details>
<summary>🔑 Answer</summary>

```python
def create_query_history_table():
    conn = psycopg2.connect(**self.db_params)
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS ai_query_history (
            id SERIAL PRIMARY KEY,
            question TEXT NOT NULL,
            generated_sql TEXT,
            answer TEXT,
            row_count INT,
            status VARCHAR(20),
            execution_time_ms INT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
    """)
    conn.commit()
    cur.close()
    conn.close()

def save_to_history(self, question, sql, answer, row_count, status, exec_time_ms):
    conn = psycopg2.connect(**self.db_params)
    cur = conn.cursor()
    cur.execute("""
        INSERT INTO ai_query_history (question, generated_sql, answer, row_count, status, execution_time_ms)
        VALUES (%s, %s, %s, %s, %s, %s)
    """, (question, sql, answer, row_count, status, exec_time_ms))
    conn.commit()
    cur.close()
    conn.close()

def get_popular_questions(self, limit=10):
    conn = psycopg2.connect(**self.db_params)
    cur = conn.cursor()
    cur.execute("""
        SELECT question, COUNT(*) as ask_count, AVG(execution_time_ms) as avg_time_ms
        FROM ai_query_history
        WHERE status = 'success'
        GROUP BY question
        ORDER BY ask_count DESC
        LIMIT %s
    """, (limit,))
    results = cur.fetchall()
    cur.close()
    conn.close()
    return results
```

</details>

**Exercise 2:** Add SQL validation. Before executing, check the generated SQL for common mistakes (missing WHERE clause on DELETE/UPDATE, SELECT *, Cartesian products). Only warn — don't block.

<details>
<summary>🔑 Answer</summary>

```python
def validate_sql(self, sql):
    """Check generated SQL for common issues."""
    warnings = []
    sql_upper = sql.upper()
    
    # Check for SELECT *
    if "SELECT *" in sql_upper:
        warnings.append("⚠️ SELECT * used — consider specifying columns for performance")
    
    # Check for missing WHERE on large table scans
    if "FROM orders" in sql_upper and "WHERE" not in sql_upper:
        warnings.append("⚠️ No WHERE clause on orders table — could be slow on large data")
    
    # Check for potential Cartesian product (multiple tables in FROM without JOIN)
    if "," in sql_upper.split("FROM")[1].split("WHERE")[0] if "FROM" in sql_upper else "":
        if "JOIN" not in sql_upper:
            warnings.append("⚠️ Possible Cartesian product — use explicit JOINs")
    
    # Check for very large LIMIT
    import re
    limit_match = re.search(r'LIMIT\s+(\d+)', sql_upper)
    if limit_match and int(limit_match.group(1)) > 1000:
        warnings.append("⚠️ Large LIMIT — consider reducing for performance")
    
    # No LIMIT at all
    if "LIMIT" not in sql_upper:
        warnings.append("💡 No LIMIT clause — adding LIMIT 100 for safety")
        sql = sql.rstrip(";") + " LIMIT 100;"
    
    return sql, warnings
```

</details>

**Exercise 3:** Build a simple command-line interface (CLI) for the AI query engine. Loop: prompt for question, show SQL and answer, ask if user wants to ask another.

<details>
<summary>🔑 Answer</summary>

```python
def cli_mode():
    """Interactive CLI for AI query engine."""
    print("=" * 50)
    print("🤖 MakanExpress AI Query Engine")
    print("=" * 50)
    print("Ask questions about your data in plain English!")
    print("Type 'quit' to exit, 'history' to see past queries\n")
    
    engine = AIQueryEngine(db_params, "YOUR_API_KEY")
    
    while True:
        question = input("❓ Your question: ").strip()
        
        if not question:
            continue
        if question.lower() == 'quit':
            print("👋 Goodbye!")
            break
        if question.lower() == 'history':
            popular = engine.get_popular_questions()
            for q, count, avg_time in popular:
                print(f"  [{count}x] {q} (avg {avg_time:.0f}ms)")
            continue
        
        import time
        start = time.time()
        result = engine.query(question)
        elapsed = (time.time() - start) * 1000
        
        print(f"\n📝 Generated SQL:")
        print(f"   {result['sql']}")
        print(f"\n💬 Answer:")
        print(f"   {result['answer']}")
        print(f"\n⏱️  {elapsed:.0f}ms | {result.get('row_count', 0)} rows")
        print()

if __name__ == "__main__":
    cli_mode()
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Add to Portfolio + README

### Create the Portfolio Entry

```bash
mkdir -p ai-query-tool
cp src/ai_query/engine.py ai-query-tool/
cp demo.py ai-query-tool/
```

### README for AI Query Tool

```markdown
# 🤖 MakanExpress AI Query Tool

Natural language interface for the MakanExpress food delivery database. 
Ask questions in English, get answers from your data.

## How It Works

1. User asks a question in plain English
2. LLM generates a PostgreSQL query
3. Query executes against the database
4. LLM formats results as a natural language answer

## Example Queries

| Question | Generated SQL |
|----------|--------------|
| "What's the most popular cuisine in Singapore?" | SELECT cuisine, COUNT(*) FROM orders JOIN restaurants... |
| "Show me orders over $50 this week" | SELECT * FROM orders WHERE total_amount > 50... |

## Tech Stack
- Python, OpenAI API (GPT-4o-mini), PostgreSQL, pgvector
- SQL validation, query history, safety checks

## Setup
\`\`\`bash
pip install openai psycopg2
export OPENAI_API_KEY=your-key
python demo.py
\`\`\`

## Safety Features
- Read-only queries (no INSERT/UPDATE/DELETE)
- SQL validation and warnings
- Query history tracking
```

---

### Connect to Main Portfolio

Add to your GitHub Pages site:

```markdown
### 🤖 AI Query Tool
Ask questions about your data in plain English. Powered by LLM + PostgreSQL.
[View Repo →](https://github.com/YOUR_USERNAME/ai-query-tool)
```

---

## 🌙 Evening (1 hour): Test + Push + Reflect

### End-to-End Test

```python
# test_ai_query.py

def test_ai_query():
    engine = AIQueryEngine(db_params, "YOUR_API_KEY")
    
    # Test 1: Simple aggregation
    r = engine.query("How many orders are there?")
    assert r["status"] == "success"
    print(f"✅ Test 1: {r['answer']}")
    
    # Test 2: Filter
    r = engine.query("Show me orders from Singapore")
    assert r["status"] == "success"
    print(f"✅ Test 2: {r['answer']}")
    
    # Test 3: Join
    r = engine.query("What are the top 3 restaurants by revenue?")
    assert r["status"] == "success"
    print(f"✅ Test 3: {r['answer']}")
    
    # Test 4: Safety (should block)
    r = engine.query("Delete all orders")
    assert r["status"] == "error"
    print(f"✅ Test 4: Correctly blocked dangerous query")
    
    print("\n✅ All tests passed!")

test_ai_query()
```

### 📝 Today's Checklist

- [ ] Built an AI-powered natural language query engine
- [ ] SQL generation from natural language using LLM
- [ ] Safe SQL execution with validation
- [ ] Natural language answer formatting
- [ ] Added query history feature
- [ ] Created CLI interface
- [ ] Added to portfolio with README
- [ ] Pushed to GitHub
- [ ] Updated GitHub Pages site

---

*Day 61 complete!* 🤖
