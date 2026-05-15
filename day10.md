# 📅 Day 10 — Saturday, 24 May 2026
# Python Data Structures: Lists, Dictionaries, Sets, Tuples

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Create and manipulate lists (ordered, changeable)
- Create and manipulate dictionaries (key-value pairs)
- Use sets for unique items and set operations
- Use tuples for fixed collections
- Choose the right data structure for each situation
- Use list comprehensions for clean, Pythonic code

**Why this matters:** Data structures are how you ORGANIZE data in Python. As a data engineer, you'll constantly transform data between formats: API returns JSON (dict of dicts), you convert to rows (list of dicts), then to CSV (list of lists). Understanding data structures is non-negotiable.

---

## ☀️ BLOCK 1: Lists — The Workhorse (Morning, ~1.5 hours)

---

### Task 1: List Basics (30 min)

A list is an ordered, changeable collection. You'll use lists more than any other data structure.

```python
# 🔹 Creating lists
numbers = [1, 2, 3, 4, 5]
names = ["Alice", "Bob", "Charlie"]
mixed = [1, "hello", True, 3.14]  # can mix types (but usually don't)
empty = []

# 🔹 Accessing elements (0-indexed, same as strings)
fruits = ["apple", "banana", "cherry", "date", "elderberry"]
print(fruits[0])     # apple (first)
print(fruits[2])     # cherry (third)
print(fruits[-1])    # elderberry (last)
print(fruits[-2])    # date (second to last)

# 🔹 Slicing (same as strings)
print(fruits[1:3])   # ['banana', 'cherry']
print(fruits[:3])    # ['apple', 'banana', 'cherry']
print(fruits[2:])    # ['cherry', 'date', 'elderberry']
print(fruits[::-1])  # reversed

# 🔹 List length
print(len(fruits))   # 5
```

---

### Task 2: Modifying Lists (30 min)

```python
# 🔹 Adding elements
fruits = ["apple", "banana"]

fruits.append("cherry")       # add to end
print(fruits)  # ['apple', 'banana', 'cherry']

fruits.insert(1, "avocado")   # insert at position 1
print(fruits)  # ['apple', 'avocado', 'banana', 'cherry']

more = ["date", "elderberry"]
fruits.extend(more)            # add another list
print(fruits)  # ['apple', 'avocado', 'banana', 'cherry', 'date', 'elderberry']

# 🔹 Removing elements
fruits.remove("avocado")       # remove by value (first match)
print(fruits)

popped = fruits.pop()          # remove & return last item
print(f"Removed: {popped}")    # elderberry
print(fruits)

popped2 = fruits.pop(0)        # remove & return index 0
print(f"Removed: {popped2}")   # apple

# 🔹 Changing elements
fruits[0] = "blueberry"        # replace by index
print(fruits)

# 🔹 Checking membership
print("banana" in fruits)      # True
print("grape" in fruits)       # False

# 🔹 Counting and finding
numbers = [1, 3, 5, 3, 7, 3, 9]
print(numbers.count(3))        # 3 (appears 3 times)
print(numbers.index(7))        # 4 (first position of 7)

# 🔹 Sorting
numbers = [3, 1, 4, 1, 5, 9, 2, 6]
numbers.sort()                  # sort in place (modifies original)
print(numbers)  # [1, 1, 2, 3, 4, 5, 6, 9]

numbers.sort(reverse=True)     # sort descending
print(numbers)  # [9, 6, 5, 4, 3, 2, 1, 1]

# sorted() returns a NEW list (doesn't modify original)
original = [3, 1, 4, 1, 5]
sorted_copy = sorted(original)
print(f"Original: {original}")     # [3, 1, 4, 1, 5]
print(f"Sorted:   {sorted_copy}")  # [1, 1, 3, 4, 5]

# 🔹 Reversing
numbers = [1, 2, 3, 4, 5]
numbers.reverse()
print(numbers)  # [5, 4, 3, 2, 1]

# 🔹 Copying (IMPORTANT — avoid reference bugs)
a = [1, 2, 3]
b = a          # ❌ b points to SAME list as a
b[0] = 99
print(a)       # [99, 2, 3] — a also changed! This is a common bug.

c = a.copy()   # ✅ c is an independent copy
c[0] = 42
print(a)       # [99, 2, 3] — a unchanged
print(c)       # [42, 2, 3]
```

> 🧠 **The #1 Python gotcha:** `b = a` does NOT create a copy. Both variables point to the SAME list. Always use `.copy()` or `list(a)` when you need a separate copy.

---

### Task 3: List Comprehensions — Pythonic Loops (20 min)

List comprehensions are a concise way to create lists. You'll see them everywhere in data engineering code.

```python
# 🔹 Traditional loop
squares = []
for i in range(10):
    squares.append(i ** 2)
print(squares)  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# 🔹 Same thing with list comprehension — one line!
squares = [i ** 2 for i in range(10)]
print(squares)

# 🔹 With condition (filter)
even_squares = [i ** 2 for i in range(10) if i % 2 == 0]
print(even_squares)  # [0, 4, 16, 36, 64]

# 🔹 Practical examples
# Convert all names to uppercase
names = ["alice", "bob", "charlie"]
upper_names = [name.upper() for name in names]
print(upper_names)  # ['ALICE', 'BOB', 'CHARLIE']

# Extract domains from emails
emails = ["alice@grab.com", "bob@shopee.sg", "charlie@grab.com"]
domains = [email.split("@")[1] for email in emails]
print(domains)  # ['grab.com', 'shopee.sg', 'grab.com']

# Filter salaries above threshold
salaries = [5000, 8500, 6200, 12000, 4800, 9200]
high_earners = [s for s in salaries if s > 8000]
print(high_earners)  # [8500, 12000, 9200]

# 🔹 Dictionary comprehension (same idea, creates dict)
names = ["Alice", "Bob", "Charlie"]
name_lengths = {name: len(name) for name in names}
print(name_lengths)  # {'Alice': 5, 'Bob': 3, 'Charlie': 7}

# Count characters in a string
text = "data engineering"
char_count = {char: text.count(char) for char in set(text) if char != " "}
print(char_count)  # {'d': 1, 'a': 2, 't': 2, ...}
```

**🔥 Practice exercises:**

1. Given `prices = [29.90, 59.90, 15.00, 99.90, 45.00]`, create a new list with 10% tax added to each
2. Given a list of strings, filter only the ones that start with "S" (case-insensitive)
3. Given `numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]`, create a list of "even" or "odd" labels
4. Create a dictionary mapping each number 1-10 to its square using dict comprehension
5. Given a list of email addresses, create a dictionary mapping domain → count of emails from that domain

<details>
<summary>📖 Answers</summary>

```python
# 1
prices = [29.90, 59.90, 15.00, 99.90, 45.00]
with_tax = [round(p * 1.10, 2) for p in prices]
print(with_tax)

# 2
words = ["Singapore", "malaysia", "Stockholm", "tokyo", "Seoul", "bangkok"]
s_words = [w for w in words if w.lower().startswith("s")]
print(s_words)  # ['Singapore', 'Stockholm', 'Seoul']

# 3
numbers = list(range(1, 11))
labels = ["even" if n % 2 == 0 else "odd" for n in numbers]
print(labels)

# 4
squares = {n: n**2 for n in range(1, 11)}
print(squares)

# 5
emails = ["alice@grab.com", "bob@shopee.sg", "carol@grab.com",
          "dave@grab.com", "eve@shopee.sg", "frank@airasia.com"]
domain_counts = {}
for email in emails:
    domain = email.split("@")[1]
    domain_counts[domain] = domain_counts.get(domain, 0) + 1
print(domain_counts)
# {'grab.com': 3, 'shopee.sg': 2, 'airasia.com': 1}
```
</details>

---

## 🔥 BLOCK 2: Dictionaries — Key-Value Power (Afternoon, ~1.5 hours)

---

### Task 4: Dictionary Basics (30 min)

A dictionary stores key-value pairs. Think of it as a real dictionary: you look up a word (key) and get its definition (value).

```python
# 🔹 Creating dictionaries
employee = {
    "name": "Alice Tan",
    "age": 28,
    "department": "Engineering",
    "salary": 8500.00,
    "cities": ["Singapore", "Kuala Lumpur"],  # values can be anything
}

# 🔹 Accessing values
print(employee["name"])           # Alice Tan
print(employee["salary"])         # 8500.0

# ⚠️ KeyError if key doesn't exist:
# print(employee["bonus"])        # ERROR!

# ✅ Safe access with .get()
print(employee.get("bonus"))              # None (no error)
print(employee.get("bonus", 0))           # 0 (default value)
print(employee.get("salary", 0))          # 8500.0 (key exists, returns actual value)

# 🔹 Modifying dictionaries
employee["salary"] = 9000          # update existing
employee["email"] = "alice@co.com" # add new key-value
employee["bonus"] = 1500           # add new
print(employee)

# 🔹 Removing
del employee["bonus"]              # delete a key
bonus = employee.pop("email", None) # pop (remove and return), safe with default

# 🔹 Checking if key exists
print("name" in employee)          # True
print("bonus" in employee)         # False

# 🔹 Dictionary methods
print(employee.keys())             # dict_keys(['name', 'age', ...])
print(employee.values())           # dict_values(['Alice Tan', 28, ...])
print(employee.items())            # dict_items([('name', 'Alice Tan'), ...])

# 🔹 Looping through dictionary
for key in employee:
    print(f"{key}: {employee[key]}")

# More readable:
for key, value in employee.items():
    print(f"{key}: {value}")

# 🔹 Nested dictionaries (very common in JSON/API data)
company = {
    "name": "Grab",
    "hq": {"city": "Singapore", "country": "Singapore"},
    "departments": {
        "engineering": {"head": "Frank", "count": 200},
        "marketing": {"head": "Eve", "count": 50},
    }
}

print(company["hq"]["city"])                        # Singapore
print(company["departments"]["engineering"]["count"])  # 200
```

---

### Task 5: Dictionaries for Data Engineering (30 min)

This is how real data flows in ETL pipelines — as lists of dictionaries.

```python
# 🔹 "Rows" of data — each dictionary is a row
employees = [
    {"id": 1, "name": "Alice", "dept": "Engineering", "salary": 8500},
    {"id": 2, "name": "Bob", "dept": "Marketing", "salary": 6200},
    {"id": 3, "name": "Charlie", "dept": "Engineering", "salary": 9200},
    {"id": 4, "name": "Diana", "dept": "HR", "salary": 5800},
    {"id": 5, "name": "Eve", "dept": "Marketing", "salary": 7100},
]

# 🔹 Common operations you'll do as a data engineer:

# Filter: only Engineering employees
engineers = [e for e in employees if e["dept"] == "Engineering"]
print(f"Engineers: {[e['name'] for e in engineers]}")

# Map: extract just names
names = [e["name"] for e in employees]
print(names)  # ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve']

# Aggregate: average salary by department
dept_salaries = {}
for e in employees:
    dept = e["dept"]
    if dept not in dept_salaries:
        dept_salaries[dept] = []
    dept_salaries[dept].append(e["salary"])

for dept, salaries in dept_salaries.items():
    avg = sum(salaries) / len(salaries)
    print(f"{dept}: avg ${avg:,.0f} ({len(salaries)} employees)")

# Transform: add a new calculated field
for e in employees:
    e["annual_salary"] = e["salary"] * 12
    e["tax"] = e["salary"] * 0.10
    e["net_salary"] = e["salary"] - e["tax"]
print(employees[0])
# Now has annual_salary, tax, net_salary fields

# Sort by salary descending
sorted_emps = sorted(employees, key=lambda e: e["salary"], reverse=True)
for e in sorted_emps:
    print(f"{e['name']}: ${e['salary']:,}")
```

> 🧠 **lambda is a one-line anonymous function.** `lambda e: e["salary"]` means "given e, return e['salary']." It's used as the sort key. You'll see lambda often in data engineering.

**🔥 Practice exercises:**

6. Given a list of sales records, find the total revenue per product category
7. Find the employee with the highest salary
8. Group employees by department, print each group

```python
sales = [
    {"product": "Laptop", "category": "Electronics", "amount": 1500},
    {"product": "T-Shirt", "category": "Clothing", "amount": 30},
    {"product": "Phone", "category": "Electronics", "amount": 900},
    {"product": "Jeans", "category": "Clothing", "amount": 60},
    {"product": "Coffee", "category": "Food", "amount": 15},
    {"product": "Tablet", "category": "Electronics", "amount": 600},
    {"product": "Tea", "category": "Food", "amount": 12},
]
```

<details>
<summary>📖 Answers</summary>

```python
# 6
category_revenue = {}
for s in sales:
    cat = s["category"]
    category_revenue[cat] = category_revenue.get(cat, 0) + s["amount"]
print(category_revenue)
# {'Electronics': 3000, 'Clothing': 90, 'Food': 27}

# 7
highest = max(employees, key=lambda e: e["salary"])
print(f"Highest paid: {highest['name']} at ${highest['salary']:,}")

# 8
from collections import defaultdict
groups = defaultdict(list)
for e in employees:
    groups[e["dept"]].append(e["name"])

for dept, names in groups.items():
    print(f"{dept}: {', '.join(names)}")
```
</details>

---

## 🔥 BLOCK 3: Sets and Tuples (Afternoon, ~1 hour)

---

### Task 6: Sets — Unique Items (20 min)

```python
# 🔹 A set is an UNORDERED collection with NO duplicates
fruits = {"apple", "banana", "cherry", "apple"}
print(fruits)  # {'apple', 'banana', 'cherry'} — duplicate removed!

# 🔹 Creating sets
numbers = set([1, 2, 3, 2, 1, 4])
print(numbers)  # {1, 2, 3, 4}

# From a string
letters = set("hello")
print(letters)  # {'h', 'e', 'l', 'o'} — only unique letters

# 🔹 Set operations (like Venn diagrams!)
sg_customers = {"Alice", "Bob", "Charlie", "Diana"}
my_customers = {"Charlie", "Diana", "Eve", "Frank"}

# Union — all unique customers
all_customers = sg_customers | my_customers
print(f"All: {all_customers}")

# Intersection — customers in BOTH
both = sg_customers & my_customers
print(f"Both: {both}")  # {'Charlie', 'Diana'}

# Difference — in SG but not MY
sg_only = sg_customers - my_customers
print(f"SG only: {sg_only}")  # {'Alice', 'Bob'}

# Symmetric difference — in one but NOT both
either = sg_customers ^ my_customers
print(f"One only: {either}")  # {'Alice', 'Bob', 'Eve', 'Frank'}

# 🔹 Practical uses in data engineering
# Remove duplicates from a list
data = [1, 2, 3, 2, 1, 4, 5, 4, 3]
unique = list(set(data))
print(unique)  # [1, 2, 3, 4, 5]

# Check membership (sets are MUCH faster than lists for this)
valid_cities = {"Singapore", "Kuala Lumpur", "Penang", "Johor Bahru"}
city = "Singapore"
print(city in valid_cities)  # True — O(1) lookup vs O(n) for list
```

---

### Task 7: Tuples — Immutable Collections (15 min)

```python
# 🔹 Tuples are like lists but CANNOT be changed (immutable)
point = (3, 4)
rgb = (255, 128, 0)
employee_record = ("Alice", 28, "Engineering", 8500.0)

# 🔹 Accessing (same as lists)
print(point[0])   # 3
print(employee_record[2])  # Engineering

# 🔹 Cannot modify
# point[0] = 5  ← ERROR! Tuples are immutable

# 🔹 Why use tuples?
# 1. Data that shouldn't change (coordinates, config)
# 2. Dictionary keys (lists can't be keys, tuples can)
# 3. Return multiple values from functions
# 4. Slightly faster than lists

# 🔹 Tuple unpacking
name, age, dept, salary = employee_record
print(f"{name} works in {dept}")

# 🔹 Useful tuple functions
coordinates = [(1, 2), (3, 4), (5, 6), (1, 2)]
print(coordinates.count((1, 2)))  # 2

# 🔹 Named tuples (cleaner than index access)
from collections import namedtuple
Employee = namedtuple("Employee", ["name", "age", "dept", "salary"])
emp = Employee("Alice", 28, "Engineering", 8500)
print(emp.name)    # Alice (much better than emp[0])
print(emp.salary)  # 8500
```

---

### Task 8: Choosing the Right Data Structure (10 min)

| Structure | Ordered? | Changeable? | Duplicates? | Use when... |
|-----------|----------|-------------|-------------|-------------|
| **List** | ✅ | ✅ | ✅ | You need a sequence of items |
| **Dict** | ✅ (Py 3.7+) | ✅ | Keys: ❌ | You need key-value lookup |
| **Set** | ❌ | ✅ | ❌ | You need unique items or fast membership check |
| **Tuple** | ✅ | ❌ | ✅ | Data shouldn't change |

```python
# Real examples of when to use each:

# List: ordered rows of data
csv_rows = [
    ["name", "age", "city"],
    ["Alice", 28, "Singapore"],
    ["Bob", 25, "KL"],
]

# Dict: structured record
employee = {"name": "Alice", "age": 28, "dept": "Engineering"}

# Set: unique values, fast lookup
valid_statuses = {"active", "pending", "completed"}
if status in valid_statuses:  # fast!
    process(status)

# Tuple: fixed data
database_config = ("localhost", 5432, "mydb", "admin")
```

---

## 🌙 BLOCK 4: Practice + Consolidation (Evening, ~1 hour)

---

### Task 9: Comprehensive Practice (30 min)

Write `day10_data_structures.py`:

1. **Word frequency counter:** Given a sentence, count each word and return a sorted dictionary (most frequent first)
2. **Merge two employee lists:** Given two lists of employee dicts, merge them (avoid duplicates by id)
3. **Data transformation:** Given a list of raw API responses, extract and clean into a structured format

```python
# For exercise 3:
raw_data = [
    {"user_id": 1, "full_name": "  Alice Tan  ", "email": "ALICE@COMPANY.COM", "salary": "8500"},
    {"user_id": 2, "full_name": "Bob Lim", "email": "bob@company.com", "salary": "6200"},
    {"user_id": 3, "full_name": "CHARLIE WONG", "email": "  charlie@company.com  ", "salary": "9200"},
]

# Transform into:
# [
#     {"id": 1, "name": "Alice Tan", "email": "alice@company.com", "salary": 8500.0},
#     ...
# ]
```

<details>
<summary>📖 Answers</summary>

```python
# 1
def word_frequency(sentence):
    words = sentence.lower().split()
    freq = {}
    for word in words:
        # Remove punctuation
        word = word.strip(".,!?")
        freq[word] = freq.get(word, 0) + 1
    # Sort by frequency descending
    return dict(sorted(freq.items(), key=lambda x: x[1], reverse=True))

print(word_frequency("the cat sat on the mat the cat ate the rat"))
# {'the': 4, 'cat': 2, 'sat': 1, ...}

# 2
list_a = [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"},
    {"id": 3, "name": "Charlie"},
]
list_b = [
    {"id": 2, "name": "Bob"},     # duplicate
    {"id": 4, "name": "Diana"},
    {"id": 5, "name": "Eve"},
]

seen_ids = set()
merged = []
for emp in list_a + list_b:
    if emp["id"] not in seen_ids:
        merged.append(emp)
        seen_ids.add(emp["id"])
print(merged)

# 3
raw_data = [
    {"user_id": 1, "full_name": "  Alice Tan  ", "email": "ALICE@COMPANY.COM", "salary": "8500"},
    {"user_id": 2, "full_name": "Bob Lim", "email": "bob@company.com", "salary": "6200"},
    {"user_id": 3, "full_name": "CHARLIE WONG", "email": "  charlie@company.com  ", "salary": "9200"},
]

cleaned = []
for record in raw_data:
    cleaned.append({
        "id": record["user_id"],
        "name": record["full_name"].strip().title(),
        "email": record["email"].strip().lower(),
        "salary": float(record["salary"]),
    })

for c in cleaned:
    print(c)
```
</details>

---

### Task 10: Online Practice (20 min)

On https://www.learnpython.org/:
- [ ] **Lists** — exercises
- [ ] **Dictionaries** — exercises
- [ ] **Sets** — if available

---

### Task 11: Daily Reflection + GitHub Push (10 min)

```markdown
# Day 10 — Data Structures
Date: 2026-05-24

## Structures I Can Now Use
- List → ordered, changeable, allows duplicates
- Dict → key-value, changeable, keys unique
- Set → unordered, unique items, fast membership
- Tuple → ordered, immutable, fixed data

## Key Pattern: List of Dicts = Table of Rows
Each dict = one row. Keys = column names. This is how data flows in ETL.

## Tomorrow: File I/O (reading/writing CSV, JSON, text)
```

```bash
git add .
git commit -m "Day 10: Data structures — lists, dicts, sets, tuples, comprehensions"
git push
```

---

## ✅ Day 10 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Created, accessed, modified lists | ☐ |
| 2 | Used list methods (append, pop, sort, extend) | ☐ |
| 3 | Wrote list comprehensions | ☐ |
| 4 | Created, accessed, modified dictionaries | ☐ |
| 5 | Processed list-of-dicts (filter, aggregate, transform) | ☐ |
| 6 | Used sets for uniqueness and membership | ☐ |
| 7 | Used tuples for immutable data | ☐ |
| 8 | Solved 8+ practice exercises | ☐ |
| 9 | Completed online exercises | ☐ |
| 10 | Day 10 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```python
# List
fruits = ["apple", "banana"]
fruits.append("cherry")
fruits[0]                    # apple
[x*2 for x in range(5)]     # comprehension

# Dict
person = {"name": "Alice", "age": 28}
person["name"]               # Alice
person.get("bonus", 0)       # safe access
for k, v in person.items()   # loop

# Set
unique = set([1, 2, 2, 3])   # {1, 2, 3}
a | b                        # union
a & b                        # intersection
a - b                        # difference

# Tuple
point = (3, 4)
x, y = point                 # unpacking
```
