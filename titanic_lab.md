# Lab: Introduction to Charts with Matplotlib

## Overview

In this lab you will use the Titanic dataset to build bar charts and line charts step by step using Python and Matplotlib. Each section contains a code cell you can paste directly into Google Colab.

By the end of this lab you will be able to:

- Load a CSV into a pandas DataFrame
- Subset and aggregate data for plotting
- Create and customize bar charts
- Create and customize line charts

---

## Part 1: Setting Up the Data

### Step 1 — Load the Dataset

Paste this into your first code cell. It reads the Titanic CSV directly from a URL and displays the first five rows so you can see what you're working with.

```python
import pandas as pd

url = "https://raw.githubusercontent.com/datasciencedojo/datasets/master/titanic.csv"
df = pd.read_csv(url)

df.head()
```

You should see columns like `PassengerId`, `Survived`, `Pclass`, `Name`, `Sex`, `Age`, and more. Take a minute to scroll through and get familiar with the data.

### Step 2 — Explore the Data

Run this cell to see how many rows and columns you have and what types they are.

```python
print(df.shape)
print()
print(df.dtypes)
```

---

## Part 2: Bar Charts

### Step 3 — Subset the Data

Before we chart anything we need to prepare our data. Let's count how many passengers were in each ticket class (1st, 2nd, 3rd).

```python
class_counts = df['Pclass'].value_counts().sort_index()

print(class_counts)
```

`value_counts()` counts how many times each value appears. `sort_index()` puts them in order (1, 2, 3).

### Step 4 — Create a Basic Bar Chart

Now let's plot it. This is the simplest possible bar chart.

```python
import matplotlib.pyplot as plt

plt.bar(class_counts.index, class_counts.values)
plt.show()
```

You should see three bars. It works, but it's not very informative yet. Let's fix that.

### Step 5 — Add Labels and a Title

A chart isn't useful without labels. Let's add an x-axis label, a y-axis label, and a title.

```python
plt.bar(class_counts.index, class_counts.values)

plt.xlabel("Passenger Class")
plt.ylabel("Number of Passengers")
plt.title("Titanic Passengers by Class")

plt.show()
```

### Step 6 — Add Custom Tick Labels

The x-axis currently shows 1, 2, 3. Let's replace those with more descriptive labels.

```python
plt.bar(class_counts.index, class_counts.values)

plt.xticks([1, 2, 3], ["1st Class", "2nd Class", "3rd Class"])
plt.xlabel("Passenger Class")
plt.ylabel("Number of Passengers")
plt.title("Titanic Passengers by Class")

plt.show()
```

### Step 7 — Add Color and Edge Lines

Let's make it look a bit more polished with a custom color and visible edges on each bar.

```python
plt.bar(class_counts.index, class_counts.values,
        color="steelblue", edgecolor="black")

plt.xticks([1, 2, 3], ["1st Class", "2nd Class", "3rd Class"])
plt.xlabel("Passenger Class")
plt.ylabel("Number of Passengers")
plt.title("Titanic Passengers by Class")

plt.show()
```

### Step 8 — Add Value Labels on Top of Each Bar

Sometimes you want the exact number displayed on the chart. This loop places a text label above each bar.

```python
bars = plt.bar(class_counts.index, class_counts.values,
               color="steelblue", edgecolor="black")

for bar in bars:
    height = bar.get_height()
    plt.text(bar.get_x() + bar.get_width() / 2, height + 5,
             str(int(height)), ha="center")

plt.xticks([1, 2, 3], ["1st Class", "2nd Class", "3rd Class"])
plt.xlabel("Passenger Class")
plt.ylabel("Number of Passengers")
plt.title("Titanic Passengers by Class")

plt.show()
```

### Step 9 — Adjust the Figure Size

If your chart feels cramped you can control its size with `figsize`.

```python
plt.figure(figsize=(8, 5))

bars = plt.bar(class_counts.index, class_counts.values,
               color="steelblue", edgecolor="black")

for bar in bars:
    height = bar.get_height()
    plt.text(bar.get_x() + bar.get_width() / 2, height + 5,
             str(int(height)), ha="center")

plt.xticks([1, 2, 3], ["1st Class", "2nd Class", "3rd Class"])
plt.xlabel("Passenger Class")
plt.ylabel("Number of Passengers")
plt.title("Titanic Passengers by Class")
plt.tight_layout()

plt.show()
```

---

## Part 3: Line Charts

Line charts are great for showing trends across ordered categories or continuous data. We'll look at survival rates by age group.

### Step 10 — Create Age Groups

The `Age` column has individual ages. Let's bin them into groups so we can see trends more clearly.

```python
df_age = df.dropna(subset=["Age"]).copy()

bins = [0, 10, 20, 30, 40, 50, 60, 80]
labels = ["0-10", "11-20", "21-30", "31-40", "41-50", "51-60", "61-80"]

df_age["AgeGroup"] = pd.cut(df_age["Age"], bins=bins, labels=labels)

print(df_age["AgeGroup"].value_counts().sort_index())
```

`pd.cut()` places each age into one of the bins we defined. We also drop rows with missing ages using `dropna()`.

### Step 11 — Calculate Survival Rate by Age Group

Now let's compute the average survival rate for each group. Since `Survived` is 0 or 1, the mean gives us the proportion who survived.

```python
survival_by_age = df_age.groupby("AgeGroup")["Survived"].mean()

print(survival_by_age)
```

### Step 12 — Create a Basic Line Chart

```python
plt.plot(survival_by_age.index, survival_by_age.values)
plt.show()
```

Simple, but let's improve it.

### Step 13 — Add Labels and a Title

```python
plt.plot(survival_by_age.index, survival_by_age.values)

plt.xlabel("Age Group")
plt.ylabel("Survival Rate")
plt.title("Titanic Survival Rate by Age Group")

plt.show()
```

### Step 14 — Add Markers and Change the Line Style

Markers make individual data points easier to read.

```python
plt.plot(survival_by_age.index, survival_by_age.values,
         marker="o", linestyle="-", color="coral")

plt.xlabel("Age Group")
plt.ylabel("Survival Rate")
plt.title("Titanic Survival Rate by Age Group")

plt.show()
```

Try changing `marker` to `"s"` (square) or `"^"` (triangle). Try `linestyle="--"` for dashed.

### Step 15 — Add a Grid and Adjust Size

Grids help the reader trace values across the chart.

```python
plt.figure(figsize=(8, 5))

plt.plot(survival_by_age.index, survival_by_age.values,
         marker="o", linestyle="-", color="coral", linewidth=2)

plt.xlabel("Age Group")
plt.ylabel("Survival Rate")
plt.title("Titanic Survival Rate by Age Group")
plt.grid(True, linestyle="--", alpha=0.5)
plt.tight_layout()

plt.show()
```

### Step 16 — Plot Multiple Lines

Let's compare survival rates for male and female passengers on the same chart.

```python
plt.figure(figsize=(8, 5))

for sex, color in [("male", "steelblue"), ("female", "coral")]:
    subset = df_age[df_age["Sex"] == sex]
    survival = subset.groupby("AgeGroup")["Survived"].mean()
    plt.plot(survival.index, survival.values,
             marker="o", linestyle="-", color=color, linewidth=2, label=sex.capitalize())

plt.xlabel("Age Group")
plt.ylabel("Survival Rate")
plt.title("Survival Rate by Age Group and Sex")
plt.legend()
plt.grid(True, linestyle="--", alpha=0.5)
plt.tight_layout()

plt.show()
```

`plt.legend()` uses the `label` we passed into each `plt.plot()` call.

---

## Part 4: Challenges

Try these on your own. All the techniques you need were covered above.

**Challenge 1 — Bar Chart: Survival Count by Sex**
Create a bar chart that shows the number of passengers who survived grouped by sex. Hint: filter the DataFrame to `Survived == 1` first, then use `value_counts()` on the `Sex` column.

**Challenge 2 — Change the Colors**
Take any chart from this lab and change its colors. Try `"mediumseagreen"`, `"tomato"`, `"goldenrod"`, or look up the full list of [Matplotlib named colors](https://matplotlib.org/stable/gallery/color/named_colors.html).

**Challenge 3 — Horizontal Bar Chart**
Recreate the passengers-by-class bar chart as a horizontal bar chart. Hint: use `plt.barh()` instead of `plt.bar()`, and swap your x/y labels.

**Challenge 4 — Line Chart: Average Fare by Class**
Create a line chart showing the average fare paid by each passenger class. Hint: use `groupby("Pclass")["Fare"].mean()`.

**Challenge 5 — Add Your Own Touches**
Pick any chart from this lab and customize it further. Some ideas:

- Change the font size of the title with `fontsize=16` inside `plt.title()`
- Rotate x-axis labels with `plt.xticks(rotation=45)`
- Change the background style with `plt.style.use("ggplot")` at the top of your cell
- Add a subtitle using `plt.suptitle()` and `plt.title()` together

---

## Quick Reference

| What you want to do | Code |
|---|---|
| Bar chart | `plt.bar(x, y)` |
| Horizontal bar chart | `plt.barh(x, y)` |
| Line chart | `plt.plot(x, y)` |
| Add title | `plt.title("Title")` |
| X-axis label | `plt.xlabel("Label")` |
| Y-axis label | `plt.ylabel("Label")` |
| Custom x-tick labels | `plt.xticks([1, 2], ["A", "B"])` |
| Legend | `plt.legend()` |
| Grid | `plt.grid(True)` |
| Figure size | `plt.figure(figsize=(8, 5))` |
| Tight layout | `plt.tight_layout()` |
