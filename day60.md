# 📅 Day 60 — Sunday, 13 July 2026
# 🔍 RAG Concepts: Embeddings, Vector Search, pgvector

---

## 🎯 Today's Goal

RAG (Retrieval-Augmented Generation) is how companies build AI systems that "know" their own data. Instead of relying on the LLM's general knowledge, you search YOUR data first, then pass relevant results to the LLM for a grounded answer.

**Real-world example:** GrabFood builds a chatbot. When you ask "What's the best chicken rice near me?", RAG:
1. Searches Grab's restaurant database for chicken rice → retrieval
2. Sends those results + your question to the LLM → augmented generation
3. Returns an answer based on REAL data, not hallucinations

**Today you learn:** embeddings, vector similarity search, and how to store vectors in PostgreSQL using pgvector.

---

## ☀️ Morning Block (2 hours): Embeddings + Vector Concepts

### What are Embeddings?

An embedding is a **numerical representation of text** — a list of numbers (a vector) that captures the meaning of the text.

```
"Singapore chicken rice is delicious" → [0.12, -0.34, 0.56, 0.78, ...]
"Nasi lemak in KL tastes amazing"     → [0.11, -0.32, 0.54, 0.77, ...]
"The stock market crashed today"       → [-0.89, 0.45, -0.12, 0.33, ...]
```

**Key insight:** Similar meanings → similar vectors. The chicken rice and nasi lemak vectors are close together (both about food). The stock market vector is far away (completely different topic).

### Visual Intuition

```
                    Food
                     ↑
    "chicken rice" ●  ● "nasi lemak"
                   ●
            "roti canai"
                   
                   
    "stock market" ●          ● "Python ETL"
                   ↓
                 Finance     Tech
```

We measure similarity using **cosine similarity** (0 to 1):
- 1.0 = identical meaning
- 0.5 = somewhat related
- 0.0 = completely unrelated

---

### Generate Embeddings with OpenAI

```python
from openai import OpenAI

client = OpenAI(api_key="YOUR_API_KEY")

def get_embedding(text, model="text-embedding-3-small"):
    """Generate embedding for a text string."""
    response = client.embeddings.create(
        input=text,
        model=model
    )
    return response.data[0].embedding

# Generate embeddings for food items
texts = [
    "Singapore chicken rice with chili sauce",
    "Nasi lemak with sambal and fried chicken",
    "Bak kut teh pork rib soup in Klang",
    "Python ETL pipeline using pandas",
    "SQL query optimization with indexes",
]

embeddings = {}
for text in texts:
    emb = get_embedding(text)
    embeddings[text] = emb
    print(f"'{text[:40]}...' → vector with {len(emb)} dimensions")
```

---

### Cosine Similarity

```python
import numpy as np

def cosine_similarity(vec_a, vec_b):
    """Calculate cosine similarity between two vectors."""
    a = np.array(vec_a)
    b = np.array(vec_b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Compare food items
items = list(embeddings.items())

print("Similarity Matrix:")
print(f"{'':>45}", end="")
for label, _ in items:
    print(f"{label[:20]:>20}", end="")
print()

for label_a, emb_a in items:
    print(f"{label_a[:45]:>45}", end="")
    for label_b, emb_b in items:
        sim = cosine_similarity(emb_a, emb_b)
        print(f"{sim:>20.3f}", end="")
    print()
```

**Expected output:** Chicken rice and nasi lemak will have high similarity (~0.7-0.8). Food and Python ETL will have low similarity (~0.1-0.3).

---

### Install pgvector on PostgreSQL

```bash
# If using local PostgreSQL
# Mac: brew install pgvector
# Ubuntu: sudo apt install postgresql-16-pgvector

# Connect to PostgreSQL and enable the extension
psql -U postgres -d makanexpress -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

**If using Neon (cloud):** pgvector is already available! Just run `CREATE EXTENSION vector;`

---

### Store Embeddings in PostgreSQL

```python
import psycopg2
from openai import OpenAI
import numpy as np

client = OpenAI(api_key="YOUR_API_KEY")

# Create table with vector column
def create_vector_table():
    conn = psycopg2.connect(
        host="localhost", port=5432,
        dbname="makanexpress", user="postgres", password="postgres"
    )
    cur = conn.cursor()
    
    cur.execute("""
        CREATE TABLE IF NOT EXISTS menu_items_vectors (
            id SERIAL PRIMARY KEY,
            name VARCHAR(200) NOT NULL,
            description TEXT,
            restaurant VARCHAR(200),
            price_sgd DECIMAL(10,2),
            cuisine VARCHAR(50),
            embedding vector(1536),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
    """)
    
    conn.commit()
    cur.close()
    conn.close()
    print("✅ Table created with vector column")

# Insert menu items with embeddings
def insert_menu_items():
    menu_items = [
        {
            "name": "Hainanese Chicken Rice",
            "description": "Poached chicken with fragrant rice, chili sauce, and ginger paste. Singapore's national dish.",
            "restaurant": "Tian Tian Hainanese Chicken Rice",
            "price_sgd": 5.00,
            "cuisine": "Chinese"
        },
        {
            "name": "Nasi Lemak Special",
            "description": "Coconut rice with fried chicken, sambal, cucumber, peanuts, and anchovies. Malaysia's favorite.",
            "restaurant": "Village Park Restaurant",
            "price_sgd": 8.50,
            "cuisine": "Malay"
        },
        {
            "name": "Bak Kut Teh",
            "description": "Pork rib soup simmered with herbs and garlic. A hearty Chinese soup from Klang.",
            "restaurant": "Song Fa Bak Kut Teh",
            "price_sgd": 12.00,
            "cuisine": "Chinese"
        },
        {
            "name": "Roti Canai with Curry",
            "description": "Flaky flatbread served with dhal curry. A beloved Malaysian breakfast staple.",
            "restaurant": "Roti Valentine",
            "price_sgd": 4.00,
            "cuisine": "Indian-Muslim"
        },
        {
            "name": "Laksa",
            "description": "Spicy noodle soup with coconut milk, shrimp, and tofu puffs. A Penang specialty.",
            "restaurant": "Penang Road Famous Laksa",
            "price_sgd": 6.00,
            "cuisine": "Nyonya"
        },
        {
            "name": "Char Kway Teow",
            "description": "Stir-fried flat rice noodles with shrimp, cockles, bean sprouts, and egg. Penang street food classic.",
            "restaurant": "Sister's Char Kway Teow",
            "price_sgd": 7.00,
            "cuisine": "Chinese"
        },
    ]
    
    conn = psycopg2.connect(
        host="localhost", port=5432,
        dbname="makanexpress", user="postgres", password="postgres"
    )
    cur = conn.cursor()
    
    for item in menu_items:
        # Generate embedding from name + description
        text_to_embed = f"{item['name']}: {item['description']}"
        embedding = get_embedding(text_to_embed)
        embedding_str = str(embedding)
        
        cur.execute("""
            INSERT INTO menu_items_vectors (name, description, restaurant, price_sgd, cuisine, embedding)
            VALUES (%s, %s, %s, %s, %s, %s::vector)
        """, (item["name"], item["description"], item["restaurant"],
              item["price_sgd"], item["cuisine"], embedding_str))
    
    conn.commit()
    cur.close()
    conn.close()
    print(f"✅ Inserted {len(menu_items)} menu items with embeddings")
```

---

### Vector Similarity Search

```python
def search_menu(query, limit=3):
    """Search menu items using vector similarity."""
    # Generate embedding for the search query
    query_embedding = get_embedding(query)
    embedding_str = str(query_embedding)
    
    conn = psycopg2.connect(
        host="localhost", port=5432,
        dbname="makanexpress", user="postgres", password="postgres"
    )
    cur = conn.cursor()
    
    # Cosine similarity search
    cur.execute("""
        SELECT name, restaurant, price_sgd, cuisine,
               1 - (embedding <=> %s::vector) AS similarity
        FROM menu_items_vectors
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (embedding_str, embedding_str, limit))
    
    results = cur.fetchall()
    cur.close()
    conn.close()
    
    print(f"\n🔍 Search: '{query}'")
    print("-" * 60)
    for name, restaurant, price, cuisine, similarity in results:
        print(f"  {name} ({cuisine})")
        print(f"    at {restaurant} — SGD ${price}")
        print(f"    Similarity: {similarity:.3f}")
        print()
    
    return results

# Test searches
search_menu("spicy noodle soup")
search_menu("cheap breakfast under 5 dollars")
search_menu("something with chicken")
search_menu("traditional Malaysian food")
```

---

### Morning Exercises

**Exercise 1:** Write a function that takes 5 food descriptions and returns a similarity matrix showing which foods are most similar to each other.

<details>
<summary>🔑 Answer</summary>

```python
import numpy as np

def build_similarity_matrix(texts):
    """Build a similarity matrix for a list of texts."""
    # Generate embeddings
    embeddings = []
    for text in texts:
        emb = get_embedding(text)
        embeddings.append(emb)
    
    # Build matrix
    n = len(texts)
    matrix = np.zeros((n, n))
    
    for i in range(n):
        for j in range(n):
            matrix[i][j] = cosine_similarity(embeddings[i], embeddings[j])
    
    # Print formatted
    labels = [t[:25] for t in texts]
    print(f"{'':>27}", end="")
    for label in labels:
        print(f"{label:>27}", end="")
    print()
    
    for i in range(n):
        print(f"{labels[i]:>27}", end="")
        for j in range(n):
            print(f"{matrix[i][j]:>27.3f}", end="")
        print()
    
    return matrix

# Test
foods = [
    "Hainanese chicken rice with chili sauce",
    "Nasi lemak with fried chicken and sambal",
    "Spicy laksa noodle soup",
    "Python data pipeline with pandas",
    "SQL query for sales report"
]

matrix = build_similarity_matrix(foods)
```

</details>

**Exercise 2:** Create a table `restaurant_reviews_vectors` that stores review text, rating, and embedding. Write a function that finds the most similar reviews to a given query (e.g., "bad delivery experience").

<details>
<summary>🔑 Answer</summary>

```python
def create_reviews_table():
    conn = psycopg2.connect(
        host="localhost", port=5432,
        dbname="makanexpress", user="postgres", password="postgres"
    )
    cur = conn.cursor()
    
    cur.execute("""
        CREATE TABLE IF NOT EXISTS restaurant_reviews_vectors (
            id SERIAL PRIMARY KEY,
            review_text TEXT NOT NULL,
            rating INT CHECK (rating BETWEEN 1 AND 5),
            restaurant VARCHAR(200),
            sentiment VARCHAR(20),
            embedding vector(1536),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
    """)
    
    conn.commit()
    cur.close()
    conn.close()
    print("✅ Reviews table created")

def insert_sample_reviews():
    reviews = [
        {"text": "Delivery took 2 hours and food was cold. Very disappointed.", "rating": 1, "restaurant": "Chicken Rice Palace", "sentiment": "negative"},
        {"text": "Amazing food! Arrived in 20 minutes, still piping hot. 5 stars!", "rating": 5, "restaurant": "Nasi Lemak Corner", "sentiment": "positive"},
        {"text": "Food was okay but delivery person was rude. Won't order again.", "rating": 2, "restaurant": "Char Kway Teow House", "sentiment": "negative"},
        {"text": "The roti canai was perfect — crispy outside, soft inside. Best I've had in SG!", "rating": 5, "restaurant": "Roti King", "sentiment": "positive"},
        {"text": "Portion too small for the price. Felt like a scam.", "rating": 2, "restaurant": "Expensive Eats", "sentiment": "negative"},
    ]
    
    conn = psycopg2.connect(
        host="localhost", port=5432,
        dbname="makanexpress", user="postgres", password="postgres"
    )
    cur = conn.cursor()
    
    for r in reviews:
        embedding = get_embedding(r["text"])
        cur.execute("""
            INSERT INTO restaurant_reviews_vectors (review_text, rating, restaurant, sentiment, embedding)
            VALUES (%s, %s, %s, %s, %s::vector)
        """, (r["text"], r["rating"], r["restaurant"], r["sentiment"], str(embedding)))
    
    conn.commit()
    cur.close()
    conn.close()
    print(f"✅ Inserted {len(reviews)} reviews")

def search_reviews(query, limit=3):
    query_emb = get_embedding(query)
    
    conn = psycopg2.connect(
        host="localhost", port=5432,
        dbname="makanexpress", user="postgres", password="postgres"
    )
    cur = conn.cursor()
    
    cur.execute("""
        SELECT review_text, rating, restaurant, sentiment,
               1 - (embedding <=> %s::vector) AS similarity
        FROM restaurant_reviews_vectors
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (str(query_emb), str(query_emb), limit))
    
    results = cur.fetchall()
    cur.close()
    conn.close()
    
    print(f"\n🔍 Review search: '{query}'")
    print("-" * 60)
    for text, rating, restaurant, sentiment, sim in results:
        emoji = "😡" if rating <= 2 else ("😐" if rating == 3 else "😊")
        print(f"  {emoji} [{rating}/5] {sentiment} (sim: {sim:.3f})")
        print(f"  \"{text[:70]}...\"")
        print(f"  Restaurant: {restaurant}\n")
    
    return results

# Setup and test
create_reviews_table()
insert_sample_reviews()
search_reviews("bad delivery experience")
search_reviews("delicious food arrived quickly")
```

</details>

**Exercise 3:** Write a function that clusters menu items into groups based on embedding similarity. Use a simple threshold (similarity > 0.7 = same cluster).

<details>
<summary>🔑 Answer</summary>

```python
def cluster_by_similarity(texts, threshold=0.7):
    """Simple clustering based on cosine similarity threshold."""
    # Generate embeddings
    embeddings = [get_embedding(t) for t in texts]
    
    n = len(texts)
    clusters = []
    assigned = set()
    
    for i in range(n):
        if i in assigned:
            continue
        
        # Start a new cluster
        cluster = [i]
        assigned.add(i)
        
        # Find all similar items
        for j in range(i + 1, n):
            if j in assigned:
                continue
            sim = cosine_similarity(embeddings[i], embeddings[j])
            if sim >= threshold:
                cluster.append(j)
                assigned.add(j)
        
        clusters.append(cluster)
    
    # Print clusters
    for ci, cluster in enumerate(clusters):
        print(f"\nCluster {ci + 1}:")
        for idx in cluster:
            print(f"  - {texts[idx][:60]}")
    
    return clusters

# Test
items = [
    "Chicken rice with ginger and chili",
    "Nasi lemak coconut rice with sambal",
    "Roti canai with dhal curry",
    "Toasted bread with kaya and butter",
    "Spicy tom yum noodle soup",
    "Laksa with prawns and coconut milk",
    "Python script for ETL pipeline",
    "SQL window function for ranking",
]

clusters = cluster_by_similarity(items, threshold=0.65)
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Build a Simple RAG Pipeline

### The RAG Architecture

```
┌─────────────────────────────────────────────────────┐
│                  RAG PIPELINE                        │
│                                                      │
│  User Question: "What's the best chicken rice?"      │
│       │                                              │
│       ▼                                              │
│  ┌──────────────────┐                               │
│  │ 1. EMBED QUERY    │ ← Convert question to vector │
│  └────────┬─────────┘                               │
│           │                                          │
│           ▼                                          │
│  ┌──────────────────┐                               │
│  │ 2. VECTOR SEARCH  │ ← Find similar docs in PG    │
│  │   (pgvector)      │                              │
│  └────────┬─────────┘                               │
│           │ Top 3 results                            │
│           ▼                                          │
│  ┌──────────────────┐                               │
│  │ 3. AUGMENT PROMPT │ ← Combine question + context │
│  └────────┬─────────┘                               │
│           │                                          │
│           ▼                                          │
│  ┌──────────────────┐                               │
│  │ 4. LLM GENERATE   │ ← Get grounded answer        │
│  └────────┬─────────┘                               │
│           │                                          │
│           ▼                                          │
│  Answer: "Based on our data, Tian Tian's chicken     │
│  rice at Maxwell is rated 4.5 stars and costs        │
│  SGD 5. It's known for its tender chicken and        │
│  fragrant rice."                                     │
└─────────────────────────────────────────────────────┘
```

---

### Building the RAG Pipeline

```python
# src/rag/pipeline.py

from openai import OpenAI
import psycopg2
import json

class MakanExpressRAG:
    """Simple RAG pipeline for MakanExpress menu data."""
    
    def __init__(self, db_params, api_key):
        self.client = OpenAI(api_key=api_key)
        self.db_params = db_params
    
    def _get_connection(self):
        return psycopg2.connect(**self.db_params)
    
    def embed(self, text):
        """Generate embedding for text."""
        response = self.client.embeddings.create(
            input=text,
            model="text-embedding-3-small"
        )
        return response.data[0].embedding
    
    def retrieve(self, query, limit=3):
        """Find most relevant menu items using vector search."""
        query_embedding = self.embed(query)
        
        conn = self._get_connection()
        cur = conn.cursor()
        
        cur.execute("""
            SELECT name, description, restaurant, price_sgd, cuisine,
                   1 - (embedding <=> %s::vector) AS similarity
            FROM menu_items_vectors
            ORDER BY embedding <=> %s::vector
            LIMIT %s
        """, (str(query_embedding), str(query_embedding), limit))
        
        results = []
        for name, desc, restaurant, price, cuisine, similarity in cur.fetchall():
            results.append({
                "name": name,
                "description": desc,
                "restaurant": restaurant,
                "price_sgd": float(price),
                "cuisine": cuisine,
                "similarity": float(similarity)
            })
        
        cur.close()
        conn.close()
        return results
    
    def augment(self, query, retrieved_docs):
        """Build the augmented prompt with context."""
        context = "\n\n".join([
            f"Menu Item: {doc['name']}\n"
            f"Restaurant: {doc['restaurant']}\n"
            f"Price: SGD ${doc['price_sgd']:.2f}\n"
            f"Cuisine: {doc['cuisine']}\n"
            f"Description: {doc['description']}"
            for doc in retrieved_docs
        ])
        
        prompt = f"""Based on the following menu data, answer the user's question.
If the data doesn't contain enough information, say so.

MENU DATA:
{context}

USER QUESTION: {query}

Answer based ONLY on the provided menu data. Include specific prices and restaurant names."""

        return prompt
    
    def generate(self, prompt):
        """Generate answer using LLM."""
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a helpful food recommendation assistant for MakanExpress, a food delivery service in Singapore and Malaysia."},
                {"role": "user", "content": prompt}
            ],
            max_tokens=300,
            temperature=0.3
        )
        return response.choices[0].message.content
    
    def query(self, question, limit=3):
        """Full RAG pipeline: retrieve → augment → generate."""
        # Step 1: Retrieve relevant documents
        docs = self.retrieve(question, limit=limit)
        
        if not docs:
            return "I couldn't find any relevant menu items. Please try a different question."
        
        # Step 2: Augment prompt with context
        prompt = self.augment(question, docs)
        
        # Step 3: Generate answer
        answer = self.generate(prompt)
        
        return {
            "question": question,
            "answer": answer,
            "sources": docs,
            "retrieved_count": len(docs)
        }

# Usage
db_params = {
    "host": "localhost", "port": 5432",
    "dbname": "makanexpress", "user": "postgres", "password": "postgres"
}

rag = MakanExpressRAG(db_params, "YOUR_API_KEY")

# Ask questions!
questions = [
    "What's the best chicken rice and how much does it cost?",
    "I want something spicy with noodles. What do you recommend?",
    "What's the cheapest breakfast option?",
    "I'm craving traditional Malaysian food. What's good?",
]

for q in questions:
    result = rag.query(q)
    print(f"❓ {result['question']}")
    print(f"💬 {result['answer']}")
    print(f"📚 Sources: {', '.join(s['name'] for s in result['sources'])}")
    print("-" * 60)
```

---

### Data Engineering Use Cases for RAG

| Use Case | How It Works | Example |
|----------|-------------|---------|
| **Data Catalog Search** | Embed table/column descriptions → search with natural language | "Find the table with customer order totals" |
| **Document Q&A** | Embed docs/pdfs → ask questions about them | "What's our SLA for data freshness?" |
| **Anomaly Explanation** | Embed anomaly descriptions → find similar past incidents | "Why did orders spike on Monday?" |
| **SQL Assistant** | Embed schema + sample queries → help write SQL | "How do I join orders with customers?" |
| **Data Lineage Query** | Embed pipeline descriptions → trace data flow | "Where does the revenue metric come from?" |

---

### Afternoon Exercises

**Exercise 4:** Extend the RAG pipeline to also search restaurant reviews. When answering a question, retrieve both menu items AND reviews to provide a more complete answer.

<details>
<summary>🔑 Answer</summary>

```python
def retrieve_multi_source(self, query, limit=3):
    """Search across both menu items and reviews."""
    query_embedding = self.embed(query)
    
    conn = self._get_connection()
    cur = conn.cursor()
    
    # Search menu items
    cur.execute("""
        SELECT 'menu' as source_type, name as title, description, 
               restaurant, price_sgd::text, cuisine,
               1 - (embedding <=> %s::vector) AS similarity
        FROM menu_items_vectors
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (str(query_embedding), str(query_embedding), limit))
    
    menu_results = cur.fetchall()
    
    # Search reviews
    cur.execute("""
        SELECT 'review' as source_type, review_text as title, review_text as description,
               restaurant, rating::text, sentiment,
               1 - (embedding <=> %s::vector) AS similarity
        FROM restaurant_reviews_vectors
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (str(query_embedding), str(query_embedding), limit))
    
    review_results = cur.fetchall()
    
    cur.close()
    conn.close()
    
    # Combine and sort by similarity
    all_results = []
    for row in menu_results:
        all_results.append({
            "type": "menu_item",
            "title": row[1],
            "detail": row[2],
            "restaurant": row[3],
            "extra": row[4],
            "similarity": float(row[6])
        })
    
    for row in review_results:
        all_results.append({
            "type": "review",
            "title": row[1][:100],
            "detail": row[2],
            "restaurant": row[3],
            "extra": f"Rating: {row[4]}/5, Sentiment: {row[5]}",
            "similarity": float(row[6])
        })
    
    all_results.sort(key=lambda x: x["similarity"], reverse=True)
    return all_results[:limit * 2]

def query_with_reviews(self, question, limit=3):
    """RAG query using both menu items and reviews."""
    docs = self.retrieve_multi_source(question, limit)
    
    context = "\n\n".join([
        f"[{d['type'].upper()}] {d['title']}\n"
        f"Restaurant: {d['restaurant']}\n"
        f"Details: {d['detail']}\n"
        f"Extra: {d['extra']}"
        for d in docs
    ])
    
    prompt = f"""Answer based on this data from MakanExpress:

{context}

Question: {question}

Use both menu items and reviews to give a comprehensive answer."""
    
    return self.generate(prompt)
```

</details>

**Exercise 5:** Build a "data catalog search" tool. Create a table that stores descriptions of your MakanExpress tables (orders, restaurants, customers, etc.) with embeddings. Write a function that lets you search "which table has delivery addresses?" and returns the right table.

<details>
<summary>🔑 Answer</summary>

```python
def create_data_catalog():
    """Create and populate a searchable data catalog."""
    conn = psycopg2.connect(**db_params)
    cur = conn.cursor()
    
    cur.execute("""
        CREATE TABLE IF NOT EXISTS data_catalog_vectors (
            id SERIAL PRIMARY KEY,
            table_name VARCHAR(100) NOT NULL,
            column_name VARCHAR(100),
            description TEXT NOT NULL,
            data_type VARCHAR(50),
            embedding vector(1536)
        );
    """)
    
    catalog_entries = [
        {"table": "orders", "column": "order_date", "desc": "Timestamp of when the food order was placed by the customer", "type": "TIMESTAMP"},
        {"table": "orders", "column": "total_amount", "desc": "Total order amount in the local currency (SGD or MYR)", "type": "DECIMAL"},
        {"table": "orders", "column": "delivery_address", "desc": "Delivery destination address for the food order", "type": "TEXT"},
        {"table": "orders", "column": "status", "desc": "Order status: pending, preparing, on_the_way, delivered, cancelled", "type": "VARCHAR"},
        {"table": "restaurants", "column": "cuisine_type", "desc": "Type of cuisine: Chinese, Malay, Indian, Western, etc.", "type": "VARCHAR"},
        {"table": "restaurants", "column": "rating", "desc": "Average customer rating for the restaurant from 1 to 5 stars", "type": "DECIMAL"},
        {"table": "customers", "column": "email", "desc": "Customer email address used for order confirmation", "type": "VARCHAR"},
        {"table": "customers", "column": "city", "desc": "City where the customer is located: Singapore, KL, Penang, etc.", "type": "VARCHAR"},
        {"table": "payments", "column": "payment_method", "desc": "How the customer paid: credit card, GrabPay, cash on delivery", "type": "VARCHAR"},
        {"table": "delivery", "column": "driver_id", "desc": "ID of the delivery driver assigned to this order", "type": "INT"},
    ]
    
    for entry in catalog_entries:
        text = f"{entry['table']}.{entry['column']}: {entry['desc']}"
        embedding = get_embedding(text)
        
        cur.execute("""
            INSERT INTO data_catalog_vectors (table_name, column_name, description, data_type, embedding)
            VALUES (%s, %s, %s, %s, %s::vector)
        """, (entry["table"], entry["column"], entry["desc"], entry["type"], str(embedding)))
    
    conn.commit()
    cur.close()
    conn.close()
    print(f"✅ Catalog: {len(catalog_entries)} entries")

def search_catalog(query, limit=3):
    """Search the data catalog with natural language."""
    query_embedding = get_embedding(query)
    
    conn = psycopg2.connect(**db_params)
    cur = conn.cursor()
    
    cur.execute("""
        SELECT table_name, column_name, description, data_type,
               1 - (embedding <=> %s::vector) AS similarity
        FROM data_catalog_vectors
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (str(query_embedding), str(query_embedding), limit))
    
    results = cur.fetchall()
    cur.close()
    conn.close()
    
    print(f"\n🔍 Catalog search: '{query}'")
    print("-" * 60)
    for table, column, desc, dtype, sim in results:
        print(f"  📋 {table}.{column} ({dtype})")
        print(f"     {desc}")
        print(f"     Similarity: {sim:.3f}\n")
    
    return results

# Test
create_data_catalog()
search_catalog("where are delivery addresses stored?")
search_catalog("how much did the customer pay?")
search_catalog("customer email for notifications")
search_catalog("which restaurant has the best rating?")
```

</details>

---

## 🌙 Evening (1 hour): Use Cases + Quick Reference

### 🔖 RAG + Vector Search Quick Reference

```
┌─────────────────────────────────────────────────────────┐
│          RAG & VECTOR SEARCH QUICK REFERENCE             │
├─────────────────────────────────────────────────────────┤
│ EMBEDDINGS                                               │
│ • Text → vector of numbers (e.g., 1536 dimensions)     │
│ • Similar text = similar vectors                        │
│ • OpenAI: text-embedding-3-small ($0.02/1M tokens)     │
│                                                          │
│ PGVECTOR SETUP                                           │
│ CREATE EXTENSION vector;                                 │
│ CREATE TABLE docs (                                      │
│     id SERIAL PRIMARY KEY,                               │
│     content TEXT,                                        │
│     embedding vector(1536)                               │
│ );                                                       │
│                                                          │
│ INSERT WITH EMBEDDING                                    │
│ INSERT INTO docs (content, embedding)                    │
│ VALUES ('text', '[0.1, 0.2, ...]'::vector);             │
│                                                          │
│ SIMILARITY SEARCH                                        │
│ SELECT content,                                          │
│        1 - (embedding <=> '[...]'::vector) AS similarity │
│ FROM docs                                                │
│ ORDER BY embedding <=> '[...]'::vector                   │
│ LIMIT 5;                                                 │
│                                                          │
│ OPERATORS                                                │
│ <=>  Cosine distance (most common)                       │
│ <->  L2 distance                                         │
│ <#>  Inner product (negative)                            │
│                                                          │
│ RAG PIPELINE                                              │
│ 1. Embed user query                                      │
│ 2. Vector search for relevant docs                       │
│ 3. Combine query + docs as LLM context                   │
│ 4. Generate grounded answer                              │
│                                                          │
│ DATA ENGINEERING USES                                    │
│ • Data catalog search                                    │
│ • Document Q&A over internal docs                        │
│ • Anomaly explanation (find similar past incidents)      │
│ • SQL assistant (find similar queries)                   │
│ • Data lineage search                                    │
└─────────────────────────────────────────────────────────┘
```

---

### 📝 Today's Checklist

- [ ] Understand what embeddings are and why they capture meaning
- [ ] Generated embeddings using OpenAI API
- [ ] Calculated cosine similarity between texts
- [ ] Installed pgvector and created tables with vector columns
- [ ] Stored menu items with embeddings in PostgreSQL
- [ ] Performed vector similarity search
- [ ] Built a complete RAG pipeline (retrieve → augment → generate)
- [ ] Explored data engineering use cases for RAG
- [ ] Completed at least 4 of 5 exercises
- [ ] Ready for tomorrow: Assessment Day

---

*Day 60 complete! Tomorrow: Week 9 Assessment + Portfolio Enhancement.* 📝🔍
