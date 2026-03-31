# Project: Using AI to Build a System

**Format:** Individual  
**Grading:** See Rubric Below

**Example System**: Diaper Tracker (https://c2vbesypwfcpm3gsn6e6np.streamlit.app/)

---

## Overview

You will design and build a **Streamlit web application** backed by a **PostgreSQL database** that tracks something you care about. It can be anything — inventory for a small business, a personal book library, a volunteer scheduling system, a recipe manager, a pet adoption tracker — as long as it meets the requirements below.

See my example app (link above) for an example of what you can build. 

Your goal is to demonstrate two things:

1. **That you can plan a system before building it.** You will document your database schema, page layouts, and form designs *before* you write any code.
2. **That you can use AI effectively to build it.** You will use an LLM (like Claude or ChatGPT) to generate your Streamlit code — and you'll submit the prompts you used alongside the finished product.

The students who plan thoroughly up front will have a much easier time prompting the LLM and will spend far less time fixing things. The students who skip the planning and jump straight to code will discover — the hard way — that changing a database schema after you've already built forms on top of it is painful.

---

## How to Use AI Effectively on This Project

This project is designed to be built with AI assistance. But here's the key insight: **an LLM is only as good as the instructions you give it.** If you open Claude and type "build me a Streamlit app that tracks stuff," you'll get something generic that doesn't match your needs and you'll spend hours fixing it. If you give it a detailed specification, you'll get working code on the first or second try.

That's why the documentation phase comes first. Everything you write during the planning phase becomes the input for your AI prompts.

### What Good Documentation Looks Like

Before you prompt any LLM, you should have the following written down:

**1. System Description (1 paragraph)**

A plain-English description of what your system does, who uses it, and what data it tracks. Example:

> *This system tracks diaper distributions for the Vanessa Behan Crisis Nursery. Staff members register approved parents and their children, log diaper inventory by size, and record each distribution event (which parent picked up diapers, for which child, what size, and how many). The system needs to track which parents are approved, which children belong to which parents, and the history of all distributions.*

**2. Entity List with Attributes**

A list of each table, its columns, data types, and constraints. Example:

> **parents** — id (SERIAL PK), first_name (VARCHAR 100, NOT NULL), last_name (VARCHAR 100, NOT NULL), email (VARCHAR 100, UNIQUE, NOT NULL), phone (VARCHAR 20), approved (BOOLEAN, DEFAULT true), created_at (TIMESTAMP, DEFAULT NOW)
>
> **children** — id (SERIAL PK), parent_id (INTEGER FK → parents.id ON DELETE CASCADE), first_name (VARCHAR 100, NOT NULL), age (INTEGER), diaper_size (VARCHAR 10)
>
> **distributions** — id (SERIAL PK), child_id (INTEGER FK → children.id), distributed_by (VARCHAR 100), quantity (INTEGER, NOT NULL), distribution_date (TIMESTAMP, DEFAULT NOW)

**3. Relationships**

Write out which tables connect and how. Example:

> - One parent has many children (one-to-many).
> - One child has many distributions (one-to-many).
> - Parents connect to distributions through children (many-to-many: one parent's multiple children can each have multiple distributions).

**4. Page-by-Page Plan**

For each page in your app, describe what the user sees and what they can do. Example:

> **Home Page** — Shows metrics: total parents, total children, total distributions this month. Displays a table of recent distributions.
>
> **Manage Parents** — Form to add a new parent (first name, last name, email, phone). Table below showing all parents with Edit and Delete buttons. Edit opens a form pre-filled with current values. Delete asks for confirmation.
>
> **Manage Children** — Form to add a child. Parent is selected from a dropdown (pulled from the parents table). Table showing all children with their parent's name.

**5. Validation Rules**

List what validation each form needs. Example:

> - Parent email: must match email regex pattern.
> - Parent phone: must be digits only, 10 characters.
> - Distribution quantity: must be a positive integer.
> - All required fields: cannot be blank.

**6. ERD (Entity-Relationship Diagram)**

A visual diagram of your tables and relationships. Use [dbdiagram.io](https://dbdiagram.io), draw.io, Lucidchart, or a clear hand-drawn photo.

### How to Prompt the LLM

Once your documentation is ready, you can give the LLM very specific prompts. Here's an example of a good prompt:

---

> I'm building a Streamlit app connected to a PostgreSQL database using psycopg2. The connection is stored in `st.secrets["DB_URL"]`.
>
> I have the following table in my database:
>
> ```sql
> CREATE TABLE parents (
>     id SERIAL PRIMARY KEY,
>     first_name VARCHAR(100) NOT NULL,
>     last_name VARCHAR(100) NOT NULL,
>     email VARCHAR(100) NOT NULL UNIQUE,
>     phone VARCHAR(20),
>     approved BOOLEAN DEFAULT true,
>     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
> );
> ```
>
> Please create a Streamlit page called `1_Manage_Parents.py` that does the following:
>
> 1. **Add Parent form** at the top with fields for first_name, last_name, email, and phone. Validate that email matches a standard email regex and that first/last name are not blank. Use parameterized queries (no f-strings in SQL). Show a success message on insert and handle duplicate email errors gracefully.
>
> 2. **Current Parents table** below the form showing all parents ordered by last name.
>
> 3. **Edit functionality** — each row in the table has an Edit button. Clicking it populates a separate edit form with the current values. The user can change fields and submit to update the record.
>
> 4. **Delete functionality** — each row has a Delete button. Clicking it shows a confirmation message. Only delete after confirmation. Use ON DELETE CASCADE since children reference this table.
>
> Use `st.form()` for the add and edit forms. Use `st.columns()` to lay out the Edit/Delete buttons next to each row.

---

Notice what makes this prompt effective:

- It gives the LLM the **exact table schema** so it knows the column names and types.
- It specifies the **connection method** (`st.secrets`, `psycopg2`).
- It describes **every feature** the page needs — not just "make a form."
- It calls out **specific validation rules** and **error handling** expectations.
- It names **specific Streamlit components** to use (`st.form`, `st.columns`).

### What to Save

**Save every prompt you give the LLM and every response you get.** You will submit these as part of your project. This includes:

- Your initial prompts to generate each page.
- Any follow-up prompts where you asked the LLM to fix or modify something.
- Notes on what you had to change manually vs. what the LLM got right.

You can save these as a single document (e.g., `ai_prompts.md` in your repo) or as screenshots. The goal is to show your process, not just the finished product.

---

## Requirements

### 1. Topic & Scope

- **Choose your own topic.** Your system must track something meaningful with at least **3 database tables**, including at least one **many-to-many relationship** (which requires a linking/junction table).
- If you don't have an idea, you may use the **Vanessa Behan Crisis Nursery diaper tracking** system from class.
- Your system should represent a realistic use case — something a small business, nonprofit, club, or individual might actually use.

### 2. Systems Analysis & Design Documents

Before you write any code, you must produce the following documentation:

| Document | Description |
|----------|-------------|
| **System description** | A paragraph explaining what your system tracks, who uses it, and why. |
| **Entity list with attributes** | Every table, every column, data types, and constraints — written out in detail. |
| **Relationships** | A written description of how your tables relate to each other. |
| **Page-by-page plan** | What each page in your app shows and what actions the user can take on it. |
| **Validation rules** | What validation each form requires. |
| **ERD** | A visual entity-relationship diagram (image file in your repo). |

These documents are **required deliverables** — see "What to Submit" below.

### 3. Database Design

- **Minimum 3 tables** with appropriate primary keys, foreign keys, and constraints.
- At least one **many-to-many relationship** implemented with a junction table.
- At least one **date or timestamp** field somewhere in your schema.

### 4. CRUD Operations

Your app must support all four operations:

| Operation | What This Means |
|-----------|----------------|
| **Create** | Users can add new records through a form (e.g., add a new parent, add a new product). |
| **Read** | Users can view data — tables, lists, or details pulled from the database. |
| **Update** | Users can edit existing records (e.g., change a name, update a status). |
| **Delete** | Users can remove records, with a **confirmation step** before deletion (e.g., "Are you sure you want to delete this record?"). |

### 5. Form Validation

- All forms must include **input validation** before writing to the database.
- At minimum: required field checks (no blank submissions) and perhaps **email validation** using a regex pattern on any email field.
- Use appropriate validation for your domain — for example, if you have a phone number field, check the format; if you have a quantity field, make sure it's a positive number.

**Example — Basic Streamlit form with required-field validation:**
```python
import streamlit as st

with st.form("add_contact"):
    name = st.text_input("Name *")
    email = st.text_input("Email *")
    phone = st.text_input("Phone")
    submitted = st.form_submit_button("Submit")

    if submitted:
        errors = []

        if not name.strip():
            errors.append("**Name** is required.")
        if not email.strip():
            errors.append("**Email** is required.")

        if errors:
            for err in errors:
                st.error(err)
        else:
            st.success("Contact added successfully!")
            # write to database here
```

The key idea: collect all validation errors into a list *before* showing them, so the user sees every problem at once rather than fixing them one at a time. Only proceed with the database write when the list is empty.

### 6. Dynamic Dropdowns

- **No hard-coded dropdown options.** Any dropdown or select box in your app must pull its options from a database table. For example, if your form has a "Category" dropdown, those categories must come from a `categories` table — not a Python list in your code.

**Example:**
```python
# ❌ Wrong — hard-coded options
status = st.selectbox("Status", ["Active", "Inactive", "Pending"])

# ✅ Correct — options pulled from the database
conn = get_connection()
cur = conn.cursor()
cur.execute("SELECT id, status_name FROM statuses ORDER BY status_name;")
rows = cur.fetchall()
cur.close()
conn.close()

status_options = {row[1]: row[0] for row in rows}  # {"Active": 1, "Inactive": 2, ...}
selected_status = st.selectbox("Status", options=status_options.keys())
status_id = status_options[selected_status]  # Use this ID when inserting/updating
```

### 7. Search / Filter

- Include at least one **search or filter feature** that lets users narrow down displayed data. Examples: search parents by name, filter distributions by date range, filter inventory by category.

**Example — Search parents by last name:**
```python
import streamlit as st
import pandas as pd

# assume `conn` is your database connection
search = st.text_input("Search parents by last name")

if search.strip():
    query = "SELECT first_name, last_name, email FROM parents WHERE last_name ILIKE :search"
    results = conn.query(query, params={"search": f"%{search.strip()}%"})
else:
    results = conn.query("SELECT first_name, last_name, email FROM parents")

if results.empty:
    st.info("No parents found.")
else:
    st.dataframe(results, use_container_width=True)
```

This gives users a text input that filters the parent list in real time. When the search box is empty, all parents are displayed. The `ILIKE` operator makes the search case-insensitive so "smith", "Smith", and "SMITH" all return the same results.

### 8. Dashboard

- Your app's home page should include a **simple dashboard** showing basic summary counts from your data. At minimum, display the total number of key items your system tracks using `st.metric()` — for example, total parents, total children, and total distributions.
- This does not need to be fancy. A clean row of 3–4 count metrics from live data is sufficient. If you want to go further with charts or breakdowns by category, that's great — but it's not required.

### 9. Code Quality & Security

- Use **parameterized queries** for all database operations. Never use string concatenation or f-strings to build SQL queries — this prevents SQL injection.
  
  ```python
  # ✅ Correct — parameterized
  cur.execute("INSERT INTO parents (name, email) VALUES (%s, %s);", (name, email))

  # ❌ Wrong — SQL injection risk
  cur.execute(f"INSERT INTO parents (name, email) VALUES ('{name}', '{email}');")
  ```

- Use **`st.secrets`** for your database connection URL. No credentials in your code.
- Include **user-friendly error handling** — your app should show helpful messages, not raw Python tracebacks.

### 10. Deployment

- Your app must be **deployed and live** on [Streamlit Community Cloud](https://share.streamlit.io).
- Your GitHub repository must be **public** (required by Streamlit's free tier).
- Make sure your `.streamlit/secrets.toml` file is **not** committed to GitHub. Add it to your `.gitignore`.

### 11. README

Your GitHub repository must include a `README.md` with:

- **Project title and description** — what does your system track and why?
- **Your ERD** — embedded as an image (`![ERD](erd.png)` or similar).
- **Table descriptions** — briefly explain each table and its columns.
- **How to run locally** — what someone would need to do to run your app on their own machine (install dependencies, set up secrets, etc.).
- **Live app URL** — a direct link to your deployed Streamlit app.

---

## Presentation (5 Minutes)

On the due date, you will give a **5-minute presentation** to the class using slides. Structure your time as follows:

| Time | What to Cover |
|------|--------------|
| **~1 minute** | **Live demo.** Open your deployed app and walk through it quickly. Show the main pages, add or edit a record, and show the dashboard. Keep it moving — this is just to show it works. |
| **~4 minutes** | **Reflection and lessons learned.** This is the main focus of your presentation. |

For the reflection portion, address the following:

- **What was challenging or difficult to figure out?** Be specific — was it the Update form? Getting the many-to-many join query right? Deployment? A frustrating Streamlit error? What did you do to work through it?
- **What did you have to change mid-project that you didn't anticipate during the design phase?** This is the most important question. Did you have to add a column? Restructure a table? Add a page you didn't plan for? Realize your relationships were wrong? This is where you learn the value of systems analysis and design — talk honestly about what you missed and what it cost you in time or effort.
- **What would you add or change if you had another week?** What features didn't make the cut? What would you improve about the design, the user experience, or the code?

Your slides should support these talking points. You do not need polished graphics — clear bullet points and a screenshot or two are fine.

---

## What to Submit

Submit **all** of the following through your LMS:

### 1. Links
- **Link to your GitHub repository**
- **Link to your live deployed Streamlit app**

### 2. Systems Analysis & Design Documents
All of the planning documents you created before coding:
- System description
- Entity list with attributes
- Relationships description
- Page-by-page plan
- Validation rules
- ERD diagram

These can be submitted as a single document, multiple files, or included in your GitHub repo — however you organized them.

### 3. AI Prompts & Process
- **Every prompt you gave the LLM** during this project.
- **Notes on what worked vs. what you had to fix** — where did the LLM get it right on the first try, and where did you need to iterate or manually adjust?
- Submit as a document (.md, .doc, or .pdf) or as screenshots in a document.

### 4. Presentation Slides
- Upload your slides (PowerPoint, Google Slides link, or PDF).
- We will have multiple presentations going at the same time, so you will likely present to 7 other students.

### 5. Reflection (5–8 sentences)
A written version of your presentation reflection:
- What does your system track, and why did you choose this topic?
- What was the hardest part of the project?
- What did you change mid-project that you didn't plan for during the design phase?
- If you had another week, what would you add?

---

## Checklist

Your project must meet **all** of the following:

| Requirement | Criteria |
|-------------|----------|
| **Design docs** | Planning documents are submitted and show genuine pre-coding thought. |
| **AI prompts** | Prompts and process notes are submitted showing how you used the LLM. |
| **Database** | At least 3 tables with a many-to-many relationship. At least one date/timestamp field. |
| **Create** | At least one working form that inserts data into the database. |
| **Read** | Data is displayed from the database on at least one page. |
| **Update** | Users can edit at least one type of existing record. |
| **Delete** | Users can delete records with a confirmation step. |
| **Validation** | Forms validate required fields and email or phone format (at minimum). |
| **Dynamic dropdowns** | All dropdowns pull options from database tables. |
| **Search/Filter** | At least one search or filter feature is functional. |
| **Dashboard** | Home page shows at minimum summary counts from live data. |
| **Parameterized SQL** | No string concatenation in SQL queries. This is the way the labs have shown you to do it. So if you are following what we did in the labs, you will be good. |
| **Secrets management** | DB credentials use `st.secrets`, not hard-coded. No secrets in the repo. |
| **Deployed** | App is live on Streamlit Community Cloud and accessible via URL. |
| **ERD** | An entity-relationship diagram is included in the repo. |
| **README** | Includes project description, ERD, table explanations, setup instructions, and live URL. |
| **Presentation** | 5-minute presentation delivered with live demo and reflection on challenges. |
| **Reflection** | Written reflection submitted and shows genuine engagement. |

---

## Tips for Success

- **Do the documentation first. Seriously.** Students who plan their schema, pages, and validation rules before touching code consistently finish faster and with fewer rewrites. Students who skip this step almost always end up restructuring their database halfway through.
- **Your design docs ARE your AI prompts.** The entity list becomes the CREATE TABLE statement you paste into your prompt. The page plan becomes the feature list. The validation rules become the specific requirements. Good planning = good prompts = good code on the first try.
- **Start with Create and Read.** Get data flowing into your database and displaying on screen before tackling Update and Delete.
- **Test locally first.** (Optional but helpful) Use `.streamlit/secrets.toml` for local development, then deploy when things are working.  Follow this [tutorial to get your vscode setup correctly](https://code.visualstudio.com/docs/python/python-tutorial)
- **Commit often.** Push to GitHub frequently so your deployed app stays up to date and you don't lose work.
- **Keep it simple.** A clean, working 3-table app is better than a broken 7-table app. You can always add more once the basics work.

---

## Resources

- [Streamlit Documentation](https://docs.streamlit.io)
- [Streamlit Cheat Sheet](https://cheat-sheet.streamlit.app/)
- [PostgreSQL + psycopg2 Basics](https://www.psycopg.org/docs/usage.html)
- [dbdiagram.io](https://dbdiagram.io) — free ERD tool
- [Streamlit Community Cloud Docs](https://docs.streamlit.io/deploy/streamlit-community-cloud)
- [Class Lab: Streamlit + PostgreSQL walkthrough](https://github.com/mrtimo/Project1/blob/main/streamlit_postgres.md)


## Grading Rubric (100 Points)

The three things I care about most are:

1. **That you planned before you built.** The design documents are the most important deliverable. They show me you thought through your system before prompting an LLM. A thorough set of design docs with a rough app will always beat a polished app with no planning behind it.
2. **That you built something you actually care about.** Pick a topic that means something to you. Students who choose something personally relevant — a system for their small business, a tracker for a hobby, something for a club they're in — consistently produce better work because they understand the problem they're solving.
3. **That you reflected honestly on what happened.** Your presentation and written reflection should tell me what went wrong, what surprised you, and what you had to change mid-project. I don't want a sales pitch for your app — I want to hear what you learned. The students who admit "I had to restructure my entire schema on day three because I didn't think through the relationships" are the ones who actually internalized the lesson.

### Point Breakdown

| Category | Points | Details |
|----------|--------|---------|
| **Design Documents** | **35** | |
| System description | 5 | A clear paragraph explaining what your system does, who uses it, and why it matters to you. |
| Entity list with attributes | 8 | Every table, every column, data types, and constraints — written out before you started coding. Thoroughness matters here. |
| Relationships described | 5 | A written explanation of how your tables connect (one-to-many, many-to-many, etc.). |
| Page-by-page plan | 7 | What each page shows and what actions the user can take — written before coding. |
| Validation rules listed | 5 | What each form checks before writing to the database. |
| ERD included in repo | 5 | A visual diagram of your tables and relationships. |
| **Presentation & Reflection** | **25** | |
| Presentation delivered (5 min) | 10 | Includes a brief live demo (~1 min) and a substantive reflection (~4 min). I'm grading the reflection portion, not your public speaking skills. |
| Written reflection (5–8 sentences) | 10 | Honest engagement with what was hard, what changed mid-project, and what you learned. Specificity matters — name the actual problems you hit. |
| Slides submitted | 5 | Slides support your talking points. They don't need to be fancy. |
| **Working Application** | **25** | |
| Create — at least one working insert form | 4 | A user can add a new record through the app. |
| Read — data displayed from the database | 3 | At least one page shows data pulled from your tables. |
| Update — users can edit existing records | 5 | An edit form pre-populated with current values that writes changes back. |
| Delete — users can delete with confirmation | 4 | A confirmation step before any record is removed. |
| Form validation on all forms | 3 | Required field checks at minimum; appropriate format checks where relevant. |
| Dynamic dropdowns from database | 3 | No hard-coded select options — all pulled from tables. |
| Search or filter feature | 3 | Users can narrow down displayed data in some meaningful way. |
| **Database & Code Quality** | **10** | |
| At least 3 tables with a many-to-many relationship | 4 | Includes a junction table, proper foreign keys, and at least one date/timestamp field. |
| Parameterized SQL throughout | 3 | No f-strings or string concatenation in any query. |
| Secrets management | 3 | Database credentials use `st.secrets`, nothing hard-coded, no secrets in the repo. |
| **Deployment & Documentation** | **5** | |
| App is live on Streamlit Community Cloud | 3 | I can click your link and use your app. |
| README complete | 2 | Includes description, ERD, table explanations, setup instructions, and live URL. |
| **TOTAL** | **100** | |

### AI Prompts & Process (Required but not scored separately)

You must submit every prompt you gave the LLM and notes on what worked vs. what you had to fix. This is a required deliverable — if it's missing, I'll deduct up to **10 points** from your overall score. I'm not grading the quality of your prompts; I'm checking that you documented your process. The prompts are there so I can see how your design docs translated into working code.

### What Will Stand Out

- **Design docs that clearly came before the code.** If your entity list matches your final schema exactly, that's a sign you planned well. If your docs show a different schema than what you ended up with *and your reflection explains why you changed it*, that's even better — it means you learned something.
- **A topic with personal meaning.** "I built this because my family runs a small bakery and we track orders on paper" hits differently than "I picked something random to get the grade."
- **A reflection that names specific failures.** "My original schema didn't have a junction table and I didn't realize I needed one until I tried to build the distribution form" is exactly the kind of learning this project is designed to produce. Don't hide from it — lean into it.
