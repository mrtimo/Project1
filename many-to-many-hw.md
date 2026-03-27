# Homework 4: Build Your Own Many-to-Many Application

## Assignment

Build a Google Colab notebook that connects to your **Retool PostgreSQL database** and manages a **many-to-many relationship** of your choosing using **ipywidgets forms**.

You pick the topic. The only requirement is that it involves two entities that have a many-to-many relationship, connected by a junction table. Here are some ideas to get you started — but feel free to come up with your own:

- **Recipes ↔ Ingredients** — A recipe uses many ingredients. An ingredient appears in many recipes.
- **Athletes ↔ Teams** — An athlete can play for many teams (over time). A team has many athletes.
- **Movies ↔ Actors** — A movie has many actors. An actor appears in many movies.
- **Songs ↔ Playlists** — A song can be on many playlists. A playlist has many songs.
- **Employees ↔ Projects** — An employee works on many projects. A project has many employees.
- **Video Games ↔ Platforms** — A game is available on many platforms. A platform has many games.
- **Books ↔ Authors** — A book can have many authors. An author can write many books.

---

## Requirements

Your notebook must include:

### 1. Three Tables
Create three tables in your **Retool PostgreSQL database**: two entity tables and one junction (linking) table. Each entity table should have at least 3 columns (including the id). The junction table should have foreign keys pointing to both entity tables.

### 2. Connection via Colab Secrets
Your database connection URL must be stored in **Colab Secrets** and accessed using `userdata.get()`. It should **not** appear anywhere in your code as plain text.

### 3. Forms to Add Items
Build ipywidgets forms that let you add new items to **each** of your two entity tables. Each form should have appropriate input widgets (text fields, dropdowns, sliders, etc.) and an "Add" button that INSERTs into the database.

### 4. A Form to Link Items
Build an ipywidgets form that lets you create a relationship between the two entities — i.e., INSERT a row into your junction table. This form should use **dropdowns** that display the names of existing items (not raw ID numbers).

### 5. Buttons to View Each Table
Include buttons that display the current contents of each of your three tables. The junction table view should use a **JOIN** so it displays human-readable names, not just ID numbers.

### 6. Starter Data
Pre-load at least **3 items** in each entity table and at least **5 rows** in the junction table so the grader can see the application working immediately.

---

## What to Submit

- Submit a word document / PDF with the following:
- Submit your **Google Colab notebook link** (.ipynb) with all cells run and output visible. 
- Screenshots of your forms. 
- Screenshots of your 3 tables in retool. 

The grader should be able to see your forms, your starter data, and evidence that you added at least one new item and one new relationship using the forms.

---

## Grading

| Criteria | Points |
|----------|--------|
| **Many-to-many relationship works** — Three tables exist with proper foreign keys in the junction table, and a JOIN query correctly displays linked data with readable names. | 30 |
| **Can add new items** — Forms with ipywidgets allow adding new records to both entity tables and to the junction table, and the data actually appears in the database. | 30 |
| **Uses Retool PostgreSQL database** — The notebook connects to your Retool cloud database (not a local SQLite file), so data persists across Colab sessions. | 20 |
| **Connection URL is stored securely** — The database URL is accessed via `userdata.get()` from Colab Secrets. It does **not** appear as a hard-coded string anywhere in the notebook. | 20 |
| **Total** | **100** |
