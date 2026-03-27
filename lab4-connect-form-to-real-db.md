# Lab 4: Connect Your Form to Your Database

## Overview

In Lab 1 you built a form. In Lab 3 you learned SQL. In Lab 3.5 you set up a cloud PostgreSQL database on Retool. Now you'll connect them — your form will write to and read from your **real cloud database**. No API in between, no server to run. Just: button click → SQL command → display the results.

Because your database lives in the cloud, **your data will persist** even when Colab disconnects. You can close your laptop, come back tomorrow, and your data will still be there.

We're also going to go further than a single table. Real applications almost never have just one table. In this lab, we'll build **three tables** that work together using a **many-to-many relationship** — one of the most common and important patterns in database design.

**Time:** ~60 minutes

**Prerequisite:** Labs 1, 2, 3, and 3.5

---

## Part 1: Why Many-to-Many?

Think about students and courses at Gonzaga. A single student takes multiple courses. A single course has multiple students enrolled. This is a **many-to-many relationship** — many students connect to many courses, and many courses connect to many students.

You see this pattern everywhere:

- **Doctors ↔ Patients** — A doctor treats many patients. A patient may see many doctors.
- **Authors ↔ Books** — An author can write many books. A book can have many authors.
- **Actors ↔ Movies** — An actor appears in many movies. A movie has many actors.
- **Employees ↔ Projects** — An employee works on many projects. A project has many employees.
- **Playlists ↔ Songs** — A playlist contains many songs. A song can appear on many playlists.

### The Problem: You Can't Do This With Two Tables

You might think you could put a `course` column in the students table. But what happens when Alice takes three courses? You'd need three rows for Alice — and her name, email, and GPA would be duplicated in all three rows. That's messy, wasteful, and a recipe for bugs.

You also can't put a `student` column in the courses table for the same reason.

### The Solution: A Junction Table

The standard solution is to create a **third table** — often called a **junction table**, **linking table**, or **bridge table** — that sits between the other two. Each row in the junction table represents one relationship: "this student is enrolled in this course."

Here's what our three tables look like:

```
┌──────────────────┐       ┌──────────────────────┐       ┌──────────────────┐
│     students     │       │    enrollments        │       │     courses      │
│──────────────────│       │──────────────────────│       │──────────────────│
│ id (PK)          │──┐    │ id (PK)              │    ┌──│ id (PK)          │
│ name             │  └───→│ student_id (FK)      │    │  │ course_name      │
│ major            │       │ course_id (FK)       │←───┘  │ instructor       │
│ year             │       │ enrolled_date        │       │ credits          │
│ email            │       └──────────────────────┘       └──────────────────┘
│ gpa              │
└──────────────────┘
```

- **PK** = Primary Key (unique identifier for each row)
- **FK** = Foreign Key (a reference to a row in another table)

The `enrollments` table doesn't hold student or course data — it just holds **pairs of IDs** that say "student #2 is in course #5." This is clean, efficient, and flexible.

### Why This Matters

This pattern shows up in virtually every real application. If you look at the database behind Zagweb, there are tables for students, tables for courses, and a junction table connecting them (with extra info like the grade, the semester, etc.). Instagram has users, posts, and a "likes" junction table. Amazon has customers, products, and an "orders" junction table.

Understanding this pattern is essential for building real systems.

---

## Part 2: Connect to Your Database

Open a **new Google Colab notebook**. We'll connect to your Retool PostgreSQL database and build three tables.

### Step 1: Install the PostgreSQL library

**Cell 1 — Install psycopg2:**

```python
!pip install psycopg2-binary
```

### Step 2: Enable your database secret

In Lab 3.5, you saved your Retool database connection URL as a Colab Secret called `DB_URL`. If you're in a new notebook, you need to make sure it's available:

1. Click the **🔑 Secrets** icon (key icon) in the left sidebar.
2. You should see `DB_URL` listed. If the **"Notebook access"** toggle is off, **turn it on** for this notebook.
3. If you don't see `DB_URL` at all, go back to Retool, copy your connection URL, and add it as a new secret (see Lab 3.5 for instructions).

### Step 3: Connect to the database

**Cell 2 — Connect to your Retool PostgreSQL database:**

```python
import psycopg2
from google.colab import userdata

# Get the connection URL from Colab Secrets (NEVER paste it directly in code!)
db_url = userdata.get('DB_URL')

# Connect to the database
conn = psycopg2.connect(db_url)
conn.autocommit = True
cursor = conn.cursor()

print("Connected to your PostgreSQL database!")
```

> **Why `autocommit = True`?** In PostgreSQL, if a SQL error occurs during a transaction, all subsequent commands will fail until you explicitly roll back. Setting `autocommit = True` means each command runs independently, so one error won't break everything that follows. This makes it much easier to experiment in a notebook. In a production app you'd manage transactions more carefully, but for learning this is the right setting.

If you see "Connected to your PostgreSQL database!" you're good to go. This is the same database you set up in Lab 3.5 — but now we're going to build real tables and forms on top of it.

### Step 4: Create all three tables

**Cell 3 — Create the tables:**

```python
# --- Table 1: Students ---
cursor.execute("""
    CREATE TABLE IF NOT EXISTS students (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        major VARCHAR(100) NOT NULL,
        year INTEGER NOT NULL,
        email VARCHAR(200),
        gpa NUMERIC(3, 2)
    )
""")

# --- Table 2: Courses ---
cursor.execute("""
    CREATE TABLE IF NOT EXISTS courses (
        id SERIAL PRIMARY KEY,
        course_name VARCHAR(200) NOT NULL,
        instructor VARCHAR(100) NOT NULL,
        credits INTEGER NOT NULL
    )
""")

# --- Table 3: Enrollments (the junction table) ---
cursor.execute("""
    CREATE TABLE IF NOT EXISTS enrollments (
        id SERIAL PRIMARY KEY,
        student_id INTEGER NOT NULL REFERENCES students(id),
        course_id INTEGER NOT NULL REFERENCES courses(id),
        enrolled_date DATE DEFAULT CURRENT_DATE
    )
""")

print("All three tables created!")
```

Notice a few differences from the SQLite we used in Lab 3:

| SQLite | PostgreSQL | Why |
|--------|-----------|-----|
| `INTEGER PRIMARY KEY AUTOINCREMENT` | `SERIAL PRIMARY KEY` | PostgreSQL's way of auto-incrementing IDs |
| `TEXT` | `VARCHAR(100)` | PostgreSQL lets you set maximum lengths |
| `REAL` | `NUMERIC(3, 2)` | More precise decimal numbers |
| `FOREIGN KEY (...) REFERENCES ...` (separate line) | `REFERENCES students(id)` (inline) | Both work, inline is more concise |

The **concepts are identical** — tables, primary keys, foreign keys, junction tables. Only the syntax is slightly different.

Look at the `enrollments` table carefully. It has two **foreign keys** — `student_id` points to the `students` table and `course_id` points to the `courses` table. Each row says "this student is enrolled in this course on this date."

### Step 5: Add starter data

**Cell 4 — Insert starter data:**

```python
# Check if we already have data (so we don't duplicate on re-runs)
cursor.execute("SELECT COUNT(*) FROM students")
count = cursor.fetchone()[0]

if count == 0:
    # Insert some students
    starter_students = [
        ("Alice Chen", "MIS", 3, "chen@zagmail.gonzaga.edu", 3.8),
        ("Bob Martinez", "Finance", 2, "martinez@zagmail.gonzaga.edu", 3.2),
        ("Carol Davis", "Marketing", 4, "davis@zagmail.gonzaga.edu", 3.5),
        ("Derek Johnson", "MIS", 2, "johnson@zagmail.gonzaga.edu", 3.6),
    ]

    for s in starter_students:
        cursor.execute("""
            INSERT INTO students (name, major, year, email, gpa)
            VALUES (%s, %s, %s, %s, %s)
        """, s)

    # Insert some courses
    starter_courses = [
        ("BMIS 490 - Systems Analysis", "Prof. Olsen", 3),
        ("BMIS 340 - Database Management", "Prof. Olsen", 3),
        ("ACCT 201 - Intro to Accounting", "Prof. Williams", 3),
        ("MKTG 310 - Marketing Research", "Prof. Garcia", 3),
    ]

    for c in starter_courses:
        cursor.execute("""
            INSERT INTO courses (course_name, instructor, credits)
            VALUES (%s, %s, %s)
        """, c)

    # Get the actual IDs that were assigned
    cursor.execute("SELECT id FROM students ORDER BY id")
    student_ids = [row[0] for row in cursor.fetchall()]

    cursor.execute("SELECT id FROM courses ORDER BY id")
    course_ids = [row[0] for row in cursor.fetchall()]

    # Enroll some students in courses
    starter_enrollments = [
        (student_ids[0], course_ids[0]),  # Alice in BMIS 490
        (student_ids[0], course_ids[1]),  # Alice in BMIS 340
        (student_ids[1], course_ids[2]),  # Bob in ACCT 201
        (student_ids[2], course_ids[3]),  # Carol in MKTG 310
        (student_ids[2], course_ids[0]),  # Carol in BMIS 490
        (student_ids[3], course_ids[0]),  # Derek in BMIS 490
        (student_ids[3], course_ids[1]),  # Derek in BMIS 340
    ]

    for e in starter_enrollments:
        cursor.execute("""
            INSERT INTO enrollments (student_id, course_id)
            VALUES (%s, %s)
        """, e)

    print(f"Loaded {len(starter_students)} students, {len(starter_courses)} courses, and {len(starter_enrollments)} enrollments.")
else:
    print(f"Data already exists ({count} students found). Skipping starter data.")
```

> **Why check first?** Because your database is persistent, running this cell a second time would duplicate all the data. The `if count == 0` check prevents that. This is a real-world pattern — you always want to be careful about duplicate inserts.

> **Why not hardcode IDs like `(1, 1)` for enrollments?** In PostgreSQL, `SERIAL` IDs don't always start at 1 — especially if you've created and deleted rows before. By querying the actual IDs after inserting, we make sure the enrollments point to the right students and courses regardless of what ID numbers PostgreSQL assigned.

**Cell 5 — Verify each table:**

```python
print("=== STUDENTS ===")
cursor.execute("SELECT * FROM students ORDER BY id")
for row in cursor.fetchall():
    print(row)

print("\n=== COURSES ===")
cursor.execute("SELECT * FROM courses ORDER BY id")
for row in cursor.fetchall():
    print(row)

print("\n=== ENROLLMENTS ===")
cursor.execute("SELECT * FROM enrollments ORDER BY id")
for row in cursor.fetchall():
    print(row)
```

You should see four students, four courses, and seven enrollments. Notice how the enrollments table just stores pairs of IDs — it's lightweight and flexible.

**Now go to Retool and check!** Open your Retool Database in your browser and refresh the page. You should see your three new tables — `students`, `courses`, and `enrollments` — with all the data you just inserted from Colab. This is the power of a cloud database: you can add data from code and see it in a visual tool.

---

## Part 3: The Power of JOIN

The real payoff of this design is the **JOIN** — a SQL command that combines data from multiple tables. Instead of seeing `student_id: 1, course_id: 2`, we can see `Alice Chen is enrolled in BMIS 340`.

**Cell 6 — See who is enrolled in what:**

```python
print("=== ENROLLMENT ROSTER ===")
print(f"{'Student':<20} {'Course':<35} {'Enrolled'}")
print("-" * 70)

cursor.execute("""
    SELECT students.name, courses.course_name, enrollments.enrolled_date
    FROM enrollments
    JOIN students ON enrollments.student_id = students.id
    JOIN courses ON enrollments.course_id = courses.id
    ORDER BY courses.course_name, students.name
""")

for name, course, date in cursor.fetchall():
    print(f"{name:<20} {course:<35} {date}")
```

This single query reaches across all three tables and gives us a human-readable enrollment roster. This is the exact same kind of query that runs behind the scenes when you log into Zagweb and see your schedule.

**Cell 7 — See all students in a specific course:**

```python
course_name = "BMIS 490 - Systems Analysis"

cursor.execute("""
    SELECT students.name, students.major, students.year
    FROM enrollments
    JOIN students ON enrollments.student_id = students.id
    JOIN courses ON enrollments.course_id = courses.id
    WHERE courses.course_name = %s
    ORDER BY students.name
""", (course_name,))

print(f"Students enrolled in {course_name}:")
for name, major, year in cursor.fetchall():
    print(f"  {name} ({major}, Year {year})")
```

**Cell 8 — See all courses a specific student is taking:**

```python
student_name = "Alice Chen"

cursor.execute("""
    SELECT courses.course_name, courses.instructor, courses.credits
    FROM enrollments
    JOIN courses ON enrollments.course_id = courses.id
    JOIN students ON enrollments.student_id = students.id
    WHERE students.name = %s
    ORDER BY courses.course_name
""", (student_name,))

print(f"Courses for {student_name}:")
for course, instructor, credits in cursor.fetchall():
    print(f"  {course} ({instructor}, {credits} credits)")
```

---

## Part 4: Build the Interactive Forms

Now let's build forms with widgets that let you manage all three tables and the relationships between them. We'll build this in sections.

### Section A: Student Management Form

**Cell 9 — Add and view students:**

```python
import ipywidgets as widgets
from IPython.display import display

# ========================================
# STUDENT FORM WIDGETS
# ========================================
majors = ["MIS", "Finance", "Marketing", "Accounting", "Management", "Economics"]

name_input = widgets.Text(description="Name:", placeholder="e.g. Jamie Smith", style={"description_width": "80px"})
major_dropdown = widgets.Dropdown(options=majors, description="Major:", style={"description_width": "80px"})
year_slider = widgets.IntSlider(value=1, min=1, max=4, description="Year:", style={"description_width": "80px"})
gpa_slider = widgets.FloatSlider(value=3.0, min=0.0, max=4.0, step=0.1, description="GPA:", style={"description_width": "80px"})
email_input = widgets.Text(description="Email:", placeholder="username@zagmail.gonzaga.edu", style={"description_width": "80px"})
add_student_btn = widgets.Button(description="Add Student", button_style="success", icon="plus")

student_status = widgets.Output()
student_list = widgets.Output()

def refresh_students():
    """Display all students."""
    student_list.clear_output()
    with student_list:
        cursor.execute("SELECT * FROM students ORDER BY id")
        rows = cursor.fetchall()
        if not rows:
            print("No students yet.")
            return
        print(f"{'ID':<5} {'Name':<22} {'Major':<14} {'Year':<6} {'GPA':<6} {'Email'}")
        print("-" * 80)
        for id, name, major, year, email, gpa in rows:
            gpa_str = f"{gpa:.1f}" if gpa else "—"
            email_str = email or "—"
            print(f"{id:<5} {name:<22} {major:<14} {year:<6} {gpa_str:<6} {email_str}")

def on_add_student(b):
    student_status.clear_output()
    with student_status:
        if not name_input.value.strip():
            print("Please enter a name.")
            return
        cursor.execute("""
            INSERT INTO students (name, major, year, email, gpa)
            VALUES (%s, %s, %s, %s, %s)
        """, (name_input.value, major_dropdown.value, year_slider.value,
              email_input.value or None, gpa_slider.value))
        print(f"Added: {name_input.value}")
        name_input.value = ""
        email_input.value = ""
    refresh_students()

add_student_btn.on_click(on_add_student)

student_form = widgets.VBox([
    widgets.HTML("<h2>👤 Student Manager</h2>"),
    widgets.HTML("<b>Add a Student</b>"),
    name_input, major_dropdown, year_slider, gpa_slider, email_input,
    add_student_btn,
    student_status,
    widgets.HTML("<h3>All Students</h3>"),
    student_list
], layout=widgets.Layout(padding="15px", border="2px solid #2e7d32", border_radius="10px", width="70%"))

display(student_form)
refresh_students()
```

### Section B: Course Management Form

**Cell 10 — Add and view courses:**

```python
# ========================================
# COURSE FORM WIDGETS
# ========================================
course_name_input = widgets.Text(description="Course:", placeholder="e.g. BMIS 490 - Systems Analysis", style={"description_width": "80px"},
                                  layout=widgets.Layout(width="400px"))
instructor_input = widgets.Text(description="Instructor:", placeholder="e.g. Prof. Smith", style={"description_width": "80px"})
credits_dropdown = widgets.Dropdown(options=[1, 2, 3, 4], value=3, description="Credits:", style={"description_width": "80px"})
add_course_btn = widgets.Button(description="Add Course", button_style="success", icon="plus")

course_status = widgets.Output()
course_list = widgets.Output()

def refresh_courses():
    """Display all courses."""
    course_list.clear_output()
    with course_list:
        cursor.execute("SELECT * FROM courses ORDER BY id")
        rows = cursor.fetchall()
        if not rows:
            print("No courses yet.")
            return
        print(f"{'ID':<5} {'Course Name':<38} {'Instructor':<20} {'Credits'}")
        print("-" * 70)
        for id, name, instructor, credits in rows:
            print(f"{id:<5} {name:<38} {instructor:<20} {credits}")

def on_add_course(b):
    course_status.clear_output()
    with course_status:
        if not course_name_input.value.strip():
            print("Please enter a course name.")
            return
        if not instructor_input.value.strip():
            print("Please enter an instructor name.")
            return
        cursor.execute("""
            INSERT INTO courses (course_name, instructor, credits)
            VALUES (%s, %s, %s)
        """, (course_name_input.value, instructor_input.value, credits_dropdown.value))
        print(f"Added: {course_name_input.value}")
        course_name_input.value = ""
        instructor_input.value = ""
    refresh_courses()

add_course_btn.on_click(on_add_course)

course_form = widgets.VBox([
    widgets.HTML("<h2>📚 Course Manager</h2>"),
    widgets.HTML("<b>Add a Course</b>"),
    course_name_input, instructor_input, credits_dropdown,
    add_course_btn,
    course_status,
    widgets.HTML("<h3>All Courses</h3>"),
    course_list
], layout=widgets.Layout(padding="15px", border="2px solid #1565c0", border_radius="10px", width="70%"))

display(course_form)
refresh_courses()
```

### Section C: Enrollment Manager (The Junction Table in Action)

This is where the many-to-many relationship comes to life. This form lets you pick a student and a course and connect them.

**Cell 11 — Enroll students in courses:**

```python
# ========================================
# ENROLLMENT FORM WIDGETS
# ========================================

def get_student_options():
    """Fetch students for the dropdown."""
    cursor.execute("SELECT id, name FROM students ORDER BY name")
    return {name: id for id, name in cursor.fetchall()}

def get_course_options():
    """Fetch courses for the dropdown."""
    cursor.execute("SELECT id, course_name FROM courses ORDER BY course_name")
    return {name: id for id, name in cursor.fetchall()}

# Dropdowns that show names but store IDs
student_options = get_student_options()
course_options = get_course_options()

enroll_student_dd = widgets.Dropdown(
    options=student_options,
    description="Student:",
    style={"description_width": "80px"}
)

enroll_course_dd = widgets.Dropdown(
    options=course_options,
    description="Course:",
    style={"description_width": "80px"}
)

enroll_btn = widgets.Button(description="Enroll Student", button_style="success", icon="link")
refresh_dd_btn = widgets.Button(description="Refresh Dropdowns", button_style="info", icon="refresh")

enroll_status = widgets.Output()
enrollment_list = widgets.Output()

def refresh_enrollments():
    """Display all enrollments using a JOIN."""
    enrollment_list.clear_output()
    with enrollment_list:
        cursor.execute("""
            SELECT enrollments.id, students.name, courses.course_name, enrollments.enrolled_date
            FROM enrollments
            JOIN students ON enrollments.student_id = students.id
            JOIN courses ON enrollments.course_id = courses.id
            ORDER BY students.name, courses.course_name
        """)
        rows = cursor.fetchall()
        if not rows:
            print("No enrollments yet.")
            return
        print(f"{'ID':<5} {'Student':<22} {'Course':<38} {'Date'}")
        print("-" * 75)
        for id, student, course, date in rows:
            print(f"{id:<5} {student:<22} {course:<38} {date}")

def on_enroll(b):
    enroll_status.clear_output()
    with enroll_status:
        student_id = enroll_student_dd.value
        course_id = enroll_course_dd.value

        # Check if already enrolled
        cursor.execute("""
            SELECT id FROM enrollments
            WHERE student_id = %s AND course_id = %s
        """, (student_id, course_id))

        if cursor.fetchone():
            print("This student is already enrolled in this course!")
            return

        cursor.execute("""
            INSERT INTO enrollments (student_id, course_id)
            VALUES (%s, %s)
        """, (student_id, course_id))
        print(f"Enrolled successfully!")
    refresh_enrollments()

def on_refresh_dropdowns(b):
    """Update dropdowns to include any newly added students/courses."""
    enroll_student_dd.options = get_student_options()
    enroll_course_dd.options = get_course_options()
    enroll_status.clear_output()
    with enroll_status:
        print("Dropdowns refreshed!")

enroll_btn.on_click(on_enroll)
refresh_dd_btn.on_click(on_refresh_dropdowns)

enrollment_form = widgets.VBox([
    widgets.HTML("<h2>🔗 Enrollment Manager</h2>"),
    widgets.HTML("<p><i>This form writes to the <b>enrollments</b> junction table, linking students to courses.</i></p>"),
    widgets.HTML("<b>Enroll a Student in a Course</b>"),
    enroll_student_dd, enroll_course_dd,
    widgets.HBox([enroll_btn, refresh_dd_btn]),
    enroll_status,
    widgets.HTML("<h3>All Enrollments <span style='font-weight:normal; color:#555;'>(JOIN across all 3 tables)</span></h3>"),
    enrollment_list
], layout=widgets.Layout(padding="15px", border="2px solid #6a1b9a", border_radius="10px", width="70%"))

display(enrollment_form)
refresh_enrollments()
```

**Important:** If you added new students or courses using the forms above, click **Refresh Dropdowns** so they appear in the enrollment form's dropdown menus.

### Section D: Course Roster Viewer

This form lets you pick a course and see who is enrolled — exactly like an instructor looking at their class roster.

**Cell 12 — View a course roster:**

```python
# ========================================
# COURSE ROSTER VIEWER
# ========================================
roster_course_dd = widgets.Dropdown(
    options=get_course_options(),
    description="Course:",
    style={"description_width": "80px"}
)
view_roster_btn = widgets.Button(description="View Roster", button_style="info", icon="users")
refresh_roster_dd_btn = widgets.Button(description="Refresh", button_style="", icon="refresh")
roster_view = widgets.Output()

def on_view_roster(b):
    roster_view.clear_output()
    with roster_view:
        course_id = roster_course_dd.value
        cursor.execute("""
            SELECT students.name, students.major, students.year, students.email
            FROM enrollments
            JOIN students ON enrollments.student_id = students.id
            WHERE enrollments.course_id = %s
            ORDER BY students.name
        """, (course_id,))
        rows = cursor.fetchall()

        # Get course name for the header
        cursor.execute("SELECT course_name FROM courses WHERE id = %s", (course_id,))
        course_name = cursor.fetchone()[0]

        if not rows:
            print(f"No students enrolled in {course_name}.")
            return

        print(f"Roster for: {course_name}")
        print(f"Enrolled: {len(rows)} student(s)\n")
        print(f"{'Name':<22} {'Major':<14} {'Year':<6} {'Email'}")
        print("-" * 60)
        for name, major, year, email in rows:
            email_str = email or "—"
            print(f"{name:<22} {major:<14} {year:<6} {email_str}")

def on_refresh_roster_dd(b):
    roster_course_dd.options = get_course_options()

view_roster_btn.on_click(on_view_roster)
refresh_roster_dd_btn.on_click(on_refresh_roster_dd)

roster_form = widgets.VBox([
    widgets.HTML("<h2>📋 Course Roster Viewer</h2>"),
    widgets.HTML("<p><i>Select a course to see which students are enrolled.</i></p>"),
    roster_course_dd,
    widgets.HBox([view_roster_btn, refresh_roster_dd_btn]),
    roster_view
], layout=widgets.Layout(padding="15px", border="2px solid #e65100", border_radius="10px", width="70%"))

display(roster_form)
```

---

## Part 5: Trace the Data Flow

Here's what happens when you enroll a student in a course:

```
1. You pick "Alice Chen" from the student dropdown (value = 1)
2. You pick "ACCT 201 - Intro to Accounting" from the course dropdown (value = 3)
3. You click "Enroll Student"
         ↓
4. on_enroll() reads: enroll_student_dd.value → 1
                      enroll_course_dd.value → 3
         ↓
5. It checks: SELECT id FROM enrollments WHERE student_id = 1 AND course_id = 3
   (Is Alice already in ACCT 201? No → continue)
         ↓
6. It runs: INSERT INTO enrollments (student_id, course_id) VALUES (1, 3)
         ↓
7. The data is saved to your cloud PostgreSQL database
         ↓
8. refresh_enrollments() runs a JOIN across all 3 tables:
   SELECT students.name, courses.course_name, enrollments.enrolled_date
   FROM enrollments
   JOIN students ON enrollments.student_id = students.id
   JOIN courses ON enrollments.course_id = courses.id
         ↓
9. The display updates and you see "Alice Chen — ACCT 201" in the list
```

Notice that the `enrollments` table only stored two numbers: `1` and `3`. The JOIN is what turns those numbers back into "Alice Chen" and "ACCT 201." The actual student and course data only exists in one place — their own tables.

**Try this:** After adding some data, go to **Retool Database** in your browser and refresh the page. You should see all the data you just added from Colab. Then close your Colab notebook, reopen it, reconnect to the database, and your data is still there. That's the whole point of a cloud database.

---

## Part 6: Your Turn — Challenges

### Challenge 1: Drop a Course

Add a way to **remove an enrollment** (drop a course). You will need:

1. A `widgets.BoundedIntText` for the enrollment ID to remove (you can see enrollment IDs in the enrollment list)
2. A `widgets.Button` with description `"Drop Course"`, button_style `"danger"`, icon `"unlink"`
3. A callback that runs: `DELETE FROM enrollments WHERE id = %s`

Add these to the enrollment form. Don't forget to call `refresh_enrollments()`.

> **Check:** After dropping an enrollment, that row should disappear from the enrollment list, but the student and course should still exist in their own tables. You can also verify in Retool that the row is gone.

### Challenge 2: Student Schedule View

Create a new form that lets you pick a **student** from a dropdown and see all the courses they are taking, along with the total number of credits. You will need a JOIN similar to Cell 8 from Part 3, plus a `SUM(courses.credits)` query.

> **Check:** If Alice is enrolled in BMIS 490 (3 credits) and BMIS 340 (3 credits), you should see both courses listed and "Total credits: 6" at the bottom.

### Challenge 3: Enrollment Count per Course

Write a query that shows how many students are enrolled in each course. Use `GROUP BY` and `COUNT(*)`. Display it in an output widget. The output should look something like:

```
BMIS 490 - Systems Analysis:    3 students
BMIS 340 - Database Management: 2 students
ACCT 201 - Intro to Accounting: 1 student
MKTG 310 - Marketing Research:  1 student
```

**Hint:** You will need to JOIN `enrollments` with `courses` and then GROUP BY the course name.

### Challenge 4: Find the Bug

This function is supposed to find students who are NOT enrolled in any course. It has a bug. Find it and fix it.

```python
def find_unenrolled():
    cursor.execute("""
        SELECT students.name
        FROM students
        LEFT JOIN enrollments ON students.id = enrollments.course_id
        WHERE enrollments.id IS NULL
    """)
    rows = cursor.fetchall()
    if rows:
        print("Students not enrolled in any course:")
        for (name,) in rows:
            print(f"  {name}")
    else:
        print("All students are enrolled in at least one course.")

find_unenrolled()
```

**Hint:** Look carefully at which column the LEFT JOIN is matching against. The enrollments table has two foreign keys — `student_id` and `course_id`. Which one should we use to find a student's enrollments?

---

## What You Should Have When You're Done

Your notebook should contain:

1. A connection to your **Retool PostgreSQL database** (via Colab Secrets — no hardcoded URLs!)
2. Three tables: `students`, `courses`, and `enrollments`
3. A student form that can add students
4. A course form that can add courses
5. An enrollment form that can enroll students in courses (writing to the junction table)
6. A course roster viewer that uses JOIN to display enrollment data
7. At least two of the four challenges completed
8. Several students, courses, and enrollments added to demonstrate everything works

Verify your data is in Retool, then take screenshots showing your enrollment manager and course roster viewer in action, and submit them.

---

## What Just Happened — The Big Picture

| Lab | What You Built | What It Teaches |
|-----|---------------|-----------------|
| Lab 1 | A form with widgets | How users interact with an application (the **frontend**) |
| Lab 2 | API calls to fill form options | How to get data from an external source |
| Lab 3 | A SQLite database | How data is stored, structured, and queried (the **data layer**) |
| Lab 3.5 | A cloud PostgreSQL database | How databases persist independently of your app |
| Lab 4 | Forms connected to a cloud multi-table database | **Many-to-many relationships** and real cloud persistence |

You now understand the three-table pattern that powers most real applications. When you use Zagweb, Canvas, or Amazon, the same architecture is at work behind the scenes — entities in separate tables, connected by junction tables, brought together with JOINs. And your data lives in the cloud, just like theirs does.

```
What you built today:              What a real app looks like:

  Form ←→ Cloud DB                   Form ←→ API ←→ Cloud DB
  (Colab)  (Retool)                 (browser)  (server)  (cloud)
```

Next, we'll put an **API** in between the form and the database, just like a real web application.

---

## Quick Reference

### Connecting to Your Database

```python
import psycopg2
from google.colab import userdata

db_url = userdata.get('DB_URL')   # From Colab Secrets — never hardcode this!
conn = psycopg2.connect(db_url)
conn.autocommit = True
cursor = conn.cursor()
```

### SQLite vs PostgreSQL Cheat Sheet

| What | SQLite | PostgreSQL |
|------|--------|-----------|
| Placeholder | `?` | `%s` |
| Auto-increment ID | `INTEGER PRIMARY KEY AUTOINCREMENT` | `SERIAL PRIMARY KEY` |
| Text type | `TEXT` | `VARCHAR(100)` |
| Decimal type | `REAL` | `NUMERIC(3, 2)` |
| Connection | `sqlite3.connect("file.db")` | `psycopg2.connect(url)` |
| Data location | Local file (lost on restart) | Cloud server (persists forever) |

### Junction Table Pattern

```python
# Create the junction table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS enrollments (
        id SERIAL PRIMARY KEY,
        student_id INTEGER NOT NULL REFERENCES students(id),
        course_id INTEGER NOT NULL REFERENCES courses(id)
    )
""")

# Add a relationship
cursor.execute("INSERT INTO enrollments (student_id, course_id) VALUES (%s, %s)", (1, 3))

# Query with JOIN to see the relationship in human-readable form
cursor.execute("""
    SELECT students.name, courses.course_name
    FROM enrollments
    JOIN students ON enrollments.student_id = students.id
    JOIN courses ON enrollments.course_id = courses.id
""")
```

### SQL ↔ Application Mapping

| What the user does | SQL command | Tables involved |
|-------------------|-------------|-----------------|
| Adds a student | `INSERT INTO students` | students |
| Adds a course | `INSERT INTO courses` | courses |
| Enrolls a student | `INSERT INTO enrollments` | enrollments (junction) |
| Views a course roster | `SELECT ... JOIN ... JOIN` | All three |
| Drops a course | `DELETE FROM enrollments` | enrollments (junction) |
| Counts students per course | `SELECT ... JOIN ... GROUP BY` | enrollments + courses |
