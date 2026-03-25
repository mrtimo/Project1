# Lab 3: Your First Database with SQLite

## Overview

In Labs 1 and 2 you built forms and pulled data from an API. But where does that data actually live? Somewhere behind PokéAPI there's a **database** — a structured place where data is stored, organized, and retrieved.

In this lab you'll build your own database from scratch, right inside Google Colab. No installs, no servers, no setup. SQLite comes built into Python — you just import it and go.

**Time:** ~45 minutes

**Prerequisite:** Labs 1 and 2

---

## Part 1: What Is a Database?

You already know spreadsheets. A database table is basically a spreadsheet with stricter rules:

| Spreadsheet | Database Table |
|-------------|---------------|
| Columns can be anything | Columns have defined names and types |
| Any cell can hold anything | Each column enforces a type (text, number, etc.) |
| Rows can be incomplete | You decide which fields are required |
| No built-in ID system | Every row gets a unique ID (primary key) |

The big advantage: a database won't let you accidentally put someone's name in the GPA column or forget to include a student's major. It has rules, and it enforces them.

You interact with a database using **SQL** (Structured Query Language). There are really only four commands you need to know right now:

| SQL Command | What It Does | Analogy |
|-------------|-------------|---------|
| `CREATE TABLE` | Set up a new table with column names and types | Creating a blank spreadsheet with headers |
| `INSERT INTO` | Add a new row | Typing a new row in the spreadsheet |
| `SELECT` | Read rows back | Looking at the spreadsheet |
| `DELETE` | Remove a row | Deleting a row from the spreadsheet |

That's it. Let's try it.

---

## Part 2: Creating a Table

Open a **new Google Colab notebook**. In the first cell:

```python
import sqlite3

# Connect to a database (creates it if it doesn't exist)
conn = sqlite3.connect("roster.db")
cursor = conn.cursor()

print("Connected to database!")
```

`conn` is your connection to the database. `cursor` is what you use to run SQL commands. Think of it like opening a spreadsheet — `conn` opens the file, `cursor` is your keyboard.

**New cell — create a table:**

```python
cursor.execute("""
    CREATE TABLE IF NOT EXISTS students (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        major TEXT NOT NULL,
        year INTEGER NOT NULL,
        email TEXT,
        gpa REAL
    )
""")
conn.commit()
print("Table 'students' created!")
```

Let's break this down line by line:

- `CREATE TABLE IF NOT EXISTS students` — "Make a table called students (skip if it already exists)"
- `id INTEGER PRIMARY KEY AUTOINCREMENT` — "Add an ID column that automatically counts up: 1, 2, 3..."
- `name TEXT NOT NULL` — "A name column that holds text. NOT NULL means it's required — you can't leave it blank."
- `email TEXT` — "An email column that holds text. No NOT NULL, so it's optional."
- `gpa REAL` — "A GPA column that holds decimal numbers. Also optional."
- `conn.commit()` — "Save this change." Always commit after making changes.

---

## Part 3: Inserting Data (INSERT INTO)

Now let's add our first student. **New cell:**

```python
cursor.execute("""
    INSERT INTO students (name, major, year, email, gpa)
    VALUES (?, ?, ?, ?, ?)
""", ("Alice Chen", "MIS", 3, "chen@zagmail.gonzaga.edu", 3.8))

conn.commit()
print("Alice added!")
```

The `?` marks are **placeholders**. The actual values go in the tuple at the end: `("Alice Chen", "MIS", 3, ...)`. This is called a **parameterized query** — it keeps your data safe and prevents SQL injection attacks. Always use `?` placeholders, never paste values directly into the SQL string.

Notice we didn't provide an `id` — the database auto-generates it because of `AUTOINCREMENT`.

### Did it actually work? Let's look.

There's a really easy way to see what's in your database using **pandas**. Pandas is already installed in Colab, and it has a function that runs a SQL query and shows you the results as a nice formatted table. **New cell:**

```python
import pandas as pd

pd.read_sql_query("SELECT * FROM students", conn)
```

That's it — one line. You should see a clean table with Alice in it, complete with column headers. Colab renders pandas DataFrames as nicely formatted tables automatically.

Compare that to the manual way of doing it (which we'll see in Part 4). Pandas is a lot less typing.

We'll keep using `pd.read_sql_query()` for the rest of this lab whenever we want to look at the data. Just remember: you pass it the SQL query as a string and `conn` (the connection, not the cursor) as the second argument.

### Add a few more students

**New cell:**

```python
# Insert multiple students at once using executemany
more_students = [
    ("Bob Martinez", "Finance", 2, "martinez@zagmail.gonzaga.edu", 3.2),
    ("Carol Davis", "Marketing", 4, None, 3.5),
    ("Diana Flores", "MIS", 1, "flores@zagmail.gonzaga.edu", None),
]

cursor.executemany("""
    INSERT INTO students (name, major, year, email, gpa)
    VALUES (?, ?, ?, ?, ?)
""", more_students)

conn.commit()

# See the full table
pd.read_sql_query("SELECT * FROM students", conn)
```

Carol has no email (`None`) and Diana has no GPA (`None`). That's fine — we made those columns optional. You should see all four students in the table.

---

## Part 4: Reading Data (SELECT) — The Manual Way

Before we rely entirely on pandas, you should see what's actually happening under the hood. `pd.read_sql_query()` is doing this for you behind the scenes. **New cell:**

```python
cursor.execute("SELECT * FROM students")
rows = cursor.fetchall()

print(f"{'ID':<5} {'Name':<20} {'Major':<14} {'Year':<6} {'GPA':<6} {'Email'}")
print("-" * 70)
for row in rows:
    id, name, major, year, email, gpa = row
    gpa_str = f"{gpa:.1f}" if gpa is not None else "—"
    email_str = email or "—"
    print(f"{id:<5} {name:<20} {major:<14} {year:<6} {gpa_str:<6} {email_str}")
```

`SELECT * FROM students` means "give me all columns from all rows." The `*` means "all columns." `fetchall()` returns a list of tuples — each tuple is one row. Then we loop through and print each row.

This is useful to understand, but for the rest of this lab we'll use pandas because it's so much easier.

### Filtering with WHERE

**You can filter results just like filtering a spreadsheet column. New cell:**

```python
# Only MIS majors
pd.read_sql_query("SELECT name, year FROM students WHERE major = 'MIS'", conn)
```

```python
# Students with GPA above 3.5
pd.read_sql_query("SELECT name, gpa FROM students WHERE gpa > 3.5", conn)
```

The `WHERE` clause filters rows. You can use `=`, `>`, `<`, `>=`, `<=`, and `!=` (not equal).

**Note:** When using pandas, you put the actual values directly in the SQL string (like `'MIS'` and `3.5`) instead of using `?` placeholders. The `?` placeholder style is important when you're accepting user input (to prevent SQL injection), but for simple lookups where you're typing the value yourself, putting it directly in the string is fine and easier to read.

---

## Part 5: Updating Data (UPDATE)

Carol just declared a new major. **New cell:**

```python
cursor.execute("""
    UPDATE students SET major = ? WHERE name = ?
""", ("MIS", "Carol Davis"))

conn.commit()

# Verify the change
pd.read_sql_query("SELECT * FROM students", conn)
```

`UPDATE` changes existing data. `SET major = ?` says what to change. `WHERE name = ?` says which row to change. **Always include a WHERE clause with UPDATE** — without it, you'd change every row in the table.

Carol should now show "MIS" instead of "Marketing".

---

## Part 6: Deleting Data (DELETE)

Bob transferred to another school. **New cell:**

```python
# See who's here before
print("Before delete:")
display(pd.read_sql_query("SELECT id, name FROM students", conn))

# Delete Bob
cursor.execute("DELETE FROM students WHERE id = ?", (2,))
conn.commit()

# See who's here after
print("\nAfter delete:")
pd.read_sql_query("SELECT id, name FROM students", conn)
```

Notice that Bob's ID (2) doesn't get reused. The IDs don't shift down — that's on purpose. IDs are permanent labels, not row numbers.

---

## Part 7: Seeing the Whole Picture

Let's look at the final state of our table. **New cell:**

```python
df = pd.read_sql_query("SELECT * FROM students", conn)
print(f"Students table: {len(df)} records")
df
```

You should see Alice, Carol (now MIS), and Diana — but no Bob.

---

## Part 8: Your Turn

### Task 1: Add two students

Using `cursor.execute()` and `INSERT INTO`, add two students with your own made-up names. Give them different majors and years. At least one should have a GPA and an email, and at least one should have `None` for one of those fields.

After inserting, use `pd.read_sql_query("SELECT * FROM students", conn)` to display the full table and verify they're there.

> **Check:** Your table should now have 5 students (Alice, Carol, Diana, plus your two).

### Task 2: Write a filtered query

Write a `SELECT` query using `pd.read_sql_query()` that only returns students whose year is **3 or higher** (`year >= 3`). Display just their names and years.

> **Check:** You should see Alice (year 3), Carol (year 4), and possibly one of your new students if you made them year 3 or 4.

### Task 3: Update a record

Change one of your new student's major to `"Accounting"` using an `UPDATE` statement. Display the full table afterward to confirm the change.

> **Check:** The student's major should now show Accounting.

### Task 4: Delete and verify

Delete the other student you added (not the one you just updated). Display the table one final time to show it without them.

> **Check:** Your final table should have 4 students.

---

## What You Should Have When You're Done

Your notebook should show:

1. A `students` table created with 6 columns (id, name, major, year, email, gpa)
2. Four original students inserted, verified with pandas
3. The manual print loop (so you've seen both ways)
4. A filtered SELECT showing only MIS students
5. Carol's major updated to MIS
6. Bob deleted
7. Your two new students added (Task 1)
8. A filtered SELECT for year >= 3 (Task 2)
9. One student's major updated to Accounting (Task 3)
10. One student deleted, final table printed (Task 4)

Take a screenshot of your final table and your Task 2 filtered query output, and submit them.

---

## Quick Reference

### Two Ways to See Your Data

```python
# THE EASY WAY — pandas (use this most of the time)
import pandas as pd
pd.read_sql_query("SELECT * FROM students", conn)

# THE MANUAL WAY — cursor + loop (good to understand what's happening)
cursor.execute("SELECT * FROM students")
rows = cursor.fetchall()
for row in rows:
    print(row)
```

### SQL Commands

```sql
-- Create a table
CREATE TABLE IF NOT EXISTS students (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    major TEXT NOT NULL,
    year INTEGER NOT NULL,
    email TEXT,
    gpa REAL
);

-- Insert a row
INSERT INTO students (name, major, year, email, gpa)
VALUES (?, ?, ?, ?, ?);

-- Read all rows
SELECT * FROM students;

-- Read with a filter
SELECT name, gpa FROM students WHERE major = 'MIS';

-- Update a row
UPDATE students SET major = ? WHERE id = ?;

-- Delete a row
DELETE FROM students WHERE id = ?;
```

### Python Pattern

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("mydata.db")    # Open (or create) the database
cursor = conn.cursor()                  # Get a cursor to run commands

cursor.execute("SQL HERE", (values,))   # Run a command
conn.commit()                           # Save changes (after INSERT/UPDATE/DELETE)

pd.read_sql_query("SQL HERE", conn)     # Easy way to see results as a table

rows = cursor.fetchall()                # Manual way — get results as list of tuples
row = cursor.fetchone()                 # Manual way — get one result
```

### Column Types

| SQLite Type | Python Type | Example |
|-------------|------------|---------|
| `TEXT` | `str` | `"Alice Chen"` |
| `INTEGER` | `int` | `3` |
| `REAL` | `float` | `3.8` |
| `NULL` | `None` | `None` |
