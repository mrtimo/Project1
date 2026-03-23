# Lab 1: Building Interactive Forms with ipywidgets in Google Colab

## Overview

In this lab you will learn how to create interactive form elements inside a Google Colab notebook using the `ipywidgets` library. By the end of this lab you will be comfortable with text inputs, sliders, dropdowns, checkboxes, buttons, and layout containers.

**Time:** ~30 minutes

---

## Part 1: Your First Widget

Open a new Google Colab notebook. In the first cell, paste the following code and run it:

```python
import ipywidgets as widgets
from IPython.display import display

greeting = widgets.Text(description='Name:', placeholder='Type your name here')
display(greeting)
```

You should see a text box with the label "Name:" appear below the cell. Type something into it.

Now in a **new cell**, paste and run this:

```python
print(f"The value in the text box is: {greeting.value}")
```

This shows that the widget **stores its value** and you can access it from other cells using `.value`.

---

## Part 2: Exploring Widget Types

Paste the following code into a **new cell**. This creates several different widget types so you can see what's available.

```python
# Text input
favorite_food = widgets.Text(description='Food:', placeholder='e.g. Pizza')

# Multi-line text
comments = widgets.Textarea(description='Comments:', placeholder='Write anything here')

# Numeric slider
rating = widgets.IntSlider(value=5, min=1, max=10, step=1, description='Rating:')

# Float slider
temperature = widgets.FloatSlider(value=72.0, min=32.0, max=120.0, step=0.5, description='Temp (F):')

# Dropdown (single selection)
color = widgets.Dropdown(
    options=['Red', 'Blue', 'Green', 'Yellow', 'Purple'],
    value='Blue',
    description='Color:'
)

# Radio buttons
size = widgets.RadioButtons(
    options=['Small', 'Medium', 'Large'],
    description='Size:'
)

# Multi-select (hold Ctrl/Cmd to pick multiple)
toppings = widgets.SelectMultiple(
    options=['Cheese', 'Pepperoni', 'Mushrooms', 'Onions', 'Peppers'],
    description='Toppings:',
    rows=5
)

# Checkbox
agree = widgets.Checkbox(value=False, description='I agree to the terms')

# Toggle button
dark_mode = widgets.ToggleButton(value=False, description='Dark Mode', icon='moon')

display(favorite_food, comments, rating, temperature, color, size, toppings, agree, dark_mode)
```

Run the cell. Play with each widget — change values, select options, slide the sliders.

---

## Part 3: Reading Widget Values with a Button

Now we'll add a **button** that reads all the widget values and prints a summary. Paste this into a **new cell**:

```python
output = widgets.Output()
submit = widgets.Button(description='Submit Order', button_style='success', icon='check')

def on_click(b):
    output.clear_output()
    with output:
        print("=== Your Order ===")
        print(f"Favorite food: {favorite_food.value}")
        print(f"Rating:        {rating.value}/10")
        print(f"Temperature:   {temperature.value}°F")
        print(f"Color:         {color.value}")
        print(f"Size:          {size.value}")
        print(f"Toppings:      {', '.join(toppings.value) or 'None'}")
        print(f"Agreed:        {'Yes' if agree.value else 'No'}")
        print(f"Dark Mode:     {'On' if dark_mode.value else 'Off'}")

submit.on_click(on_click)
display(submit, output)
```

Run it, change some form values in Part 2, then click the button. You should see a printed summary of everything you selected.

---

## Part 4: Layout with VBox and HBox

Right now the widgets are just stacked one after another. Let's organize them using `VBox` (vertical stack) and `HBox` (horizontal, side-by-side). Paste this into a **new cell**:

```python
title = widgets.HTML("<h2>🍕 Pizza Order Form</h2>")

left_column = widgets.VBox([favorite_food, color, size, rating])
right_column = widgets.VBox([
    widgets.Label("Hold Ctrl/Cmd for multiple:"),
    toppings,
    temperature
])

columns = widgets.HBox([left_column, widgets.Box(layout=widgets.Layout(width='30px')), right_column])

form = widgets.VBox([
    title,
    columns,
    comments,
    agree,
    dark_mode,
    submit,
    output
], layout=widgets.Layout(padding='15px', border='2px solid #e74c3c', border_radius='10px', width='65%'))

display(form)
```

Now you should have a nicely organized form with two columns inside a bordered box. The button and output area are at the bottom.

---

## Part 5: Your Turn — Small Modifications

Now it's time for you to make some changes. Complete **all four tasks** below in your notebook. You may need to re-run cells after making edits.

### Task 1: Add a new widget

Add a **DatePicker** widget to the form. The widget should be:

- Stored in a variable called `delivery_date`
- Have the description `'Deliver by:'`

Add it to the form layout so it appears somewhere visible (your choice where). Add a line to the button callback so the delivery date prints in the summary.

> **Hint:** The widget is `widgets.DatePicker(description='...')`

### Task 2: Change the dropdown options

Change the `color` dropdown so it contains **exactly these options** instead of the current ones:

`'Crimson'`, `'Teal'`, `'Gold'`, `'Slate'`

Set the default value to `'Teal'`.

### Task 3: Adjust the slider range

Change the `rating` IntSlider so it goes from **1 to 5** (instead of 1 to 10), with a default value of **3**.

### Task 4: Change the button style

Change the submit button so that:

- The description says `'Place Order'` instead of `'Submit Order'`
- The `button_style` is `'info'` instead of `'success'`
- The icon is `'shopping-cart'` instead of `'check'`

---

## What You Should Have When You're Done

When you click your button, the printed summary should show:

- A delivery date line (even if it says `None` — that's fine if you haven't picked a date)
- The color dropdown should show Crimson / Teal / Gold / Slate as options, defaulting to Teal
- The rating should max out at 5, not 10
- The button should be blue (info style) with a shopping cart icon and say "Place Order"

Take a screenshot of your completed form with the summary output visible and submit it.

---

## Quick Reference

| Widget | What it does |
|---|---|
| `widgets.Text()` | Single-line text input |
| `widgets.Textarea()` | Multi-line text input |
| `widgets.IntSlider()` | Integer slider with min/max |
| `widgets.FloatSlider()` | Decimal slider with min/max |
| `widgets.Dropdown()` | Pick one from a list |
| `widgets.RadioButtons()` | Pick one (all options visible) |
| `widgets.SelectMultiple()` | Pick several from a list |
| `widgets.Checkbox()` | True/false toggle |
| `widgets.ToggleButton()` | On/off button |
| `widgets.DatePicker()` | Calendar date selector |
| `widgets.Button()` | Clickable button with callback |
| `widgets.Output()` | Area to print results into |
| `widgets.VBox()` | Stack children vertically |
| `widgets.HBox()` | Place children side by side |
| `widgets.HTML()` | Display formatted HTML text |
