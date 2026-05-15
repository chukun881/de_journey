# 📅 Day 8 — Thursday, 22 May 2026
# Python Basics: Variables, Data Types, Operators, Input/Output

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Write and run Python scripts
- Use variables and understand Python data types
- Perform arithmetic, comparison, and logical operations
- Take user input and print formatted output
- Use f-strings for clean string formatting

**Why this matters:** Python is the #1 language for data engineering. You'll use it to write ETL scripts, automate data processing, connect to APIs, and orchestrate pipelines. Today is your foundation.

---

## ☀️ BLOCK 1: Setup + First Script (Morning, ~1 hour)

---

### Task 1: Python Environment Setup (20 min)

You have two options. Pick ONE:

**Option A: Your own laptop (recommended)**
1. Download Python from https://www.python.org/downloads/
2. During install, ✅ tick "Add Python to PATH"
3. Open terminal/cmd, verify:
```bash
python --version
# Should show Python 3.11+ or 3.12+

pip --version
# Should show pip version
```
4. Install a code editor: **VS Code** from https://code.visualstudio.com/
5. Install VS Code extension: **Python** (by Microsoft)

**Option B: Use browser (when on someone else's laptop)**
1. Go to https://replit.com or https://www.onlinegdb.com/online_python_compiler
2. No installation needed
3. Good for practice, but you'll want local Python for projects

> 💡 **From now on, you have TWO workspaces:**
> - SQL practice → Neon console (browser)
> - Python practice → Local Python + VS Code

**Create your Python project folder:**
```bash
mkdir -p python-practice
cd python-practice
```

---

### Task 2: Your First Python Script (15 min)

Create a file called `day08_basics.py`:

```python
# ============================================
# Day 8: Python Basics
# ============================================

# This is a comment. Python ignores everything after #
# Use comments to explain your code. ALWAYS.

# 🔹 Your first program
print("Hello, Data Engineer!")
print("Today I start learning Python.")
print(2026)  # numbers don't need quotes

# 🔹 print() is a FUNCTION — it displays output
# You'll use print() constantly for debugging

# 🔹 Multiple values in one print
name = "Alice"
role = "Data Engineer"
print("Name:", name, "| Role:", role)

# 🔹 Run this file:
# python day08_basics.py
```

Run it:
```bash
python day08_basics.py
```

You should see:
```
Hello, Data Engineer!
Today I start learning Python.
2026
Name: Alice | Role: Data Engineer
```

✅ If you see this, Python is working.

---

### Task 3: Variables — Storing Data (25 min)

A variable is a name that stores a value. Think of it as a labeled box.

```python
# 🔹 Creating variables
company = "Grab"              # string (text)
founded = 2012                # integer (whole number)
revenue = 2.3                 # float (decimal)
is_profitable = True          # boolean (True/False)
cities = ["SG", "MY", "TH"]   # list (collection, more on Day 10)

# 🔹 Variable naming rules:
# ✅ Must start with letter or underscore
# ✅ Can contain letters, numbers, underscores
# ✅ Case-sensitive (company ≠ Company)
# ❌ Cannot start with number
# ❌ Cannot contain spaces

# Good names:
customer_name = "Alice"
order_count = 5
is_active = True

# Bad names (these will error):
# 1st_place = "Alice"    ← starts with number
# my name = "Alice"      ← has space
# for = "Alice"          ← reserved keyword

# 🔹 Python is dynamically typed — no need to declare type
x = 10          # x is an integer
x = "hello"     # now x is a string — Python allows this!
# (But in real code, don't change types randomly — it's confusing)

# 🔹 Multiple assignment
x, y, z = 1, 2, 3
print(x, y, z)  # 1 2 3

# 🔹 Swap values (Python makes this easy!)
a = "Singapore"
b = "Malaysia"
a, b = b, a
print(a)  # Malaysia
print(b)  # Singapore
```

---

## 🔥 BLOCK 2: Data Types Deep Dive (Morning, ~1.5 hours)

---

### Task 4: String — Text Data (30 min)

```python
# 🔹 Creating strings
name = 'Alice'          # single quotes
city = "Singapore"      # double quotes (both work, be consistent)
bio = """She is a data
engineer based in SG."""  # triple quotes for multi-line

# 🔹 String operations
first = "Data"
last = "Engineer"
full = first + " " + last    # concatenation with +
print(full)  # Data Engineer

repeat = "Ha" * 3
print(repeat)  # HaHaHa

# 🔹 String indexing (starts at 0!)
text = "Python"
#        P y t h o n
#        0 1 2 3 4 5
print(text[0])    # P
print(text[3])    # h
print(text[-1])   # n (last character)
print(text[-2])   # o (second to last)

# 🔹 String slicing
print(text[0:3])  # Pyt (index 0, 1, 2 — stops BEFORE 3)
print(text[:3])   # Pyt (same as above, 0 is implied)
print(text[2:])   # thon (from index 2 to end)
print(text[::-1]) # nohtyP (reversed!)

# 🔹 Useful string methods
email = "Alice@Company.COM"
print(email.lower())        # alice@company.com
print(email.upper())        # ALICE@COMPANY.COM
print(email.strip())        # removes leading/trailing spaces
print(email.replace("@", " at "))  # Alice at Company.COM
print(email.split("@"))     # ['Alice', 'Company.COM'] — returns a list!
print(email.startswith("Ali"))     # True
print(email.endswith(".com"))      # False (case-sensitive)
print(email.endswith(".COM"))      # True
print(len(email))           # 16 (length)

# 🔹 Check if something is in a string
sentence = "I love data engineering"
print("data" in sentence)      # True
print "python" in sentence)    # False

# 🔹 f-strings (formatted strings) — THE way to combine variables and text
name = "Alice"
age = 25
city = "Singapore"
print(f"My name is {name}, I am {age} years old, based in {city}.")
# Just put f before the string and use {variable} inside

# f-strings can do math too:
price = 49.90
qty = 3
print(f"Total: ${price * qty:.2f}")  # Total: $149.70
# :.2f means format as float with 2 decimal places

salary = 8500
print(f"Annual: ${salary * 12:,}")  # Annual: $102,000
# :, adds comma separators for thousands
```

**🔥 Practice exercises:**

1. Create variables for your name, age, and city. Print a sentence about yourself using f-string.
2. Given `email = "bob lim@company.com"`, extract just the domain name (everything after @)
3. Given `phone = "+60-12-3456789"`, remove all dashes and the + sign
4. Given `sentence = "  The QUICK brown FOX  "`, convert to lowercase and remove extra spaces
5. Given `filename = "sales_report_2024_05.csv"`, extract just "sales_report" from it (hint: split)

<details>
<summary>📖 Answers</summary>

```python
# 1
name = "Your Name"
age = 22
city = "Kuala Lumpur"
print(f"Hi, I'm {name}, {age} years old, living in {city}.")

# 2
email = "bob lim@company.com"
domain = email.split("@")[1]
print(domain)  # company.com

# 3
phone = "+60-12-3456789"
clean = phone.replace("+", "").replace("-", "")
print(clean)  # 60123456789

# 4
sentence = "  The QUICK brown FOX  "
clean = sentence.strip().lower()
print(clean)  # the quick brown fox

# 5
filename = "sales_report_2024_05.csv"
name_part = filename.replace(".csv", "").rsplit("_", 2)[0]
print(name_part)  # sales_report
# Alternative:
# parts = filename.split("_")
# print("_".join(parts[:2]))  # sales_report
```
</details>

---

### Task 5: Numbers — Integer and Float (20 min)

```python
# 🔹 Integers (whole numbers)
count = 42
negative = -10
big = 1_000_000  # underscores for readability, same as 1000000

# 🔹 Floats (decimal numbers)
price = 19.99
pi = 3.14159
scientific = 1.5e10  # 15000000000.0 (scientific notation)

# 🔹 Arithmetic operators
a = 10
b = 3
print(a + b)    # 13 (addition)
print(a - b)    # 7 (subtraction)
print(a * b)    # 30 (multiplication)
print(a / b)    # 3.333... (division — ALWAYS returns float)
print(a // b)   # 3 (floor division — rounds down)
print(a % b)    # 1 (modulo — remainder after division)
print(a ** b)   # 1000 (exponent — 10 to the power of 3)

# 🔹 Rounding
price = 19.999
print(round(price, 2))    # 20.0 (round to 2 decimals)
print(round(price))       # 20 (round to nearest integer)

# 🔹 Be careful with float precision!
print(0.1 + 0.2)  # 0.30000000000000004 (floating point error!)
# This is why we use DECIMAL in SQL for money, not FLOAT
# In Python, for money, use the decimal module:
from decimal import Decimal
print(Decimal("0.1") + Decimal("0.2"))  # 0.3 (exact!)

# 🔹 Comparison operators (return True/False)
x = 10
print(x == 10)   # True (equal)
print(x != 5)    # True (not equal)
print(x > 5)     # True (greater than)
print(x < 20)    # True (less than)
print(x >= 10)   # True (greater than or equal)
print(x <= 10)   # True (less than or equal)

# 🔹 Logical operators
age = 25
has_degree = True
print(age > 21 and has_degree)   # True (both must be True)
print(age < 18 or has_degree)    # True (at least one True)
print(not has_degree)            # False (opposite)
```

---

### Task 6: Type Conversion (15 min)

```python
# 🔹 Converting between types
# String to number
age_str = "25"
age_int = int(age_str)
age_float = float(age_str)
print(type(age_str))    # <class 'str'>
print(type(age_int))    # <class 'int'>

# Number to string
num = 42
text = str(num)
print(type(text))  # <class 'str'>

# 🔹 Common pitfall
user_input = "25"  # input() always returns string!
# user_input + 5  ← ERROR! Can't add str + int
print(int(user_input) + 5)  # 30 ✅

# 🔹 Boolean conversion
print(bool(1))      # True
print(bool(0))      # False
print(bool(""))     # False (empty string)
print(bool("hello")) # True (non-empty string)
print(bool([]))     # False (empty list)
print(bool([1, 2])) # True (non-empty list)
```

---

### Task 7: input() — Getting User Input (10 min)

```python
# 🔹 input() pauses the program and waits for user to type something
name = input("What is your name? ")
print(f"Hello, {name}!")

# ⚠️ input() ALWAYS returns a string!
age = input("How old are you? ")
print(type(age))  # <class 'str'>
# To do math with it:
age = int(input("How old are you? "))
print(f"In 5 years you'll be {age + 5}")
```

---

## 🌙 BLOCK 3: Practice + Mini-Projects (Evening, ~2 hours)

---

### Task 8: Practice Problems (45 min)

Write a Python script `day08_exercises.py` for each:

1. **Temperature converter:** Ask user for Celsius, convert to Fahrenheit. Formula: F = C * 9/5 + 32
2. **Salary calculator:** Ask for monthly salary, print annual salary, weekly salary (÷52), and daily rate (÷365). Format with 2 decimal places.
3. **String analyzer:** Ask user for a sentence, then print: length, first 5 characters, last 5 characters, all uppercase, word count (hint: split then count)
4. **Simple interest calculator:** Ask for principal, rate, and years. Calculate: interest = principal * rate/100 * years. Show the total amount.
5. **Name formatter:** Ask for first name and last name. Print: full name, initials (e.g., "A.T."), email suggestion (first.last@company.com in lowercase), and username (first initial + last name in lowercase)

<details>
<summary>📖 Answers</summary>

```python
# 1
celsius = float(input("Enter temperature in Celsius: "))
fahrenheit = celsius * 9/5 + 32
print(f"{celsius}°C = {fahrenheit:.1f}°F")

# 2
monthly = float(input("Enter monthly salary: "))
annual = monthly * 12
weekly = annual / 52
daily = annual / 365
print(f"Annual: ${annual:,.2f}")
print(f"Weekly: ${weekly:,.2f}")
print(f"Daily: ${daily:,.2f}")

# 3
sentence = input("Enter a sentence: ")
print(f"Length: {len(sentence)}")
print(f"First 5: {sentence[:5]}")
print(f"Last 5: {sentence[-5:]}")
print(f"Uppercase: {sentence.upper()}")
words = sentence.split()
print(f"Word count: {len(words)}")

# 4
principal = float(input("Principal amount: "))
rate = float(input("Annual interest rate (%): "))
years = float(input("Number of years: "))
interest = principal * rate / 100 * years
total = principal + interest
print(f"Interest: ${interest:,.2f}")
print(f"Total amount: ${total:,.2f}")

# 5
first = input("First name: ")
last = input("Last name: ")
print(f"Full name: {first} {last}")
print(f"Initials: {first[0].upper()}.{last[0].upper()}.")
print(f"Email: {first.lower()}.{last.lower()}@company.com")
print(f"Username: {first[0].lower()}{last.lower()}")
```
</details>

---

### Task 9: FreeCodeCamp — Python Basics (45 min)

Go to https://www.freecodecamp.org/learn/scientific-computing-with-python/

Complete these sections:
- [ ] **Introduction to Python** (read through)
- [ ] **Variables** (do the exercises)

Alternative (if FreeCodeCamp doesn't load): https://www.learnpython.org/
- [ ] **Hello, World!**
- [ ] **Variables and Types**
- [ ] **Lists** (preview for Day 10)
- [ ] **Basic Operators**

---

### Task 10: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 8 — Python Basics
Date: 2026-05-22

## What I Learned
- Variables and naming rules
- Data types: str, int, float, bool
- String operations: indexing, slicing, methods, f-strings
- Number operations: arithmetic, comparison, logical
- Type conversion
- input() and print()

## Key Differences from SQL
- SQL: declare types explicitly. Python: dynamic typing
- SQL: uppercase keywords. Python: snake_case variables
- SQL: declare once. Python: can reassign anytime

## Python vs SQL — My Observation
(Write your own comparison)

## Tomorrow: Control Flow (if/else, loops, functions)
```

```bash
cd data-engineering-journey
mkdir -p python-practice
git add .
git commit -m "Day 8: Python basics — variables, data types, strings, numbers"
git push
```

---

## ✅ Day 8 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Python installed and running | ☐ |
| 2 | VS Code installed with Python extension | ☐ |
| 3 | Wrote and ran first Python script | ☐ |
| 4 | Understand variables and 4 data types | ☐ |
| 5 | Used string indexing, slicing, and methods | ☐ |
| 6 | Used f-strings for formatting | ☐ |
| 7 | Used arithmetic, comparison, logical operators | ☐ |
| 8 | Solved 5 practice problems | ☐ |
| 9 | Completed FreeCodeCamp / learnpython exercises | ☐ |
| 10 | Day 8 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card — Python Basics

```
# Variables
name = "Alice"       # string
age = 25             # int
price = 19.99        # float
active = True        # bool

# Strings
text[0]              # first char
text[-1]             # last char
text[1:4]            # slice
text.lower()         # lowercase
text.split(",")      # split into list
f"Hello {name}"      # f-string

# Numbers
+ - * / // % **      # arithmetic
== != > < >= <=      # comparison
and or not           # logical

# Type conversion
int("42")            # string → int
str(42)              # int → string
float("3.14")        # string → float

# I/O
input("Prompt: ")    # always returns string
print(f"text {var}") # formatted output
```
