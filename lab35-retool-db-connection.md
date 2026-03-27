# Lab 3.5: Set Up a PostgreSQL Database

## Background: What is PostgreSQL?

**PostgreSQL** (often just called "Postgres") is a powerful, open-source relational database system. It has been actively developed for over 35 years and has earned a strong reputation for reliability, feature richness, and performance.

### How is PostgreSQL Different from SQLite?

In our previous labs, we used **SQLite** — a lightweight database that stores everything in a single file. SQLite is fantastic for learning and prototyping, but it has some important limitations:

- **SQLite runs inside your application.** There is no separate database server. When your app stops (or when your Colab runtime resets), the database file can be lost.
- **SQLite is single-user.** It is not designed for multiple applications or users to connect to it at the same time.
- **SQLite has limited data types and features** compared to full-scale database systems.

**PostgreSQL**, on the other hand:

- **Runs as a separate server.** Your application connects to it over the network. The database persists independently of your application — if Colab restarts, your data is still there.
- **Supports multiple concurrent connections.** Many users and applications can read and write at the same time.
- **Offers advanced features** like complex queries, JSON support, full-text search, and robust security controls.

### Why PostgreSQL?

PostgreSQL is one of the most popular databases in the world. It is **completely free and open source**, maintained by a global community of developers. It is used by companies of all sizes — from startups to organizations like Apple, Instagram, Spotify, and the U.S. Federal Aviation Administration. When you see a job listing that mentions database experience, there is a very good chance PostgreSQL is one of the expected technologies.

Learning to connect to and work with a real PostgreSQL database is an important step beyond SQLite and toward the kind of architecture you will encounter in professional settings.

---

## Lab Overview

In this lab, we will:

1. **Create a free PostgreSQL database** using [Retool](https://retool.com).
2. **Find our database connection URL** in Retool's interface.
3. **Store our connection URL securely** in Google Colab using Secrets.
4. **Connect to the database from Colab** and create a test table using Python.
5. **Verify the table appeared** back in Retool's database UI.

### Why Retool?

There are many ways to host a PostgreSQL database in the cloud. You could set one up on services like **AWS RDS**, **Google Cloud SQL**, **Supabase**, **Neon**, or **Railway**. These are all great options and you may use them in future courses or projects.

We are using **Retool** because:

- Free accounts come with a **free PostgreSQL database** — no credit card required.
- Retool provides a **nice visual interface** to browse your tables and data, which makes it easy to see what is happening in your database without writing queries.
- It is quick to set up.

The most important thing is that our data will now **persist** — it will not be lost every time our Colab runtime disconnects or restarts.

---

## Step 1: Create a Retool Account

1. Go to [https://retool.com](https://retool.com) and click **Sign Up** (or **Get Started**).
2. **Sign up using your `@zagmail.gonzaga.edu` email address.**
3. ***Do not join someone else's organization - create your own organization.*** - otherwise you will not have database access.
4. When it asks for an organization name, you can put in whatever you like (your name, "BMIS490", etc.).
5. Continue filling out the sign-up form. For any questions about your role or what you are building, just pick whatever seems reasonable — it does not matter for our purposes.
6. If you see an intro tutorial or walkthrough, you can **press "Skip"** to get past it.

---

## Step 2: Navigate to Retool Database

1. Once you are inside Retool, look at the **top-left corner** of the screen. You should see the **Retool icon** (it looks like a small logo/icon).
2. **Click on the Retool icon.** A navigation menu will appear.
3. In that menu, click on **"Retool Database"**.
4. You are now on the database page. You should see a default table (Retool may create a sample table for you).

---

## Step 3: Find Your PostgreSQL Connection URL

1. On the database page, look at the **top-left corner** again. You should see the word **"Database"** next to the Retool icon.
2. **Click on "Database"** — a dropdown or menu will appear.
3. Look for **"Connections"** in this menu and click on it.
4. You should now see your **PostgreSQL connection URL**. It will look something like this:

```
postgresql://retool:some-long-password@ep-something.retooldb.com/retool?sslmode=require
```

5. **Copy this entire connection URL.** You will need it in the next step.

> **Important:** This connection URL is essentially a password to your database. Anyone who has it can connect to your database and read or modify your data. Do not share it publicly or commit it to a public repository.

---

## Step 4: Store the Connection URL in Google Colab Secrets

We do not want to paste our connection URL directly into our code (that would be insecure). Instead, we will use Colab's **Secrets** feature.

1. Open **Google Colab** and create a new notebook (or open your existing lab notebook).
2. In the left sidebar, click the **🔑 Secrets** icon (it looks like a key).
3. Click **"Add new secret"**.
4. For the **Name**, enter: `DB_URL`
5. For the **Value**, paste your full PostgreSQL connection URL that you copied from Retool.
6. Make sure the **"Notebook access"** toggle is turned **on** for this notebook.

---

## Step 5: Connect to the Database and Create a Test Table

Now let's verify everything works. In a new code cell in your Colab notebook, run the following code:

### Install the required library

```python
!pip install psycopg2-binary
```

### Connect and create a test table

```python
import psycopg2
from google.colab import userdata

# Get the connection URL from Colab Secrets
db_url = userdata.get('DB_URL')

# Connect to the database
conn = psycopg2.connect(db_url)
cur = conn.cursor()

# Create a simple test table
cur.execute("""
    CREATE TABLE IF NOT EXISTS test_students (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100),
        major VARCHAR(100),
        gpa NUMERIC(3, 2)
    );
""")

# Insert a couple of test rows
cur.execute("""
    INSERT INTO test_students (name, major, gpa)
    VALUES
        ('Maria Santos', 'Business Analytics', 3.85),
        ('James Chen', 'Management Information Systems', 3.72),
        ('Aisha Patel', 'Accounting', 3.91);
""")

# Commit the changes
conn.commit()

# Verify by reading the data back
cur.execute("SELECT * FROM test_students;")
rows = cur.fetchall()

print("Rows in test_students:")
for row in rows:
    print(row)

# Close the connection
cur.close()
conn.close()

print("\nDone! Your test table has been created and populated.")
```

You should see output showing the three rows you just inserted.

---

## Step 6: Verify in Retool

1. Go back to **Retool Database** in your browser.
2. **Refresh the page** (this is important — Retool will not automatically show new tables).
3. You should now see your **`test_students`** table in the table list on the left side.
4. Click on the table to browse the data. You should see the three student rows you inserted from Colab.

🎉 **Congratulations!** You now have a cloud-hosted PostgreSQL database that your Colab notebooks can connect to. Your data will persist even when Colab disconnects. This is how real applications work — the database lives separately from the application code.

---

## What's Next?

In upcoming labs, we will use this database connection to store real application data instead of using SQLite. We will replace our local database calls with PostgreSQL queries, giving our application a true cloud-backed data layer.
