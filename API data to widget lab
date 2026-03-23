# Lab 2: Populating Forms with Live API Data

## Overview

In Lab 1 you built interactive forms with manually typed options. In this lab you'll learn to **fetch data from a live API** and use it to fill in your form widgets automatically. We'll use the free PokéAPI (pokeapi.co) — no API key needed.

**Prerequisite:** Lab 1 (you should be comfortable with widgets, buttons, and VBox/HBox layouts)

**Time:** ~40 minutes

---

## Part 1: Making Your First API Call

APIs (Application Programming Interfaces) let your code request data from a server. The pattern is simple: you send a request to a URL, and you get back structured data (usually JSON).

Paste this into a **new Colab notebook** cell and run it:

```python
import requests
import json

response = requests.get("https://pokeapi.co/api/v2/pokemon/pikachu")
data = response.json()

print(f"Status code: {response.status_code}")
print(f"Name: {data['name']}")
print(f"Height: {data['height']}")
print(f"Weight: {data['weight']}")
```

You should see Pikachu's name, height, and weight printed out. A status code of `200` means the request was successful.

### Understanding the response

The JSON that comes back is a Python dictionary. You access values with bracket notation like `data['name']`. In a **new cell**, paste this to explore the structure:

```python
print("Top-level keys in the response:")
for key in data.keys():
    print(f"  {key}")
```

There are a lot of keys. We'll focus on a few useful ones: `types`, `stats`, and `abilities`.

---

## Part 2: Fetching a List from the API

The PokéAPI has "list" endpoints that return many items at once. Paste this in a **new cell**:

```python
response = requests.get("https://pokeapi.co/api/v2/type")
type_data = response.json()

print(f"Number of types: {type_data['count']}")
print(f"\nFirst 5 results:")
for t in type_data['results'][:5]:
    print(f"  {t['name']}")
```

Notice the response has a `results` key containing a list of dictionaries. Each dictionary has a `name` and a `url`. This pattern is the same across most PokéAPI list endpoints.

Now let's turn those results into a simple Python list of names. **New cell:**

```python
type_names = [t['name'].title() for t in type_data['results']]
print(type_names)
```

The `.title()` method capitalizes the first letter of each word. This list is exactly what we need to feed into a dropdown.

---

## Part 3: API Data → Widget Options

Now let's connect the dots. We'll fetch Pokémon names from the API and put them into a dropdown. Paste this in a **new cell**:

```python
import ipywidgets as widgets
from IPython.display import display

# Fetch the first 50 Pokémon names from the API
pokemon_response = requests.get("https://pokeapi.co/api/v2/pokemon?limit=50")
pokemon_data = pokemon_response.json()
pokemon_names = [p['name'].title() for p in pokemon_data['results']]

# Create a dropdown populated with API data
pokemon_dropdown = widgets.Dropdown(
    options=pokemon_names,
    description='Pokémon:'
)

display(pokemon_dropdown)
```

Open the dropdown — you should see 50 Pokémon names that were fetched **live from the internet**, not hard-coded by you. That's the core idea of this lab.

---

## Part 4: Building a Full API-Powered Form

Now let's build a complete form where **every selection widget** gets its options from the API. Paste this entire block into a **new cell**:

```python
# --- Fetch data from multiple API endpoints ---

# Pokémon names (first 100)
poke_resp = requests.get("https://pokeapi.co/api/v2/pokemon?limit=100")
poke_names = [p['name'].title() for p in poke_resp.json()['results']]

# Types (filter out "unknown" and "shadow")
types_resp = requests.get("https://pokeapi.co/api/v2/type")
type_options = [t['name'].title() for t in types_resp.json()['results'] if t['name'] not in ('unknown', 'shadow')]

# Natures (all 25)
natures_resp = requests.get("https://pokeapi.co/api/v2/nature?limit=25")
nature_options = [n['name'].title() for n in natures_resp.json()['results']]

# --- Build widgets using API data ---
pokemon_pick = widgets.Dropdown(options=poke_names, description='Pokémon:', style={'description_width': '90px'})
nature_pick = widgets.Dropdown(options=nature_options, description='Nature:', style={'description_width': '90px'})
type_filter = widgets.SelectMultiple(options=type_options, description='Types:', rows=6, style={'description_width': '90px'})
level_slider = widgets.IntSlider(value=50, min=1, max=100, description='Level:', style={'description_width': '90px'})
shiny_check = widgets.Checkbox(value=False, description='Shiny?')

output = widgets.Output()
sprite_box = widgets.Output()
```

This cell does the setup. Now paste the **button logic** in the **next cell**:

```python
lookup_btn = widgets.Button(description='Look Up', button_style='info', icon='search')

def on_lookup(b):
    output.clear_output()
    sprite_box.clear_output()
    name = pokemon_pick.value.lower()
    resp = requests.get(f"https://pokeapi.co/api/v2/pokemon/{name}")
    info = resp.json()

    with sprite_box:
        sprite_key = 'front_shiny' if shiny_check.value else 'front_default'
        sprite_url = info['sprites'][sprite_key]
        if sprite_url:
            display(widgets.Image.from_url(sprite_url))

    with output:
        print(f"=== {info['name'].title()} #{info['id']} ===")
        print(f"Types:     {', '.join(t['type']['name'].title() for t in info['types'])}")
        print(f"Height:    {info['height'] / 10} m")
        print(f"Weight:    {info['weight'] / 10} kg")
        print(f"Abilities: {', '.join(a['ability']['name'].replace('-',' ').title() for a in info['abilities'])}")
        print(f"\nBase Stats:")
        for s in info['stats']:
            bar = '█' * (s['base_stat'] // 5)
            print(f"  {s['stat']['name'].replace('-',' ').title():20s} {s['base_stat']:>3d} {bar}")

lookup_btn.on_click(on_lookup)
```

And finally paste the **layout** in one more **new cell**:

```python
title = widgets.HTML("<h2>🔴 Pokédex Lookup</h2><p><i>All dropdown options loaded from PokéAPI</i></p>")

left = widgets.VBox([pokemon_pick, nature_pick, level_slider, shiny_check, lookup_btn])
right = widgets.VBox([widgets.Label("Hold Ctrl/Cmd for multiple:"), type_filter, sprite_box])

form = widgets.VBox([
    title,
    widgets.HBox([left, widgets.Box(layout=widgets.Layout(width='20px')), right]),
    output
], layout=widgets.Layout(padding='15px', border='2px solid #cc0000', border_radius='10px', width='70%'))

display(form)
```

Run all three cells in order. You should see a two-column form. Pick a Pokémon and click "Look Up" to see its stats and sprite.

---

## Part 5: Your Turn — Small Modifications

Complete **all four tasks** below.

### Task 1: Fetch more Pokémon

Right now we load 100 Pokémon. Change the API call so it loads the first **151** Pokémon (the original Generation 1). You only need to change one number in one URL.

> **Check:** Open the dropdown. The last Pokémon in the list should be **Mew**.

### Task 2: Add an Abilities dropdown

The API has an endpoint for abilities: `https://pokeapi.co/api/v2/ability?limit=30`

Do the following:

1. Add a `requests.get()` call to fetch from that URL (put it in the cell with the other API calls)
2. Build a list of ability names from the response, the same way we did for types and natures
3. Create a new `widgets.Dropdown` using those ability names, with the description `'Ability:'`
4. Add the new dropdown to the **left column** of the form layout, between `nature_pick` and `level_slider`

> **Check:** Your form should now have four dropdowns: Pokémon, Nature, Ability, and the type multi-select on the right.

### Task 3: Add the selected nature and ability to the printout

In the `on_lookup` function, add two more `print()` lines so the output also shows:

- The currently selected **nature** from `nature_pick`
- The currently selected **ability** from your new ability dropdown

Place these lines after the "Abilities" line and before "Base Stats."

> **Check:** When you click Look Up, the output should include lines like:
> ```
> Selected Nature: Hardy
> Selected Ability: Stench
> ```
> (The exact values depend on what you've selected in the form.)

### Task 4: Change the border color

Change the form border from red (`#cc0000`) to a dark blue (`#1a237e`).

> **Check:** The form should have a dark blue border instead of red.

---

## What You Should Have When You're Done

Your completed notebook should show:

1. A dropdown with **151** Pokémon (ending with Mew)
2. **Four** selection widgets powered by API data: Pokémon, Nature, Ability, and Types
3. Clicking "Look Up" prints stats **plus** your selected nature and ability
4. The form border is **dark blue**
5. The sprite image still loads correctly (including shiny toggle)

Take a screenshot of your completed form showing a Pokémon lookup result (with the nature and ability lines visible) and submit it.

---

## Quick Reference

### API Call Pattern
```python
response = requests.get("https://some-api.com/endpoint")
data = response.json()              # Parse JSON into a Python dict
items = data['results']             # Access the list of results
names = [item['name'] for item in items]  # Extract just the names
```

### Common PokéAPI Endpoints

| Endpoint | What it returns |
|---|---|
| `/pokemon?limit=N` | List of N Pokémon names |
| `/pokemon/{name}` | Full details for one Pokémon |
| `/type` | All Pokémon types |
| `/nature?limit=25` | All 25 natures |
| `/ability?limit=N` | List of N abilities |
| `/generation/{id}` | Pokémon in a specific generation |

### Feeding API Data into a Widget
```python
response = requests.get("https://pokeapi.co/api/v2/some-endpoint")
options = [item['name'].title() for item in response.json()['results']]

my_dropdown = widgets.Dropdown(options=options, description='Pick one:')
```
