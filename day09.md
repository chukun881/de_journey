# 📅 Day 9 — Friday, 23 May 2026
# Python Control Flow: if/else, for/while loops, functions

---

## 🎯 Today's Big Picture
By end of today, you should be able to:
- Write if/elif/else conditions for decision-making
- Use for loops to iterate over sequences
- Use while loops for repetition
- Create and call your own functions
- Understand scope (local vs global variables)
- Handle errors with try/except

**Why this matters:** Control flow is the BRAIN of your program. Without it, your code just runs top to bottom doing the same thing. With it, your code can make decisions, repeat tasks, and handle unexpected situations — essential for writing ETL scripts.

---

## ☀️ BLOCK 1: Conditionals — if/elif/else (Morning, ~1 hour)

---

### Task 1: if/elif/else Basics (30 min)

```python
# 🔹 Basic if statement
age = 20

if age >= 18:
    print("You are an adult")
# ⚠️ INDENTATION MATTERS in Python!
# Everything indented under if runs ONLY when condition is True
# Use 4 spaces (not tab) — VS Code does this automatically

# 🔹 if/else — two options
age = 15
if age >= 18:
    print("Adult")
else:
    print("Minor")

# 🔹 if/elif/else — multiple options
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
elif score >= 60:
    grade = "D"
else:
    grade = "F"
print(f"Score: {score}, Grade: {grade}")

# 🔹 Nested conditions
age = 25
has_experience = True

if age >= 18:
    if has_experience:
        print("Welcome to the team!")
    else:
        print("Try our internship program.")
else:
    print("Too young, sorry.")

# 🔹 One-liner (ternary operator)
age = 20
status = "Adult" if age >= 18 else "Minor"
print(status)

# 🔹 Multiple conditions with and/or
salary = 8500
city = "Singapore"
if salary > 7000 and city == "Singapore":
    print("Comfortable living in SG")
elif salary > 5000 and city == "Kuala Lumpur":
    print("Comfortable living in KL")
```

> 🧠 **Python uses INDENTATION instead of curly braces {} like other languages.** This is Python's most distinctive feature. Get used to it — it makes code readable but you MUST be consistent.

---

### Task 2: Truthy and Falsy (15 min)

```python
# In Python, these are "falsy" (treated as False):
# None, False, 0, 0.0, "", [], {}, set()

# Everything else is "truthy"

# This means you can write clean conditions:
name = ""
if name:  # equivalent to: if name != ""
    print(f"Hello {name}")
else:
    print("No name provided")

data = []
if data:
    print(f"Found {len(data)} items")
else:
    print("No data found")

result = None
if not result:
    print("No result")  # This will print
```

---

### Task 3: Practice — Decision Logic (15 min)

Write a script `day09_conditionals.py`:

1. **Loan qualifier:** Ask for salary and credit score. Approve loan if salary >= 3000 AND credit score >= 700. If salary >= 5000 and credit >= 650, also approve. Otherwise reject.
2. **Shipping calculator:** Ask for order amount and country. If Singapore, shipping is $5. If Malaysia, $10. If order > $100, free shipping. Print total.
3. **Employee bonus:** Ask for years of service and performance rating (A/B/C). Bonus = 20% if A + 5+ years, 15% if A + <5 years, 10% if B + 5+ years, 5% if B + <5 years, 0% if C. Ask for base salary, print bonus amount.

<details>
<summary>📖 Answers</summary>

```python
# 1
salary = float(input("Monthly salary: "))
credit = int(input("Credit score: "))
if (salary >= 3000 and credit >= 700) or (salary >= 5000 and credit >= 650):
    print("✅ Loan approved!")
else:
    print("❌ Loan rejected.")

# 2
amount = float(input("Order amount: $"))
country = input("Country (Singapore/Malaysia): ").strip().lower()

if amount > 100:
    shipping = 0
elif country == "singapore":
    shipping = 5
elif country == "malaysia":
    shipping = 10
else:
    shipping = 15  # international default

total = amount + shipping
if shipping == 0:
    print(f"Free shipping! Total: ${total:.2f}")
else:
    print(f"Shipping: ${shipping}. Total: ${total:.2f}")

# 3
years = int(input("Years of service: "))
rating = input("Performance rating (A/B/C): ").strip().upper()
base = float(input("Base salary: "))

if rating == "A" and years >= 5:
    bonus_pct = 0.20
elif rating == "A":
    bonus_pct = 0.15
elif rating == "B" and years >= 5:
    bonus_pct = 0.10
elif rating == "B":
    bonus_pct = 0.05
else:
    bonus_pct = 0.00

bonus = base * bonus_pct
print(f"Bonus: ${bonus:,.2f} ({bonus_pct*100:.0f}% of base)")
```
</details>

---

## 🔥 BLOCK 2: Loops (Morning/Afternoon, ~1.5 hours)

---

### Task 4: for Loops (30 min)

```python
# 🔹 Loop through a list
cities = ["Singapore", "Kuala Lumpur", "Penang", "Johor Bahru"]
for city in cities:
    print(f"City: {city}")

# 🔹 Loop with index using enumerate
for index, city in enumerate(cities):
    print(f"{index}: {city}")
# 0: Singapore
# 1: Kuala Lumpur
# etc.

# 🔹 Loop with range
for i in range(5):
    print(i)  # 0, 1, 2, 3, 4

for i in range(1, 11):
    print(i)  # 1 to 10

for i in range(0, 20, 3):
    print(i)  # 0, 3, 6, 9, 12, 15, 18 (step by 3)

# 🔹 Loop through a string
for char in "Python":
    print(char)

# 🔹 Nested loops
for i in range(1, 4):
    for j in range(1, 4):
        print(f"{i} x {j} = {i*j}", end="  ")
    print()  # new line after each row

# 🔹 break — exit the loop early
for num in range(100):
    if num * num > 50:
        print(f"First number whose square > 50: {num}")
        break

# 🔹 continue — skip to next iteration
for num in range(10):
    if num % 2 == 0:
        continue  # skip even numbers
    print(num)  # prints only odd: 1, 3, 5, 7, 9
```

**🔥 Practice:**

4. Print a multiplication table (1-12) formatted nicely
5. Find the sum of all numbers from 1 to 100
6. Find all numbers between 1 and 1000 that are divisible by both 3 and 5

<details>
<summary>📖 Answers</summary>

```python
# 4
for i in range(1, 13):
    for j in range(1, 13):
        print(f"{i*j:4}", end="")
    print()

# 5
total = sum(range(1, 101))
print(f"Sum: {total}")  # 5050

# Or with loop:
total = 0
for i in range(1, 101):
    total += i

# 6
result = [n for n in range(1, 1001) if n % 3 == 0 and n % 5 == 0]
print(f"Count: {len(result)}")
print(f"First 10: {result[:10]}")
```
</details>

---

### Task 5: while Loops (20 min)

```python
# 🔹 while loop — repeat while condition is True
count = 0
while count < 5:
    print(f"Count: {count}")
    count += 1  # DON'T FORGET to increment!

# 🔹 while with user input
password = ""
while password != "data123":
    password = input("Enter password: ")
print("Access granted!")

# 🔹 Infinite loop with break (common pattern)
while True:
    command = input("Enter command (type 'quit' to exit): ")
    if command == "quit":
        break
    print(f"You typed: {command}")
```

> 🧠 **for vs while:**
> - **for** → when you know how many times to loop (iterate over a collection)
> - **while** → when you DON'T know how many times (wait for a condition)
> - In data engineering, you'll use for 90% of the time (process each row, each file, each API page)

---

### Task 6: Functions — Reusable Code Blocks (40 min)

Functions are THE most important concept in programming. They let you write code once and reuse it.

```python
# 🔹 Defining a function
def greet(name):
    """Greet a person by name."""  # docstring — explains what function does
    print(f"Hello, {name}!")

# 🔹 Calling (using) the function
greet("Alice")
greet("Bob")

# 🔹 Function with return value
def calculate_tax(salary, tax_rate=0.10):
    """Calculate monthly tax from salary.
    
    Args:
        salary: monthly salary amount
        tax_rate: tax rate as decimal (default 10%)
    
    Returns:
        tax amount
    """
    tax = salary * tax_rate
    return tax

tax = calculate_tax(8500)
print(f"Tax: ${tax:.2f}")  # $850.00

tax_custom = calculate_tax(8500, tax_rate=0.15)
print(f"Tax (15%): ${tax_custom:.2f}")  # $1,275.00

# 🔹 Multiple return values
def analyze_salary(salary):
    annual = salary * 12
    weekly = annual / 52
    daily = annual / 365
    return annual, weekly, daily

annual, weekly, daily = analyze_salary(8500)
print(f"Annual: ${annual:,.0f}, Weekly: ${weekly:,.0f}")

# 🔹 Function with conditional
def categorize_customer(total_spent):
    if total_spent >= 500:
        return "VIP"
    elif total_spent >= 100:
        return "Regular"
    else:
        return "New"

print(categorize_customer(750))  # VIP
print(categorize_customer(50))   # New

# 🔹 Function calling another function
def format_currency(amount):
    return f"${amount:,.2f}"

def print_salary_report(name, salary):
    tax = calculate_tax(salary)
    net = salary - tax
    print(f"=== Salary Report for {name} ===")
    print(f"Gross: {format_currency(salary)}")
    print(f"Tax:   {format_currency(tax)}")
    print(f"Net:   {format_currency(net)}")

print_salary_report("Alice", 8500)
```

**🔥 Practice exercises:**

7. Write a function `celsius_to_fahrenheit(celsius)` that returns the converted temperature
8. Write a function `calculate_bmi(weight_kg, height_m)` that returns BMI and a category ("Underweight" < 18.5, "Normal" 18.5-24.9, "Overweight" 25-29.9, "Obese" >= 30)
9. Write a function `clean_phone(phone)` that takes a messy phone string like "+60-12-345-6789" and returns clean digits: "60123456789"
10. Write a function `count_words(text)` that returns a dictionary with each word and its count. Example: "hello world hello" → {"hello": 2, "world": 1}

<details>
<summary>📖 Answers</summary>

```python
# 7
def celsius_to_fahrenheit(celsius):
    return celsius * 9/5 + 32

print(celsius_to_fahrenheit(37))  # 98.6

# 8
def calculate_bmi(weight_kg, height_m):
    bmi = weight_kg / (height_m ** 2)
    if bmi < 18.5:
        category = "Underweight"
    elif bmi < 25:
        category = "Normal"
    elif bmi < 30:
        category = "Overweight"
    else:
        category = "Obese"
    return round(bmi, 1), category

bmi, cat = calculate_bmi(70, 1.75)
print(f"BMI: {bmi} ({cat})")  # BMI: 22.9 (Normal)

# 9
def clean_phone(phone):
    digits = ""
    for char in phone:
        if char.isdigit():
            digits += char
    return digits

print(clean_phone("+60-12-345-6789"))  # 60123456789

# 10
def count_words(text):
    words = text.lower().split()
    counts = {}
    for word in words:
        if word in counts:
            counts[word] += 1
        else:
            counts[word] = 1
    return counts

print(count_words("hello world hello"))
# {'hello': 2, 'world': 1}
```
</details>

---

## 🔥 BLOCK 3: Error Handling + More Practice (Afternoon, ~1.5 hours)

---

### Task 7: try/except — Handling Errors Gracefully (25 min)

```python
# 🔹 Without error handling — program crashes
# age = int("abc")  ← ValueError! Program stops.

# 🔹 With try/except — program continues
try:
    age = int(input("Enter age: "))
    print(f"You are {age} years old")
except ValueError:
    print("That's not a valid number!")

# 🔹 Multiple exception types
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero!")

# 🔹 Practical example: robust number input
def get_number(prompt):
    """Keep asking until user enters a valid number."""
    while True:
        try:
            return float(input(prompt))
        except ValueError:
            print("Please enter a valid number.")

age = get_number("Enter your age: ")
salary = get_number("Enter salary: ")
print(f"Age: {age}, Salary: ${salary:,.2f}")

# 🔹 try/except/else/finally
try:
    file = open("data.txt", "r")
    content = file.read()
except FileNotFoundError:
    print("File not found!")
else:
    print(f"File has {len(content)} characters")
finally:
    # This ALWAYS runs, whether error or not
    # Use for cleanup (closing files, connections)
    print("Cleanup done")
```

> 🧠 **In data engineering, EVERYTHING can fail.** Network goes down, API returns unexpected data, file is corrupted, database is unreachable. Your code MUST handle errors gracefully. try/except is your safety net.

---

### Task 8: Mini-Project — Simple Data Validator (45 min)

Write `day09_validator.py` — a script that validates employee data:

```python
# Create a data validator that checks a list of employee records
# Each record is a dictionary:
employees = [
    {"name": "Alice Tan", "email": "alice@company.com", "salary": 8500, "age": 28},
    {"name": "Bob", "email": "bob@email", "salary": -5000, "age": 25},  # invalid email, negative salary
    {"name": "", "email": "charlie@company.com", "salary": 6000, "age": 30},  # empty name
    {"name": "Diana Chen", "email": "diana@company.com", "salary": 7000, "age": 17},  # under 18
    {"name": "Eve", "email": "eve@company.com", "salary": 9000, "age": 35},
]

# Write functions:
# validate_name(name) → True if non-empty string
# validate_email(email) → True if contains @ and .
# validate_salary(salary) → True if positive number
# validate_age(age) → True if >= 18

# Then loop through all employees, validate each field,
# and print a report of which records are valid/invalid

# Example output:
# ✅ Employee 1 (Alice Tan): All valid
# ❌ Employee 2 (Bob): Invalid email (missing .), Invalid salary (negative)
# ❌ Employee 3 (): Invalid name (empty)
# ❌ Employee 4 (Diana Chen): Invalid age (under 18)
# ✅ Employee 5 (Eve): All valid
```

<details>
<summary>📖 Answer</summary>

```python
def validate_name(name):
    return isinstance(name, str) and len(name.strip()) > 0

def validate_email(email):
    return isinstance(email, str) and "@" in email and "." in email.split("@")[-1]

def validate_salary(salary):
    return isinstance(salary, (int, float)) and salary > 0

def validate_age(age):
    return isinstance(age, (int, float)) and age >= 18

employees = [
    {"name": "Alice Tan", "email": "alice@company.com", "salary": 8500, "age": 28},
    {"name": "Bob", "email": "bob@email", "salary": -5000, "age": 25},
    {"name": "", "email": "charlie@company.com", "salary": 6000, "age": 30},
    {"name": "Diana Chen", "email": "diana@company.com", "salary": 7000, "age": 17},
    {"name": "Eve", "email": "eve@company.com", "salary": 9000, "age": 35},
]

valid_count = 0
for i, emp in enumerate(employees, 1):
    errors = []
    if not validate_name(emp["name"]):
        errors.append("Invalid name (empty)")
    if not validate_email(emp["email"]):
        errors.append("Invalid email")
    if not validate_salary(emp["salary"]):
        errors.append("Invalid salary")
    if not validate_age(emp["age"]):
        errors.append("Invalid age (under 18)")
    
    if errors:
        print(f"❌ Employee {i} ({emp['name']}): {', '.join(errors)}")
    else:
        print(f"✅ Employee {i} ({emp['name']}): All valid")
        valid_count += 1

print(f"\nSummary: {valid_count}/{len(employees)} valid records")
```
</details>

---

### Task 9: FreeCodeCamp / Practice (30 min)

Continue on https://www.learnpython.org/ or https://www.freecodecamp.org/learn/:

- [ ] **Conditions** (if/else exercises)
- [ ] **Loops** (for/while exercises)
- [ ] **Functions** (def, return exercises)

---

### Task 10: Daily Reflection + GitHub Push (15 min)

```markdown
# Day 9 — Control Flow & Functions
Date: 2026-05-23

## Key Concepts
- if/elif/else → 
- for loop → 
- while loop → 
- def (functions) → 
- return → 
- try/except → 

## Data Validator Project
- Validated X employee records
- Found X invalid records

## Tomorrow: Data Structures (lists, dicts, sets, tuples)
```

```bash
git add .
git commit -m "Day 9: Control flow, functions, error handling, data validator"
git push
```

---

## ✅ Day 9 Complete Checklist

| # | Task | Done? |
|---|------|-------|
| 1 | Wrote if/elif/else conditions | ☐ |
| 2 | Solved 3 conditional exercises | ☐ |
| 3 | Used for loops with range, enumerate, break, continue | ☐ |
| 4 | Used while loops | ☐ |
| 5 | Created functions with parameters, defaults, return values | ☐ |
| 6 | Solved 4 function exercises | ☐ |
| 7 | Used try/except for error handling | ☐ |
| 8 | Built data validator mini-project | ☐ |
| 9 | Completed online exercises | ☐ |
| 10 | Day 9 notes pushed to GitHub | ☐ |

---

## 📌 Quick Reference Card

```python
# Conditions
if condition:
    ...
elif condition:
    ...
else:
    ...

# Loops
for item in collection:
    ...
for i in range(10):
    ...
while condition:
    ...

# Functions
def name(param1, param2="default"):
    """Docstring."""
    return value

# Error handling
try:
    risky_operation()
except ValueError:
    handle_error()
finally:
    cleanup()
```
