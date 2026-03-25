***

# Homework: The "Personal Passion" Database 
### Python + SQLite Challenge

## 🎯 Objective
In our last lab, we saved data to simple `.txt` files. While useful, text files are hard to search and organize. Today, you will use Python to create a **SQL Database** (SQLite) to store information about something **you actually care about**. 

**Examples of topics:**
*   A "Top 10 Video Games" tracker.
*   A "Sneaker Collection" inventory.
*   A "Favorite Recipes" list.
*   A "Sports Stats" tracker for your favorite team.

---

## 🛠 Instructions

### 1. Setup in Google Colab
*   Open a new Google Colab notebook.
*   At the top, create a Markdown cell with your **Name** and your **Database Topic**.
*   Import the database library using: `import sqlite3`

### 2. Coding Tasks
Your Python code must use the `sqlite3` library to perform the following:

1.  **Create a Table:** Build a table with at least 4 columns (Example: `id`, `item_name`, `rating`, `year_released`).
2.  **Add Data (INSERT):** Run **2 separate INSERT statements** to add items to your database.
3.  **Read Data (SELECT):** Run **3 different SELECT statements**. 
    *   One to show everything.
    *   Two using a `WHERE` clause to filter (e.g., "Show me all games with a rating > 9").
4.  **Update Data (UPDATE):** Run **3 different UPDATE statements** to change information already in your table.
5.  **Remove Data (DELETE):** Run **3 different DELETE statements** to remove specific rows from your table.

### 3. Submission Format
You must turn in **one PDF or .doc file** that includes:
1.  A brief description of why you chose your topic, and a reflection on what you learned. (3-5 sentences)
2.  Copy and paste you code in the .doc file
3.  A **clickable link** to your Google Colab notebook. 
    *   *Note:* Ensure your Colab sharing settings are set to **"Anyone with the link can view"** so I can grade it!

---

## 📊 Grading Rubric

| Criteria | 5 Points (Excellent) | 3 Points (Developing) | 1 Point (Beginning) |
| :--- | :--- | :--- | :--- |
| **Topic & Personalization** | Database topic is clearly personal/unique; columns make sense for the topic. | Topic is generic (e.g., "Student List") or lacks clear column structure. | Topic is missing or unclear. |
| **Python/SQL Integration** | Uses `sqlite3` and `cursor.execute()` correctly to send commands. | Commands are present but contain frequent syntax errors. | Python code does not run. |
| **Insert & Select** | Includes 2 Inserts and 3 Selects (including filtered queries). | Missing 1 or more of the required statements. | Only 1 or 2 statements total. |
| **Update & Delete** | Includes exactly 3 Updates and 3 Deletes as requested. | Missing 1 or more of the required statements. | No updates or deletes present. |
| **Submission Format** | Turned in as a PDF/Doc with a working, shared Colab link. | Link is provided but sharing permissions are locked. | No link provided. |

**Total Points: 25**

---

### 💡 Quick Code Tip for Students:
Remember the "Python-to-SQL" workflow:
1. `conn = sqlite3.connect('my_data.db')`
2. `c = conn.cursor()`
3. `c.execute("YOUR SQL COMMAND HERE")`
4. `conn.commit()` (Don't forget to commit your changes!)
5. `conn.close()`
