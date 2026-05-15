# 📅 Day 59 — Saturday, 12 July 2026
# 🤖 LLM API Basics: OpenAI/Anthropic API, Prompt Engineering for Data Tasks

---

## 🎯 Today's Goal

Large Language Models (LLMs) are transforming data engineering. Companies in Singapore are using AI to classify data, generate SQL, detect anomalies, and automate data quality checks. Today you learn to call LLM APIs and use them for real data engineering tasks.

**Philosophy:** You're not training AI models — you're a data engineer who uses AI as a tool. Think of it like using Pandas: you don't need to know how Pandas is built, you need to know how to use it to get your job done.

**Important:** LLM APIs cost money. We'll use minimal tokens and keep costs under $1 for the entire day.

---

## ☀️ Morning Block (2 hours): LLM API Fundamentals

### What are LLMs? (Quick Version)

A Large Language Model (LLM) is an AI system trained on massive amounts of text. You send it text (a "prompt") and it generates a response. Think of it like a very smart autocomplete.

**Popular LLM APIs (2026):**
| Provider | Model | Price (input/1M tokens) | Best For |
|----------|-------|------------------------|----------|
| OpenAI | GPT-4o-mini | ~$0.15 | Cheap, fast, good enough for most tasks |
| OpenAI | GPT-4o | ~$2.50 | Complex reasoning |
| Anthropic | Claude Haiku | ~$0.25 | Fast, good at structured output |
| Anthropic | Claude Sonnet | ~$3.00 | Balanced performance |

**What's a token?** Roughly 4 characters or 0.75 words. "Singapore is hot" = ~4 tokens. Costs are per million tokens.

---

### Setting Up

```bash
pip install openai anthropic
```

**Get an API key:**
- OpenAI: https://platform.openai.com/api-keys (requires $5 minimum credit)
- Anthropic: https://console.anthropic.com/ (requires $5 minimum credit)

> **💡 Budget tip:** If you don't want to spend money, you can practice the concepts using the free tier of Google Gemini API or use a local model like Ollama. The patterns are identical.

---

### Your First LLM API Call

```python
from openai import OpenAI

client = OpenAI(api_key="YOUR_API_KEY")

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a helpful data engineering assistant."},
        {"role": "user", "content": "What are the 3 most important skills for a data engineer?"}
    ],
    max_tokens=200,
    temperature=0.3  # Lower = more deterministic, Higher = more creative
)

print(response.choices[0].message.content)
print(f"\nTokens used: {response.usage.total_tokens}")
```

**Understanding the message roles:**
- `system` — Sets the AI's behavior/persona (like giving instructions to an employee)
- `user` — Your actual question/request
- `assistant` — The AI's previous responses (for multi-turn conversations)

---

### Token Counting and Cost Estimation

```python
import tiktoken

def count_tokens(text, model="gpt-4o-mini"):
    """Estimate token count for a text."""
    encoding = tiktoken.encoding_for_model(model)
    return len(encoding.encode(text))

def estimate_cost(input_tokens, output_tokens, model="gpt-4o-mini"):
    """Estimate API cost in USD."""
    prices = {
        "gpt-4o-mini": {"input": 0.15 / 1_000_000, "output": 0.60 / 1_000_000},
        "gpt-4o": {"input": 2.50 / 1_000_000, "output": 10.00 / 1_000_000},
    }
    
    p = prices.get(model, prices["gpt-4o-mini"])
    cost = (input_tokens * p["input"]) + (output_tokens * p["output"])
    return cost

# Example
prompt = "Classify this food review: 'The chicken rice was amazing, best in Singapore!'"
tokens = count_tokens(prompt)
print(f"Prompt tokens: {tokens}")
print(f"Estimated cost per call: ${estimate_cost(tokens, 50):.6f}")
print(f"Cost per 1000 calls: ${estimate_cost(tokens, 50) * 1000:.4f}")
```

---

### Batch Processing Pattern

When classifying hundreds of records, batch them to save API calls:

```python
def batch_classify(items, batch_size=10):
    """Process items in batches to reduce API calls."""
    results = []
    
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        
        # Create numbered list for the prompt
        items_text = "\n".join(f"{j+1}. {item}" for j, item in enumerate(batch))
        
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You classify food reviews. Reply with just the number and classification (positive/negative/neutral), one per line."},
                {"role": "user", "content": f"Classify each review:\n{items_text}"}
            ],
            max_tokens=200,
            temperature=0.1
        )
        
        # Parse response
        lines = response.choices[0].message.content.strip().split("\n")
        for j, line in enumerate(lines):
            results.append({
                "review": batch[j] if j < len(batch) else "",
                "classification": line.strip()
            })
        
        print(f"Batch {i//batch_size + 1}: {len(batch)} items processed")
    
    return results
```

---

### Morning Exercises

**Exercise 1:** Write a function `ask_data_engineer(question)` that calls the OpenAI API with a system prompt of "You are a senior data engineer answering interview questions. Be concise and practical." Test with 3 data engineering interview questions.

<details>
<summary>🔑 Answer</summary>

```python
from openai import OpenAI

client = OpenAI(api_key="YOUR_API_KEY")

def ask_data_engineer(question):
    """Ask a data engineering question to the LLM."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "You are a senior data engineer with 10 years of experience at Grab Singapore. Answer interview questions concisely and practically. Use examples where possible."
            },
            {
                "role": "user",
                "content": question
            }
        ],
        max_tokens=300,
        temperature=0.3
    )
    
    answer = response.choices[0].message.content
    tokens = response.usage.total_tokens
    cost = estimate_cost(response.usage.prompt_tokens, response.usage.completion_tokens)
    
    print(f"Q: {question}")
    print(f"A: {answer}")
    print(f"\n[Tokens: {tokens}, Cost: ${cost:.6f}]")
    return answer

# Test with interview questions
questions = [
    "What is the difference between ETL and ELT?",
    "How would you handle slowly changing dimensions in a data warehouse?",
    "What is idempotency and why does it matter in data pipelines?"
]

for q in questions:
    print("=" * 60)
    ask_data_engineer(q)
    print()
```

</details>

**Exercise 2:** Write a function that takes a table schema and a natural language question, and generates a SQL query using the LLM. Use the MakanExpress orders table schema.

<details>
<summary>🔑 Answer</summary>

```python
def text_to_sql(question, schema_description):
    """Convert natural language question to SQL using LLM."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": f"""You are a SQL expert. Generate PostgreSQL queries based on the user's question.

Schema:
{schema_description}

Rules:
- Return ONLY the SQL query, no explanation
- Use proper PostgreSQL syntax
- Include appropriate WHERE, GROUP BY, ORDER BY
- Use SGT timezone (UTC+8) for any date operations"""
            },
            {
                "role": "user",
                "content": question
            }
        ],
        max_tokens=200,
        temperature=0.1
    )
    
    sql = response.choices[0].message.content.strip()
    # Remove markdown code blocks if present
    if sql.startswith("```"):
        sql = sql.split("\n", 1)[1].rsplit("```", 1)[0].strip()
    
    return sql

# MakanExpress schema
schema = """
Tables:
- orders (id INT, customer_id INT, restaurant_id INT, order_date TIMESTAMP, 
  total_amount DECIMAL, status VARCHAR, delivery_city VARCHAR)
- restaurants (id INT, name VARCHAR, cuisine VARCHAR, city VARCHAR, rating DECIMAL)
- customers (id INT, name VARCHAR, city VARCHAR, signup_date DATE)
"""

# Test questions
questions = [
    "What are the top 5 restaurants by total revenue in Singapore?",
    "How many orders were placed each day this week?",
    "Which cuisine type has the highest average order value in KL?",
    "Find customers who ordered more than 10 times but never rated above 3 stars"
]

for q in questions:
    sql = text_to_sql(q, schema)
    print(f"Q: {q}")
    print(f"SQL: {sql}\n")
```

</details>

**Exercise 3:** Build a cost tracker class that accumulates token usage across multiple API calls and reports total cost at the end.

<details>
<summary>🔑 Answer</summary>

```python
class LLMCostTracker:
    """Track LLM API usage and costs."""
    
    def __init__(self, model="gpt-4o-mini"):
        self.model = model
        self.calls = []
        
        self.prices = {
            "gpt-4o-mini": {"input": 0.15 / 1_000_000, "output": 0.60 / 1_000_000},
            "gpt-4o": {"input": 2.50 / 1_000_000, "output": 10.00 / 1_000_000},
        }
    
    def record(self, prompt_tokens, completion_tokens, label=""):
        """Record an API call."""
        self.calls.append({
            "prompt_tokens": prompt_tokens,
            "completion_tokens": completion_tokens,
            "total_tokens": prompt_tokens + completion_tokens,
            "label": label,
            "cost": self._calculate_cost(prompt_tokens, completion_tokens)
        })
    
    def _calculate_cost(self, input_tokens, output_tokens):
        p = self.prices.get(self.model, self.prices["gpt-4o-mini"])
        return (input_tokens * p["input"]) + (output_tokens * p["output"])
    
    @property
    def summary(self):
        total_input = sum(c["prompt_tokens"] for c in self.calls)
        total_output = sum(c["completion_tokens"] for c in self.calls)
        total_cost = sum(c["cost"] for c in self.calls)
        
        return {
            "model": self.model,
            "total_calls": len(self.calls),
            "total_input_tokens": total_input,
            "total_output_tokens": total_output,
            "total_cost_usd": total_cost
        }
    
    def print_report(self):
        s = self.summary
        print(f"{'='*45}")
        print(f"LLM Cost Report — {s['model']}")
        print(f"{'='*45}")
        print(f"Total API calls:    {s['total_calls']}")
        print(f"Input tokens:       {s['total_input_tokens']:,}")
        print(f"Output tokens:      {s['total_output_tokens']:,}")
        print(f"Total cost:         ${s['total_cost_usd']:.6f}")
        print(f"{'='*45}")
        
        print(f"\n{'Label':<30} {'Tokens':<10} {'Cost':<12}")
        print("-" * 52)
        for call in self.calls:
            label = call["label"][:28]
            print(f"{label:<30} {call['total_tokens']:<10} ${call['cost']:<11.6f}")

# Usage
tracker = LLMCostTracker("gpt-4o-mini")

# Simulate some API calls
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is ETL?"}],
    max_tokens=100
)
tracker.record(response.usage.prompt_tokens, response.usage.completion_tokens, "ETL explanation")

tracker.print_report()
```

</details>

---

## 🌤️ Afternoon Block (2 hours): Prompt Engineering for Data Tasks

### Prompt Engineering Patterns for Data Engineers

**Pattern 1: Classification**

```python
def classify_reviews(reviews):
    """Classify food delivery reviews as positive/negative/neutral."""
    results = []
    
    for review in reviews:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {
                    "role": "system",
                    "content": """Classify food delivery reviews. Return JSON only:
{"sentiment": "positive|negative|neutral", "category": "food_quality|delivery_speed|service|price|other", "confidence": 0.0-1.0}"""
                },
                {"role": "user", "content": f"Review: {review}"}
            ],
            max_tokens=100,
            temperature=0.1
        )
        
        import json
        try:
            result = json.loads(response.choices[0].message.content)
            result["review"] = review
            results.append(result)
        except json.JSONDecodeError:
            results.append({"review": review, "sentiment": "error", "error": "Failed to parse"})
    
    return results

# Test
reviews = [
    "Chicken rice came cold and delivery took 1 hour. Terrible!",
    "Best nasi lemak in KL! Will order again. 5 stars!",
    "Food was okay, nothing special. Delivery was on time.",
    "RM25 for a small portion of char kway teow? Overpriced!",
    "The bak kut teh arrived piping hot and the soup was flavorful. Great service!"
]

results = classify_reviews(reviews)
for r in results:
    print(f"{'✅' if r.get('sentiment') == 'positive' else '❌' if r.get('sentiment') == 'negative' else '➖'} "
          f"[{r.get('sentiment', '?')}] {r.get('category', '?')} — {r['review'][:50]}...")
```

---

**Pattern 2: Entity Extraction**

```python
def extract_entities(text):
    """Extract structured data from unstructured text."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """Extract entities from the text. Return JSON:
{
    "restaurant_name": "...",
    "dishes": ["..."],
    "price_mentions": [{"amount": 0, "currency": "SGD|MYR"}],
    "location": "...",
    "rating": null,
    "cuisine_type": "..."
}
Use null for missing fields."""
            },
            {"role": "user", "content": text}
        ],
        max_tokens=200,
        temperature=0.1
    )
    
    import json
    try:
        return json.loads(response.choices[0].message.content)
    except:
        return {"error": "Parse failed"}

# Test with food reviews
texts = [
    "Had amazing Hainanese Chicken Rice at Tian Tian in Maxwell Food Centre for just SGD 5. Worth every cent!",
    "Ordered Nasi Lemak from Village Park in Damansara Uptown, PJ. RM 15 for the special with ayam goreng. 4.5/5!",
    "The char kway teow at Penang Road was only MYR 8 but the portion was tiny. Wouldn't go back."
]

for text in texts:
    entities = extract_entities(text)
    print(f"Text: {text[:60]}...")
    print(f"Entities: {json.dumps(entities, indent=2)}\n")
```

---

**Pattern 3: Data Quality Checks**

```python
def suggest_data_quality_rules(table_name, columns):
    """Use LLM to suggest data quality rules for a table."""
    schema_desc = f"Table: {table_name}\nColumns:\n"
    for col in columns:
        schema_desc += f"  - {col['name']} ({col['type']}): {col.get('description', '')}\n"
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """You are a data quality expert. Suggest data quality rules as a JSON array.
Each rule: {"rule_name": "...", "check_type": "not_null|unique|range|regex|custom", 
"column": "...", "description": "...", "sql_check": "..."}"""
            },
            {"role": "user", "content": f"Suggest 5 data quality rules:\n\n{schema_desc}"}
        ],
        max_tokens=500,
        temperature=0.3
    )
    
    import json
    try:
        return json.loads(response.choices[0].message.content)
    except:
        return []

# Test with MakanExpress orders table
columns = [
    {"name": "id", "type": "INT", "description": "Primary key"},
    {"name": "customer_id", "type": "INT", "description": "FK to customers"},
    {"name": "restaurant_id", "type": "INT", "description": "FK to restaurants"},
    {"name": "order_date", "type": "TIMESTAMP", "description": "When order was placed"},
    {"name": "total_amount", "type": "DECIMAL", "description": "Order total in SGD/MYR"},
    {"name": "status", "type": "VARCHAR", "description": "pending/preparing/delivered/cancelled"},
    {"name": "delivery_city", "type": "VARCHAR", "description": "Delivery destination city"},
]

rules = suggest_data_quality_rules("orders", columns)
for rule in rules:
    print(f"✅ {rule.get('rule_name', '?')}: {rule.get('description', '?')}")
    print(f"   Check: {rule.get('sql_check', '?')}\n")
```

---

### Afternoon Exercises

**Exercise 4:** Build a `ReviewClassifier` class that classifies MakanExpress food reviews. Include batch processing (classify 5 reviews in one API call to save tokens) and a results DataFrame export.

<details>
<summary>🔑 Answer</summary>

```python
import json
import pandas as pd
from openai import OpenAI

class ReviewClassifier:
    """Classify food delivery reviews using LLM."""
    
    def __init__(self, api_key):
        self.client = OpenAI(api_key=api_key)
        self.results = []
    
    def classify_single(self, review):
        """Classify a single review."""
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {
                    "role": "system",
                    "content": """Classify this food delivery review. Return JSON only:
{"sentiment": "positive|negative|neutral", 
 "category": "food_quality|delivery_speed|service|price|packaging|other",
 "confidence": 0.0-1.0,
 "summary": "one sentence summary"}"""
                },
                {"role": "user", "content": review}
            ],
            max_tokens=100,
            temperature=0.1
        )
        
        try:
            result = json.loads(response.choices[0].message.content)
            result["review"] = review
            result["tokens_used"] = response.usage.total_tokens
            return result
        except json.JSONDecodeError:
            return {"review": review, "sentiment": "error", "tokens_used": 0}
    
    def classify_batch(self, reviews, batch_size=5):
        """Classify reviews in batches to save API calls."""
        self.results = []
        
        for i in range(0, len(reviews), batch_size):
            batch = reviews[i:i + batch_size]
            numbered = "\n".join(f"{j+1}. {r}" for j, r in enumerate(batch))
            
            response = self.client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[
                    {
                        "role": "system",
                        "content": """Classify each food review. Return a JSON array:
[{"sentiment": "positive|negative|neutral", "category": "food_quality|delivery|service|price|other", "confidence": 0.0-1.0}]
One object per review, same order as input."""
                    },
                    {"role": "user", "content": numbered}
                ],
                max_tokens=300,
                temperature=0.1
            )
            
            try:
                classifications = json.loads(response.choices[0].message.content)
                for j, review in enumerate(batch):
                    if j < len(classifications):
                        result = classifications[j]
                        result["review"] = review
                        self.results.append(result)
                    else:
                        self.results.append({"review": review, "sentiment": "unknown"})
            except json.JSONDecodeError:
                for review in batch:
                    self.results.append({"review": review, "sentiment": "parse_error"})
            
            print(f"Batch {i//batch_size + 1}: classified {len(batch)} reviews")
        
        return self.results
    
    def to_dataframe(self):
        """Export results to DataFrame."""
        return pd.DataFrame(self.results)
    
    def summary(self):
        """Print classification summary."""
        df = self.to_dataframe()
        if df.empty:
            print("No results yet")
            return
        
        print("📊 Classification Summary")
        print("=" * 35)
        print(f"Total reviews: {len(df)}")
        print(f"\nSentiment distribution:")
        print(df["sentiment"].value_counts().to_string())
        print(f"\nCategory distribution:")
        print(df["category"].value_counts().to_string())

# Test
classifier = ReviewClassifier(api_key="YOUR_API_KEY")
reviews = [
    "The laksa was spicy and delicious! Fast delivery too.",
    "Waited 2 hours for my roti canai. Never again.",
    "Mee goreng was average. Portion size was okay.",
    "SGD 3 for a kopi? That's daylight robbery!",
    "Kaya toast set was perfect. Will order every morning!"
]

results = classifier.classify_batch(reviews)
classifier.summary()
```

</details>

**Exercise 5:** Write a function `generate_dbt_test(model_name, column_descriptions)` that uses the LLM to suggest dbt tests for a given model. Return the SQL for each test.

<details>
<summary>🔑 Answer</summary>

```python
def generate_dbt_tests(model_name, columns):
    """Generate dbt test suggestions using LLM."""
    col_desc = "\n".join(f"- {c['name']} ({c['type']}): {c.get('desc', '')}" for c in columns)
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """Generate dbt YAML test definitions for the given model. Return YAML inside ```yaml code blocks.
Include: unique, not_null, accepted_values, relationships tests where appropriate.
Also include 1-2 custom SQL tests (schema tests) with the actual SQL query."""
            },
            {
                "role": "user",
                "content": f"Model: {model_name}\nColumns:\n{col_desc}"
            }
        ],
        max_tokens=600,
        temperature=0.3
    )
    
    return response.choices[0].message.content

# Test
columns = [
    {"name": "order_id", "type": "INT", "desc": "Primary key"},
    {"name": "customer_id", "type": "INT", "desc": "FK to customers"},
    {"name": "restaurant_name", "type": "VARCHAR", "desc": "Restaurant name"},
    {"name": "total_amount", "type": "DECIMAL", "desc": "Order total"},
    {"name": "order_status", "type": "VARCHAR", "desc": "pending/preparing/delivered/cancelled"},
    {"name": "order_date", "type": "DATE", "desc": "Date of order"},
]

yaml_tests = generate_dbt_tests("stg_orders", columns)
print(yaml_tests)
```

</details>

**Exercise 6:** Write an error-handling wrapper for LLM API calls that handles: rate limits, invalid JSON responses, timeouts, and empty responses. Include retry logic.

<details>
<summary>🔑 Answer</summary>

```python
import time
import json
import logging

logger = logging.getLogger(__name__)

def safe_llm_call(messages, model="gpt-4o-mini", max_tokens=200, 
                  max_retries=3, expect_json=True):
    """Call LLM API with full error handling."""
    
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                max_tokens=max_tokens,
                temperature=0.1
            )
            
            content = response.choices[0].message.content.strip()
            
            # Handle empty response
            if not content:
                logger.warning(f"Empty response (attempt {attempt+1})")
                continue
            
            # Try to parse JSON if expected
            if expect_json:
                # Remove markdown code blocks if present
                if content.startswith("```"):
                    content = content.split("\n", 1)[1].rsplit("```", 1)[0].strip()
                
                try:
                    return json.loads(content)
                except json.JSONDecodeError:
                    logger.warning(f"Invalid JSON response (attempt {attempt+1}): {content[:100]}")
                    if attempt == max_retries - 1:
                        return {"raw_response": content, "parse_error": True}
                    continue
            
            return content
            
        except Exception as e:
            error_str = str(e).lower()
            
            if "rate" in error_str or "429" in error_str:
                wait = 2 ** attempt
                logger.warning(f"Rate limited. Waiting {wait}s...")
                time.sleep(wait)
            elif "timeout" in error_str:
                logger.warning(f"Timeout (attempt {attempt+1})")
                time.sleep(2 ** attempt)
            else:
                logger.error(f"API error: {e}")
                if attempt == max_retries - 1:
                    return {"error": str(e)}
                time.sleep(1)
    
    return {"error": f"Failed after {max_retries} retries"}
```

</details>

---

## 🌙 Evening (1 hour): Quick Reference + Practice

### 🔖 LLM API Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│            LLM API QUICK REFERENCE                      │
├─────────────────────────────────────────────────────────┤
│ OPENAI SDK                                               │
│ from openai import OpenAI                                │
│ client = OpenAI(api_key="...")                           │
│ response = client.chat.completions.create(               │
│     model="gpt-4o-mini",                                │
│     messages=[                                           │
│         {"role": "system", "content": "..."},            │
│         {"role": "user", "content": "..."}               │
│     ],                                                   │
│     max_tokens=200,                                      │
│     temperature=0.3                                      │
│ )                                                        │
│ text = response.choices[0].message.content               │
│ tokens = response.usage.total_tokens                     │
│                                                          │
│ MESSAGE ROLES                                            │
│ system    → AI behavior/persona                          │
│ user      → Your question/request                        │
│ assistant → AI's previous response (multi-turn)          │
│                                                          │
│ PARAMETERS                                               │
│ temperature: 0=deterministic, 1=creative, 0.1-0.3=best  │
│ max_tokens: max response length (shorter=cheaper)        │
│                                                          │
│ COST TIPS                                                │
│ • Use gpt-4o-mini ($0.15/1M input tokens)               │
│ • Set low max_tokens                                     │
│ • Batch multiple items in one call                       │
│ • Cache results (don't re-ask same questions)            │
│                                                          │
│ DATA ENGINEERING USES                                    │
│ • Text classification (sentiment, categories)            │
│ • Entity extraction (from unstructured text)             │
│ • SQL generation (natural language → SQL)                │
│ • Data quality rule suggestions                          │
│ • Data documentation generation                          │
│ • Anomaly explanation                                    │
│                                                          │
│ BEST PRACTICES                                           │
│ ✅ Always handle errors (rate limits, JSON parse)        │
│ ✅ Set max_tokens to control cost                        │
│ ✅ Use temperature=0.1 for structured output             │
│ ✅ Ask for JSON output explicitly                        │
│ ✅ Validate LLM output before using                      │
│ ❌ Don't trust LLM output blindly                        │
│ ❌ Don't send sensitive data to APIs                     │
└─────────────────────────────────────────────────────────┘
```

---

### 📝 Today's Checklist

- [ ] Understand what LLMs are and how data engineers use them
- [ ] Made first LLM API call (OpenAI or alternative)
- [ ] Understand token counting and cost estimation
- [ ] Can implement classification using LLM (review sentiment)
- [ ] Can implement entity extraction (restaurant name, dishes, prices)
- [ ] Can use LLM for SQL generation (text-to-SQL)
- [ ] Can use LLM for data quality rule suggestions
- [ ] Implemented batch processing to save tokens
- [ ] Built error handling for LLM API calls
- [ ] Completed at least 5 of 6 exercises
- [ ] Ready for tomorrow: RAG concepts (embeddings + vector search)

---

*Day 59 complete! Tomorrow: RAG — search your data with AI-powered semantic search.* 🔍🤖
