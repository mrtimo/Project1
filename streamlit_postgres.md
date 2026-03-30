# Lab: Streamlit App with PostgreSQL Database

---

## Introduction

### What is Streamlit?

Streamlit is an open-source Python framework that lets you build interactive web applications with just Python — no HTML, CSS, or JavaScript required. You write a `.py` file, and Streamlit turns it into a live web app with widgets like buttons, text inputs, dropdowns, and data tables.

### How Streamlit Apps Are Structured

A Streamlit app has an **entry point** file (usually `app.py` or `streamlit_app.py`) that serves as your home page. To add more pages, you create a `pages/` folder and put additional `.py` files inside it. Each file in the `pages/` folder becomes its own page in the sidebar navigation — no routing code needed.

```
my-streamlit-app/
├── streamlit_app.py          ← Home page (entry point)
├── pages/
│   ├── 1_Add_Student.py      ← Page 1
│   ├── 2_Add_Course.py       ← Page 2
│   └── 3_Enroll_Student.py   ← Page 3
└── requirements.txt
```

> **Tip:** Prefixing filenames with numbers (e.g., `1_`, `2_`) controls the order they appear in the sidebar. Underscores in filenames are displayed as spaces.

### How Deployment Works

Streamlit offers a free **Community Cloud** hosting service. You push your app code to a **GitHub repository**, then connect that repo to Streamlit Community Cloud. Every time you push changes to GitHub, your app automatically redeploys. It's that simple.

### How Secrets Work

Your app will need a database connection URL, and you should **never** put passwords or connection strings directly in your code. Streamlit handles this with **Secrets Management**:

- On **Streamlit Community Cloud**, you enter secrets in the app's settings dashboard (Settings → Secrets). They are stored encrypted and injected at runtime.
- In your Python code, you access them with `st.secrets["SECRET_NAME"]`.
- On **Google Colab** (where we'll set up our database), you'll paste your connection URL into a variable manually — but never commit it to GitHub.

---

## Requirements

Before starting this lab, make sure you have the following:

| Requirement | Details |
|---|---|
| **A computer with Python installed** | Python 3.9 or newer. Verify with `python --version` in your terminal. |
| **VS Code (or any code editor)** | We recommend VS Code with the Python extension installed. |
| **A new, empty project folder** | Create a folder on your computer called `streamlit-student-app`. |
| **A GitHub account** | Go to [github.com](https://github.com) and **log in before starting this lab**. |
| **A Streamlit Community Cloud account** | Go to [share.streamlit.io](https://share.streamlit.io) and sign in with your GitHub account. |
| **A PostgreSQL connection URL** | Your instructor will provide this. It looks like: `postgresql://user:password@host:port/dbname` |
| **Access to Google Colab** | Go to [colab.research.google.com](https://colab.research.google.com). We'll use this to set up database tables. |

---

## Part 1: Create the Database Tables (Google Colab)

We'll start by creating our database tables using Google Colab. This is a one-time setup step.

Our data model has three tables:

- **`students10`** — stores student names and emails
- **`courses10`** — stores course names
- **`student_courses10`** — a linking table that connects students to courses (many-to-many relationship)

> **Why the `10` suffix?** The database may be shared, and other students might already have tables called `students` or `courses`. The suffix avoids conflicts.

### Steps

1. Open [Google Colab](https://colab.research.google.com) and create a new notebook.

2. **Cell 1** — Install the database driver and set your connection URL:

```python
!pip install psycopg2-binary

# ⚠️ PASTE YOUR CONNECTION URL BELOW — DO NOT SHARE THIS NOTEBOOK PUBLICLY
DB_URL = "postgresql://your_user:your_password@your_host:5432/your_db"
```

> **Important:** Replace the placeholder above with your actual PostgreSQL connection URL. This is the same connection URL you used in a previous assignment — if you still have it saved, use that. If not, your instructor can provide it again. Treat it like a password — never share it publicly.

3. **Cell 2** — Create the tables:

```python
import psycopg2

conn = psycopg2.connect(DB_URL)
cur = conn.cursor()

cur.execute("""
    CREATE TABLE IF NOT EXISTS students10 (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(100) NOT NULL UNIQUE
    );
""")

cur.execute("""
    CREATE TABLE IF NOT EXISTS courses10 (
        id SERIAL PRIMARY KEY,
        course_name VARCHAR(100) NOT NULL UNIQUE
    );
""")

cur.execute("""
    CREATE TABLE IF NOT EXISTS student_courses10 (
        id SERIAL PRIMARY KEY,
        student_id INTEGER REFERENCES students10(id) ON DELETE CASCADE,
        course_id INTEGER REFERENCES courses10(id) ON DELETE CASCADE,
        enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(student_id, course_id)
    );
""")

conn.commit()
cur.close()
conn.close()

print("✅ All three tables created successfully!")
```

4. Run both cells. You should see the success message. If you get a connection error, double-check your `DB_URL` string.

---

## Part 2: Build the Streamlit App (First Two Files)

Now let's build the app locally using VS Code.

### Step 1: Set up your project in VS Code

1. Create a new empty folder on your computer called `streamlit-student-app`.
2. Open VS Code, then go to **File → Open Folder...** and select the `streamlit-student-app` folder you just created.
3. In the VS Code **Explorer** panel (left sidebar), hover over the project name and click the **New Folder** icon (📁). Name the folder `pages`.

### Step 2: Create `requirements.txt`

In the VS Code Explorer panel, hover over the project name and click the **New File** icon (📄). Name the file `requirements.txt` and add this content:

```
streamlit
psycopg2-binary
```

### Step 3: Create the entry point — `streamlit_app.py`

In the VS Code Explorer panel, click the **New File** icon again (make sure you're creating it at the root level, not inside `pages/`). Name it `streamlit_app.py` and paste in the following code:

```python
import streamlit as st
import psycopg2

st.set_page_config(page_title="Student Enrollment App", page_icon="🎓")

def get_connection():
    return psycopg2.connect(st.secrets["DB_URL"])

st.title("🎓 Student Enrollment App")
st.write("Welcome! Use the sidebar to navigate between pages.")

st.markdown("---")
st.subheader("📊 Current Data")

try:
    conn = get_connection()
    cur = conn.cursor()

    cur.execute("SELECT COUNT(*) FROM students10;")
    student_count = cur.fetchone()[0]

    cur.execute("SELECT COUNT(*) FROM courses10;")
    course_count = cur.fetchone()[0]

    cur.execute("SELECT COUNT(*) FROM student_courses10;")
    enrollment_count = cur.fetchone()[0]

    col1, col2, col3 = st.columns(3)
    col1.metric("Students", student_count)
    col2.metric("Courses", course_count)
    col3.metric("Enrollments", enrollment_count)

    st.markdown("---")
    st.subheader("📋 All Enrollments")
    cur.execute("""
        SELECT s.name, s.email, c.course_name, sc.enrolled_at
        FROM student_courses10 sc
        JOIN students10 s ON sc.student_id = s.id
        JOIN courses10 c ON sc.course_id = c.id
        ORDER BY sc.enrolled_at DESC;
    """)
    rows = cur.fetchall()

    if rows:
        st.table(
            [{"Student": r[0], "Email": r[1], "Course": r[2], "Enrolled": r[3].strftime("%Y-%m-%d %H:%M")} for r in rows]
        )
    else:
        st.info("No enrollments yet. Add some students and courses, then enroll them!")

    cur.close()
    conn.close()

except Exception as e:
    st.error(f"Database connection error: {e}")
```

### Step 4: Create the first form — `pages/1_Add_Student.py`

In the VS Code Explorer panel, click on the `pages` folder to select it, then click the **New File** icon. Name the file `1_Add_Student.py` (it will be created inside `pages/`). Paste in the following code:

```python
import streamlit as st
import psycopg2

st.set_page_config(page_title="Add Student", page_icon="👤")

def get_connection():
    return psycopg2.connect(st.secrets["DB_URL"])

st.title("👤 Add a New Student")

with st.form("add_student_form"):
    name = st.text_input("Student Name")
    email = st.text_input("Student Email")
    submitted = st.form_submit_button("Add Student")

    if submitted:
        if name and email:
            try:
                conn = get_connection()
                cur = conn.cursor()
                cur.execute(
                    "INSERT INTO students10 (name, email) VALUES (%s, %s);",
                    (name, email)
                )
                conn.commit()
                cur.close()
                conn.close()
                st.success(f"✅ Student '{name}' added successfully!")
            except psycopg2.errors.UniqueViolation:
                st.error("⚠️ A student with that email already exists.")
            except Exception as e:
                st.error(f"Error: {e}")
        else:
            st.warning("Please fill in both fields.")

st.markdown("---")
st.subheader("Current Students")

try:
    conn = get_connection()
    cur = conn.cursor()
    cur.execute("SELECT id, name, email FROM students10 ORDER BY name;")
    students = cur.fetchall()
    cur.close()
    conn.close()

    if students:
        st.table([{"ID": s[0], "Name": s[1], "Email": s[2]} for s in students])
    else:
        st.info("No students yet.")
except Exception as e:
    st.error(f"Error: {e}")
```

Your folder should now look like this:

```
streamlit-student-app/
├── streamlit_app.py
├── requirements.txt
└── pages/
    └── 1_Add_Student.py
```

---

## Part 2.5 (Optional): Test the App Locally

Before deploying, you can try running the app on your own machine to see it in action. **This step is optional** — if you run into any issues (Python version problems, install errors, etc.), don't worry about it. Just skip ahead to Part 3. You'll be able to test everything once it's deployed to Streamlit Community Cloud.

### Step 1: Install the dependencies

Open a terminal in VS Code (**Terminal → New Terminal**) and run:

```bash
pip install streamlit psycopg2-binary
```

> **Note:** On some machines you may need to use `pip3` instead of `pip`.

### Step 2: Create a local secrets file

Streamlit needs a special file to find your secrets when running locally. In your terminal, run the following commands to create it:

**On Mac/Linux:**
```bash
mkdir -p .streamlit
echo 'DB_URL = "postgresql://your_user:your_password@your_host:5432/your_db"' > .streamlit/secrets.toml
```

**On Windows (PowerShell):**
```powershell
mkdir .streamlit
echo 'DB_URL = "postgresql://your_user:your_password@your_host:5432/your_db"' > .streamlit/secrets.toml
```

Replace the placeholder with your actual connection URL (the same one you used in Colab).

> **Important:** This `.streamlit/secrets.toml` file contains your password. **Do not upload this file to GitHub.** It's only for local testing.

### Step 3: Run the app

In your terminal, run:

```bash
streamlit run streamlit_app.py
```

Streamlit should open a new browser tab at `http://localhost:8501` showing your app. Try clicking the **Add Student** page in the sidebar and adding a student to verify the database connection works.

Press **Ctrl+C** in the terminal to stop the app when you're done.

> **If anything didn't work** — that's totally fine! Different machines have different Python configurations, and local setup can be finicky. The important thing is that your code files are ready. Move on to Part 3 and you'll see it all working once it's deployed to Streamlit Community Cloud.

---

## Part 3: Push to GitHub

Follow these steps carefully to get your code into a GitHub repository.

### Step 1: Create a new GitHub repository

1. Go to [github.com](https://github.com) (make sure you're logged in).
2. Click the **`+`** button in the top-right corner, then click **New repository**.
3. Fill in:
   - **Repository name:** `streamlit-student-app`
   - **Description:** (optional) `Student enrollment app built with Streamlit and PostgreSQL`
   - **Visibility:** **Public** (Streamlit Community Cloud requires public repos on the free tier)
   - **Check** ✅ "Add a README file" (this initializes the repo so you can upload files right away).
4. Click **Create repository**.

### Step 2: Upload your files through the browser

1. You should now be on your new repository's page. Click the **"Add file"** dropdown button, then select **"Upload files"**.
2. Open your `streamlit-student-app` folder on your computer in a file explorer window.
3. **Drag and drop** the following files and folder from your computer into the upload area on GitHub:
   - `streamlit_app.py`
   - `requirements.txt`
   - `pages/` folder (drag the entire folder — it will include `1_Add_Student.py` inside it)
4. Scroll down to the bottom of the page. You'll see a "Commit changes" section.
5. You can leave the default commit message or type something like `Initial commit - student enrollment app`.
6. Click the green **"Commit changes"** button.

> **⚠️ Don't forget to press Commit!** Your files are not saved to the repository until you click the Commit button. If you navigate away without committing, you'll need to upload again.

Refresh your GitHub repo page — you should see all your files listed there.

---

## Part 4: Deploy to Streamlit Community Cloud

### Step 1: Create the app

1. Go to [share.streamlit.io](https://share.streamlit.io) and sign in with GitHub.
2. Click **"Create app"** (or **"New app"**).
3. Select **"Yep, I have an app"** and then choose **"From an existing repo"**.
4. You can paste the URL that points directly to your app's entry point file in your repo. For example:
   `https://github.com/YOUR_GITHUB_USERNAME/streamlit-student-app/blob/main/streamlit_app.py`
   (For reference, a completed example looks like: `https://github.com/mrtimo/test_streamlit_1/blob/main/app.py`)
5. Click **Deploy!**

The app will start building. After a minute or so, it will attempt to load — and you'll see a **database connection error**. That's expected! We haven't added our secret yet.

### Step 2: Add your database secret

1. On your deployed app page, click the **three dots (`⋯`)** menu in the bottom-right corner.
2. Click **Settings**.
3. Click the **Secrets** tab.
4. Paste the following into the secrets text box, replacing the placeholder with your actual connection URL:

```toml
DB_URL = "postgresql://your_user:your_password@your_host:5432/your_db"
```

5. Click **Save**.
6. Your app will automatically reboot. Wait a moment, then refresh the page.

You should now see the home page with the metrics (showing 0 students, 0 courses, and 0 enrollments) and the **Add Student** page in the sidebar. Try adding a student!

> **Checkpoint:** Verify that you can add a student and see them listed on the Add Student page, and that the home page counter updates. If this works, you're ready to continue.

---

## Part 5: Add the Remaining Forms

Now let's add the **Add Course** and **Enroll Student** pages.

### File: `pages/2_Add_Course.py`

In VS Code, click on the `pages` folder, then click the **New File** icon and name it `2_Add_Course.py`. Paste in the following code:

```python
import streamlit as st
import psycopg2

st.set_page_config(page_title="Add Course", page_icon="📚")

def get_connection():
    return psycopg2.connect(st.secrets["DB_URL"])

st.title("📚 Add a New Course")

with st.form("add_course_form"):
    course_name = st.text_input("Course Name")
    submitted = st.form_submit_button("Add Course")

    if submitted:
        if course_name:
            try:
                conn = get_connection()
                cur = conn.cursor()
                cur.execute(
                    "INSERT INTO courses10 (course_name) VALUES (%s);",
                    (course_name,)
                )
                conn.commit()
                cur.close()
                conn.close()
                st.success(f"✅ Course '{course_name}' added successfully!")
            except psycopg2.errors.UniqueViolation:
                st.error("⚠️ A course with that name already exists.")
            except Exception as e:
                st.error(f"Error: {e}")
        else:
            st.warning("Please enter a course name.")

st.markdown("---")
st.subheader("Current Courses")

try:
    conn = get_connection()
    cur = conn.cursor()
    cur.execute("SELECT id, course_name FROM courses10 ORDER BY course_name;")
    courses = cur.fetchall()
    cur.close()
    conn.close()

    if courses:
        st.table([{"ID": c[0], "Course Name": c[1]} for c in courses])
    else:
        st.info("No courses yet.")
except Exception as e:
    st.error(f"Error: {e}")
```

### File: `pages/3_Enroll_Student.py`

Again in VS Code, click on the `pages` folder, click **New File**, and name it `3_Enroll_Student.py`. Paste in the following code:

```python
import streamlit as st
import psycopg2

st.set_page_config(page_title="Enroll Student", page_icon="📝")

def get_connection():
    return psycopg2.connect(st.secrets["DB_URL"])

st.title("📝 Enroll a Student in a Course")

try:
    conn = get_connection()
    cur = conn.cursor()

    cur.execute("SELECT id, name FROM students10 ORDER BY name;")
    students = cur.fetchall()

    cur.execute("SELECT id, course_name FROM courses10 ORDER BY course_name;")
    courses = cur.fetchall()

    cur.close()
    conn.close()

    if not students:
        st.warning("No students found. Please add a student first.")
    elif not courses:
        st.warning("No courses found. Please add a course first.")
    else:
        student_options = {s[1]: s[0] for s in students}
        course_options = {c[1]: c[0] for c in courses}

        with st.form("enroll_form"):
            selected_student = st.selectbox("Select Student", options=student_options.keys())
            selected_course = st.selectbox("Select Course", options=course_options.keys())
            submitted = st.form_submit_button("Enroll")

            if submitted:
                student_id = student_options[selected_student]
                course_id = course_options[selected_course]
                try:
                    conn = get_connection()
                    cur = conn.cursor()
                    cur.execute(
                        "INSERT INTO student_courses10 (student_id, course_id) VALUES (%s, %s);",
                        (student_id, course_id)
                    )
                    conn.commit()
                    cur.close()
                    conn.close()
                    st.success(f"✅ '{selected_student}' enrolled in '{selected_course}'!")
                except psycopg2.errors.UniqueViolation:
                    st.error("⚠️ This student is already enrolled in that course.")
                except Exception as e:
                    st.error(f"Error: {e}")

except Exception as e:
    st.error(f"Error: {e}")
```

### Upload the new files to GitHub

1. Go to your repository on GitHub.
2. Click the **"Add file"** dropdown, then select **"Upload files"**.
3. Drag and drop the two new files from your `pages/` folder: `2_Add_Course.py` and `3_Enroll_Student.py`.
4. In the commit message, type something like `Add course and enrollment pages`.
5. Click the green **"Commit changes"** button.

> **⚠️ Remember:** Always press **Commit changes** after uploading — your files aren't saved until you do!

Your Streamlit app will automatically redeploy within a minute. Refresh your app and verify that all three pages appear in the sidebar and work correctly.

> **Checkpoint:** Add a course, then enroll a student in that course. Go back to the home page and confirm the enrollment appears in the table and the metrics have updated.

---

## Part 6: Mini Challenge — Input Validation

Your app currently accepts any text as an email address. That's not great! Let's add a simple email validation to the **Add Student** form.

### The Challenge

Modify `pages/1_Add_Student.py` so that:

- The email field is validated to check that it contains an `@` symbol and at least one `.` after the `@`.
- If the email is invalid, show a warning and **do not** insert it into the database.

Take a few minutes to try it yourself before looking at the answer below.

---

### The Answer

In `pages/1_Add_Student.py`, find this block inside the `if submitted:` section:

```python
    if submitted:
        if name and email:
```

Replace it with:

```python
    if submitted:
        # Simple email validation
        import re
        email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        is_valid_email = re.match(email_pattern, email)

        if not name or not email:
            st.warning("Please fill in both fields.")
        elif not is_valid_email:
            st.warning("⚠️ Please enter a valid email address (e.g., student@example.com).")
        else:
```

Make sure the `try:` block that follows is indented one level deeper under the new `else:`. The full updated section looks like this:

```python
    if submitted:
        import re
        email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        is_valid_email = re.match(email_pattern, email)

        if not name or not email:
            st.warning("Please fill in both fields.")
        elif not is_valid_email:
            st.warning("⚠️ Please enter a valid email address (e.g., student@example.com).")
        else:
            try:
                conn = get_connection()
                cur = conn.cursor()
                cur.execute(
                    "INSERT INTO students10 (name, email) VALUES (%s, %s);",
                    (name, email)
                )
                conn.commit()
                cur.close()
                conn.close()
                st.success(f"✅ Student '{name}' added successfully!")
            except psycopg2.errors.UniqueViolation:
                st.error("⚠️ A student with that email already exists.")
            except Exception as e:
                st.error(f"Error: {e}")
```

Upload your updated `1_Add_Student.py` to GitHub:

1. Go to your repository on GitHub.
2. Navigate into the `pages/` folder by clicking on it.
3. Click on `1_Add_Student.py` to open it, then click the **pencil icon** (✏️) to edit the file directly in the browser.
4. Make the changes shown above, then scroll down and click **"Commit changes"**.

Alternatively, you can use the **"Add file" → "Upload files"** method to re-upload the updated file — GitHub will overwrite the existing one.

Verify on your live app that entering something like `notanemail` now shows the warning, while `student@school.edu` works normally.

---

## Congratulations! 🎉

You just built and deployed a full-stack web application with a database — entirely in Python. Take a moment to appreciate what you've accomplished:

- You **designed a relational database schema** with a many-to-many relationship.
- You **built a multi-page web app** with forms, live data, and input validation.
- You **deployed it to the internet** with automatic updates from Git.
- You **managed secrets securely**, keeping your credentials out of your codebase.

### Where to Go from Here

This lab gave you a working prototype. If you want to take your skills further, here are some directions to explore:

- **Authentication and Users** — Streamlit has community packages like `streamlit-authenticator` that let you add login pages with usernames and passwords, so different users can have different access levels.
- **Private Deployment** — Streamlit Community Cloud requires public repos. If you want to keep your code private, look into deploying on platforms like **Railway**, **Render**, **Fly.io**, or **AWS App Runner**. These services let you deploy from private repos and often have generous free tiers.
- **More Robust Frameworks** — When you're ready for a production-grade tool, explore frameworks like **FastAPI** (for building APIs), **Django** (a full-featured web framework with built-in user management), or **Flask**. These give you more control over routing, middleware, and security.
- **Connection Pooling** — Our app opens a new database connection on every interaction. In a real production app, you'd use a connection pool (like `psycopg2.pool` or `SQLAlchemy`) to reuse connections efficiently.

The most important thing is that you now understand the fundamentals: **forms, databases, deployment, and secrets**. Every web application — from a simple internal tool to a massive SaaS product — is built on these same building blocks. Keep building! 🚀
