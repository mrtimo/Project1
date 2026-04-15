# Lab: Building a Multi-Outcome Customer Support Bot with LangChain

### System architecture — APIs and data sources
![Architecture diagram showing how user questions flow from Gradio through the classifier to different APIs and data sources](architecture_diagram.svg)

### Code flow — functions and routing logic
![Code flow diagram showing how handle_customer_message routes through Python functions](code_flow_diagram.svg)

---

## Introduction

In many real-world business applications, a single call to an LLM API is not enough. Consider a customer support chatbot for an electronics retailer. A customer might ask:

- **"Where is my order?"** → The bot needs to call an **Order API** to look up shipping status.
- **"What's the status of my support ticket?"** → The bot needs to query a **ticket database** to find open issues.
- **"Where can I recycle electronics in Spokane?"** → The bot needs to search a **knowledge base** of documents to find the answer.
- **"I want to speak to a manager."** → The bot needs to **escalate** — log the request and notify a human.

Each of these requires a completely different action. The LLM isn't just generating text — it's deciding *which tool to use*, calling it, and then composing a helpful response from the result. This is called **agentic behavior**, and it's the core pattern behind most production AI applications today.

**LangChain** is a Python framework that makes it easy to wire together these multi-step workflows. Instead of writing a massive if/else chain yourself, you define "tools" and let the LLM decide which one to call based on the user's question.

### What You'll Build

In this lab, you will build a customer support chatbot that can:

1. **Classify** the customer's question into a category
2. **Look up order information** from the DummyJSON API
3. **Check support ticket status** from a CSV file
4. **Answer knowledge base questions** using RAG (Retrieval-Augmented Generation) on uploaded markdown documents
5. **Escalate** to a human agent and log the interaction to a CSV file
6. **Save conversation history** to a CSV file for record-keeping

Here is the flow:

```
Customer Question
       │
       ▼
┌──────────────┐
│  Classifier  │  LLM decides: what kind of question is this?
└──────────────┘
       │
       ├── "order status"  ──→  Call DummyJSON API  ──→  Generate response
       │
       ├── "ticket status" ──→  Query CSV file      ──→  Generate response
       │
       ├── "knowledge base"──→  RAG on ChromaDB     ──→  Generate response
       │
       └── "escalation"    ──→  Log to CSV + notify  ──→  Generate response
       │
       ▼
┌──────────────┐
│ Save to      │  Log the conversation to memory.csv
│ Memory       │
└──────────────┘
       │
       ▼
  Bot Response
```

---

## Part 1: Get Your API Key

We'll use Google's Gemini API for this lab because it has a generous free tier.

### Steps to get your API key:

1. Go to [https://aistudio.google.com/](https://aistudio.google.com/)
2. Sign in with your Google account
3. Click **"Get API Key"** in the bottom-left corner of the page
4. Click **"Create API Key"**
5. You'll be prompted to create a Google Cloud project first — give it any name (e.g., `langchain-lab`)
6. Once the project is created, your API key will be generated — **copy it**

### Store the key in Google Colab Secrets:

1. In your Google Colab notebook, click the **🔑 key icon** in the left sidebar (or go to **Runtime → Manage Secrets**)
2. Click **"Add a new secret"**
3. Set the name to: `GOOGLE_API_KEY`
4. Paste your API key as the value
5. Toggle the switch to enable notebook access

---

## Part 2: Test Your API Key

Create a new Google Colab notebook and run the following cells.

### Cell 1: Install dependencies

```python
!pip install -q langchain langchain-google-genai chromadb langchain-community langchain-text-splitters
```

### Cell 2: Test that your key works

```python
from google.colab import userdata
import os

os.environ["GOOGLE_API_KEY"] = userdata.get("GOOGLE_API_KEY")

from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(model="gemini-flash-latest")

response = llm.invoke("Say 'Hello, LangChain is working!' and nothing else.")
print(response.content)
```

You should see: `Hello, LangChain is working!`

If you get an error, double-check that your secret name is exactly `GOOGLE_API_KEY` and that you toggled notebook access on.

---

## Part 3: Set Up the Support Ticket Data

In a real company, ticket data would live in a database. For this lab, we'll use a CSV file that has been provided to you called `customer_support_tickets1.csv`. Here it is: https://github.com/mrtimo/Project1/blob/main/customer_support_tickets1.csv

### Step 3a: Upload the ticket data file

1. Download `customer_support_tickets1.csv` from the course materials
2. In the Colab sidebar, click the **📁 folder icon**
3. Drag and drop `customer_support_tickets1.csv` into the file browser (upload it to the root, not inside a subfolder)

Alternatively, you can upload it with code:

```python
from google.colab import files
uploaded = files.upload()  # Select customer_support_tickets1.csv
```

### Cell 3: Load and explore the ticket data

```python
import pandas as pd

tickets_df = pd.read_csv("customer_support_tickets1.csv")
print(f"✅ Loaded {len(tickets_df)} tickets!")
print(f"\nColumns: {list(tickets_df.columns)}\n")
print(tickets_df[["Ticket ID", "Customer Name", "Product Purchased", "Ticket Status"]].head(10).to_string(index=False))
```

You should see ticket records with columns like Ticket ID, Customer Name, Product Purchased, Ticket Status, Ticket Priority, and more. Take a moment to scroll through the data so you understand what's in it — your bot will be querying this file when customers ask about their support tickets.

---

## Part 4: Set Up the DummyJSON Order Lookup Tool

The DummyJSON API (`https://dummyjson.com`) is a free API that provides fake e-commerce data. We'll use it to look up order information when a customer asks about their order.

### Cell 4: Create the order lookup function

```python
import requests
import json

def lookup_order(order_id: str) -> str:
    """Look up an order by ID using the DummyJSON API."""
    try:
        order_id = int(order_id)
    except ValueError:
        return f"Invalid order ID: {order_id}. Please provide a numeric order ID."

    # DummyJSON has carts that simulate orders
    url = f"https://dummyjson.com/carts/{order_id}"
    response = requests.get(url)

    if response.status_code != 200:
        return f"Order #{order_id} not found. Please check your order number and try again."

    order = response.json()

    # Build a readable summary
    products = []
    for item in order.get("products", []):
        products.append(f"  - {item['title']} (Qty: {item['quantity']}, ${item['price']:.2f} each)")

    product_list = "\n".join(products)

    summary = f"""Order #{order['id']}:
- User ID: {order.get('userId', 'N/A')}
- Total Products: {order.get('totalProducts', 'N/A')}
- Total Quantity: {order.get('totalQuantity', 'N/A')}
- Subtotal: ${order.get('total', 0):.2f}
- Discounted Total: ${order.get('discountedTotal', 0):.2f}

Products:
{product_list}"""

    return summary


# Test it
print(lookup_order("5"))
```

---

## Part 5: Set Up the Ticket Lookup Tool

### Cell 5: Create the ticket lookup function

```python
import pandas as pd

def lookup_ticket(ticket_id: str = None, customer_name: str = None, email: str = None) -> str:
    """Look up a support ticket by ticket ID, customer name, or email."""
    df = pd.read_csv("customer_support_tickets1.csv")

    if ticket_id:
        try:
            tid = int(ticket_id)
            results = df[df["Ticket ID"] == tid]
        except ValueError:
            return f"Invalid ticket ID: {ticket_id}"
    elif customer_name:
        results = df[df["Customer Name"].str.contains(customer_name, case=False, na=False)]
    elif email:
        results = df[df["Customer Email"].str.contains(email, case=False, na=False)]
    else:
        return "Please provide a ticket ID, customer name, or email to look up."

    if results.empty:
        return "No tickets found matching your query."

    # Format the results
    output_parts = []
    for _, row in results.iterrows():
        ticket_info = f"""Ticket #{int(row['Ticket ID'])}:
- Customer: {row['Customer Name']} ({row['Customer Email']})
- Product: {row['Product Purchased']}
- Type: {row['Ticket Type']} — {row['Ticket Subject']}
- Status: {row['Ticket Status']}
- Priority: {row['Ticket Priority']}
- Description: {row['Ticket Description'][:150]}...
- Resolution: {row['Resolution'] if pd.notna(row['Resolution']) and row['Resolution'] else 'Pending'}"""
        output_parts.append(ticket_info)

    return "\n\n".join(output_parts)


# Test it
print(lookup_ticket(ticket_id="1"))
print()
print(lookup_ticket(customer_name="Jessica"))
```

---

## Part 6: Set Up RAG with ChromaDB

This is the most powerful piece. We'll load markdown documents into a vector database so the bot can answer questions by searching through them.

### Step 6a: Upload your markdown files

1. In the Colab sidebar, click the **📁 folder icon**
2. Create a new folder called `docs`
3. Upload your recycling/sustainability `.md` files (e.g., `spokane_recycling.md`) into the `docs` folder
4. Here is an example file with recycling info for spokane: https://github.com/mrtimo/Project1/blob/main/recycle.md
5. 

You can also upload files programmatically:

```python
import os
os.makedirs("docs", exist_ok=True)

# Option 1: Upload manually via the file browser sidebar

# Option 2: Upload via Colab's file upload dialog
from google.colab import files
uploaded = files.upload()  # Select your .md files
for filename, content in uploaded.items():
    with open(f"docs/{filename}", "wb") as f:
        f.write(content)
    print(f"✅ Saved {filename} to docs/")
```

### Step 6b: Build the ChromaDB vector store

### Cell 6: Create the vector store from all markdown files

```python
from langchain_community.document_loaders import DirectoryLoader, TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_google_genai import GoogleGenerativeAIEmbeddings

# 1. Load ALL .md files from the docs directory using glob
loader = DirectoryLoader(
    "./docs",
    glob="**/*.md",            # This grabs any .md file in docs/ or subdirectories
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"}
)
documents = loader.load()
print(f"📄 Loaded {len(documents)} document(s)")

# 2. Split documents into chunks
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,           # Characters per chunk
    chunk_overlap=100          # Overlap between chunks to preserve context
)
chunks = splitter.split_documents(documents)
print(f"✂️  Split into {len(chunks)} chunks")

# 3. Create embeddings and store in ChromaDB
embeddings = GoogleGenerativeAIEmbeddings(model="models/gemini-embedding-001")

vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db"
)
print(f"✅ ChromaDB created with {len(chunks)} chunks!")
```

### Step 6c: Test the retriever

### Cell 7: Test that RAG retrieval works

```python
# Test a query against the vector store
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
test_results = retriever.invoke("Where can I recycle electronics in Spokane?")

for i, doc in enumerate(test_results):
    print(f"\n--- Chunk {i+1} (from {doc.metadata.get('source', 'unknown')}) ---")
    print(doc.page_content[:300] + "...")
```

You should see relevant chunks from your uploaded documents. If this works, your RAG pipeline is ready.

---

## Part 7: Create the Escalation and Memory Tools

For a production system, escalations would create tickets in Zendesk or Jira, and memory would go to a real database. For this lab, we'll log everything to CSV files.

### Cell 8: Create escalation and memory logging

```python
import csv
from datetime import datetime

def escalate_to_human(customer_message: str, reason: str = "Customer requested escalation") -> str:
    """Log an escalation to a CSV file. In production this would create a ticket."""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    # Append to escalations CSV
    file_exists = os.path.exists("escalations.csv")
    with open("escalations.csv", "a", newline="") as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(["Timestamp", "Customer Message", "Reason", "Status"])
        writer.writerow([timestamp, customer_message, reason, "Open"])

    return f"Your request has been escalated to a human agent. Reference time: {timestamp}. A support representative will reach out to you shortly."


def save_to_memory(user_message: str, bot_response: str, category: str) -> None:
    """Save the conversation turn to a CSV file for record-keeping."""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    file_exists = os.path.exists("memory.csv")
    with open("memory.csv", "a", newline="") as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(["Timestamp", "Category", "User Message", "Bot Response"])
        writer.writerow([timestamp, category, user_message, bot_response[:500]])


# Test escalation
print(escalate_to_human("I want to speak to a manager!", "Customer frustrated"))
```

---

## Part 8: Build the Classifier

This is the "brain" of the system. The classifier looks at the customer's message and decides which tool to use.

### Cell 9: Build the question classifier

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.prompts import ChatPromptTemplate

llm = ChatGoogleGenerativeAI(model="gemini-flash-latest", temperature=0)

classifier_prompt = ChatPromptTemplate.from_template("""
You are a customer support routing assistant. Classify the following customer message
into exactly ONE of these categories. Respond with ONLY the category name, nothing else.

Categories:
- ORDER_STATUS: Customer is asking about an order, shipment, delivery, or tracking
- TICKET_STATUS: Customer is asking about an existing support ticket, previous complaint, or prior issue
- KNOWLEDGE_BASE: Customer is asking a general question about programs, policies, catalog info, or anything that might be in our documentation
- ESCALATION: Customer is angry, wants to speak to a human/manager, or the issue seems urgent and unresolvable by a bot

Customer message: {message}

Category:""")

classifier_chain = classifier_prompt | llm


def extract_text(content):
    """Extract only the text from an LLM response, ignoring thought signatures."""
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        parts = []
        for item in content:
            if isinstance(item, dict) and "text" in item:
                parts.append(item["text"])
            elif isinstance(item, str):
                parts.append(item)
            elif hasattr(item, "text"):
                parts.append(item.text)
        return " ".join(parts)
    return str(content)


def classify_message(message: str) -> str:
    """Classify a customer message into a routing category."""
    result = classifier_chain.invoke({"message": message})
    category = extract_text(result.content).strip().upper()

    # Normalize the response
    valid_categories = ["ORDER_STATUS", "TICKET_STATUS", "KNOWLEDGE_BASE", "ESCALATION"]
    for valid in valid_categories:
        if valid in category:
            return valid

    return "KNOWLEDGE_BASE"  # Default fallback


# Test the classifier
test_messages = [
    "Where is my order #5?",
    "What's the status of my ticket? My name is Jessica Rios.",
    "Where can I recycle old batteries in Spokane?",
    "This is ridiculous! I want to speak to a manager right now!",
    "What goes in the blue recycling bin?",
    "I placed order number 12 last week, has it shipped?"
]

for msg in test_messages:
    category = classify_message(msg)
    print(f"  [{category:>15}]  {msg}")
```

---

## Part 9: Wire It All Together

Now we connect every piece. The main function classifies the message, routes to the right tool, generates a response, and saves to memory.

### Cell 10: The complete customer support bot

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_google_genai import ChatGoogleGenerativeAI
import re

llm = ChatGoogleGenerativeAI(model="gemini-flash-latest", temperature=0.3)

response_prompt = ChatPromptTemplate.from_template("""
You are a friendly and professional customer support agent. Using the context below,
write a helpful response to the customer's message.

Be concise but thorough. If the context doesn't fully answer the question, say so honestly
and suggest next steps.

Context from our systems:
{context}

Customer's message:
{message}

Your response:""")

response_chain = response_prompt | llm


def extract_order_id(message: str) -> str:
    """Try to extract an order ID number from the message."""
    numbers = re.findall(r'#?(\d+)', message)
    return numbers[0] if numbers else "1"


def extract_ticket_info(message: str) -> dict:
    """Try to extract ticket ID or customer name from the message."""
    # Check for ticket ID
    ticket_match = re.findall(r'ticket\s*#?\s*(\d+)', message, re.IGNORECASE)
    if ticket_match:
        return {"ticket_id": ticket_match[0]}

    # Check for names (very simplified — in production you'd use NER)
    # For this lab, we'll check against known names in our data
    df = pd.read_csv("customer_support_tickets1.csv")
    for name in df["Customer Name"]:
        if name.lower() in message.lower() or name.split()[0].lower() in message.lower():
            return {"customer_name": name}

    for email in df["Customer Email"]:
        if email.lower() in message.lower():
            return {"email": email}

    return {"ticket_id": "1"}  # Fallback


def handle_customer_message(message: str) -> str:
    """Main function: classify, route, respond, and log."""

    # Step 1: Classify
    category = classify_message(message)
    print(f"🏷️  Classified as: {category}")

    # Step 2: Route to the right tool and get context
    if category == "ORDER_STATUS":
        order_id = extract_order_id(message)
        context = lookup_order(order_id)
        print(f"📦 Looked up order #{order_id}")

    elif category == "TICKET_STATUS":
        ticket_info = extract_ticket_info(message)
        context = lookup_ticket(**ticket_info)
        print(f"🎫 Looked up ticket: {ticket_info}")

    elif category == "KNOWLEDGE_BASE":
        retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
        docs = retriever.invoke(message)
        context = "\n\n---\n\n".join([doc.page_content for doc in docs])
        sources = [doc.metadata.get("source", "unknown") for doc in docs]
        print(f"📚 Retrieved {len(docs)} chunks from: {set(sources)}")

    elif category == "ESCALATION":
        context = escalate_to_human(message)
        print(f"🚨 Escalated!")

    else:
        context = "No specific context available."

    # Step 3: Generate response using LLM
    result = response_chain.invoke({"context": context, "message": message})
    bot_response = extract_text(result.content)

    # Step 4: Save to memory
    save_to_memory(message, bot_response, category)
    print(f"💾 Saved to memory\n")

    return bot_response
```

---

## Part 10: Test Your Bot!

### Cell 11: Run test conversations

```python
# Test 1: Order status
print("=" * 60)
print("TEST 1: Order Status")
print("=" * 60)
response = handle_customer_message("Hi, can you check on my order #5? I haven't received it yet.")
print(f"🤖 Bot: {response}\n")

# Test 2: Ticket status
print("=" * 60)
print("TEST 2: Ticket Status")
print("=" * 60)
response = handle_customer_message("I submitted a ticket a while ago. My name is Marisa Obrien. What's happening with it?")
print(f"🤖 Bot: {response}\n")

# Test 3: Knowledge base (RAG)
print("=" * 60)
print("TEST 3: Knowledge Base (RAG)")
print("=" * 60)
response = handle_customer_message("Where can I drop off old electronics for recycling in Spokane?")
print(f"🤖 Bot: {response}\n")

# Test 4: Escalation
print("=" * 60)
print("TEST 4: Escalation")
print("=" * 60)
response = handle_customer_message("This is unacceptable! I've been waiting for weeks. Let me talk to a real person!")
print(f"🤖 Bot: {response}\n")
```

### Cell 12: Interactive chatbot UI with Gradio

Instead of a basic `input()` loop, we'll use **Gradio** to create a real chat interface right inside Colab. This gives you a proper chatbot UI where you can ask multiple questions and see the full conversation history.

First, install Gradio:

```python
!pip install -q gradio
```

Then create the chat interface:

```python
import gradio as gr

def chat_fn(message, history):
    """Gradio chat function — takes a message and conversation history."""
    response = handle_customer_message(message)
    return response

demo = gr.ChatInterface(
    fn=chat_fn,
    title="🤖 Customer Support Bot",
    description="Ask about orders, support tickets, recycling info, or request escalation.",
    examples=[
        "Where is my order #3?",
        "What's the status of my ticket? My name is Marisa Obrien.",
        "Where can I recycle old batteries in Spokane?",
        "What goes in the blue bin vs the gray bin?",
        "I want to speak to a manager!",
    ],
    theme="soft",
)

demo.launch()
```

This will open a chat window with a text box, send button, and clickable example questions. You can have a full multi-turn conversation — ask a question, read the answer, then follow up. The conversation history is displayed in the chat window so you can scroll back through it.

> **Note:** Gradio runs a small web server inside Colab. You'll see a URL in the output — click it to open the chat in a new tab, or use the embedded interface directly in the notebook.

---

## Part 11: Inspect the Logs

After running some conversations, check the CSV files to see what was recorded.

### Cell 13: View the logs

```python
# View conversation memory
print("📝 CONVERSATION MEMORY")
print("=" * 60)
if os.path.exists("memory.csv"):
    memory_df = pd.read_csv("memory.csv")
    for _, row in memory_df.iterrows():
        print(f"\n[{row['Timestamp']}] Category: {row['Category']}")
        print(f"  User: {row['User Message'][:80]}...")
        print(f"  Bot:  {row['Bot Response'][:80]}...")
else:
    print("No conversations logged yet.")

print("\n")

# View escalations
print("🚨 ESCALATIONS")
print("=" * 60)
if os.path.exists("escalations.csv"):
    esc_df = pd.read_csv("escalations.csv")
    print(esc_df.to_string(index=False))
else:
    print("No escalations logged yet.")
```

---

## Wrapping Up: The Mental Model

Let's step back and understand what you just built, because the pattern you followed here is the same pattern behind most production AI applications.

### The Core Idea: LLMs as Routers

The most important concept in this lab isn't any single tool — it's the **pattern**. You used an LLM not just to generate text, but to **make a decision**. The classifier looked at a customer's message and chose a path. This is fundamentally different from a traditional chatbot with hardcoded keyword matching. The LLM understands *intent*, not just keywords.

### The Chain of Operations

Every message flowed through the same pipeline:

```
Input → Classify → Route → Retrieve Context → Generate Response → Log
```

This is what LangChain means by a "chain" — a sequence of operations where each step feeds into the next. In this lab you built the chain manually with Python functions, but LangChain provides abstractions (like `RunnableSequence`, `Tool`, and `Agent`) that formalize this pattern and make it easier to extend.

### What Each Piece Does

| Component | What It Does | Real-World Equivalent |
|-----------|-------------|----------------------|
| **Classifier** | LLM decides which category a message belongs to | A receptionist routing your call |
| **Order Lookup (DummyJSON)** | Calls an external API to get structured data | Querying Shopify, your ERP, or a database |
| **Ticket Lookup (CSV)** | Searches a local data file for matching records | Querying Zendesk, Jira, or Salesforce |
| **RAG (ChromaDB)** | Finds semantically similar text chunks from documents | Searching a knowledge base or internal wiki |
| **Escalation (CSV)** | Logs that a human needs to intervene | Creating a Zendesk ticket or Slack alert |
| **Memory (CSV)** | Records the conversation for later review | Writing to a CRM or conversation analytics tool |

### RAG in 30 Seconds

RAG stands for **Retrieval-Augmented Generation**. Here's what happens:

1. **Chunk**: Your documents get split into small pieces (~1000 characters each)
2. **Embed**: Each chunk gets converted into a vector (a list of numbers representing its meaning)
3. **Store**: The vectors go into ChromaDB (a specialized database for similarity search)
4. **Retrieve**: When a question comes in, it also gets embedded, and ChromaDB finds the most similar chunks
5. **Generate**: Those chunks get stuffed into the LLM's prompt as context, and the LLM writes an answer based on them

The LLM never "reads" your documents directly. It reads the *most relevant pieces* that the retriever found. This is why chunk size and overlap matter — too big and you waste context, too small and you lose meaning.

### Why Not Just Put Everything in the Prompt?

You might wonder: why not just paste all the documents, ticket data, and order info into one giant prompt? Three reasons:

1. **Context limits**: LLMs have a maximum input size. Your recycling knowledge base alone might exceed it.
2. **Cost**: You pay per token. Sending 50 pages of docs with every question is expensive.
3. **Accuracy**: LLMs actually perform *worse* with too much irrelevant context. Retrieval focuses the LLM on what matters.

### What Would Change in Production?

The architecture you built is real, but in production:

- The **CSV files** would be replaced by databases (PostgreSQL, MongoDB) and ticketing systems (Zendesk, Jira)
- The **DummyJSON API** would be your actual order management system (Shopify, internal ERP)
- The **ChromaDB** would run on a server (Pinecone, Weaviate, or a hosted Chroma instance) instead of locally
- The **memory** would use a proper conversation store (Redis, DynamoDB) with session IDs
- The **escalation** would trigger real workflows — creating tickets, sending Slack messages, emailing managers
- You'd add **guardrails** — checking the LLM's response before sending it to make sure it doesn't promise unauthorized refunds or share internal data
- You'd add **authentication** — knowing *which* customer is chatting so you can look up their specific orders and tickets automatically

But the *pattern* — classify, route, retrieve, generate, log — stays exactly the same.

### Key Takeaways

1. **LangChain is plumbing, not magic.** It wires together LLM calls, API calls, and database queries into a coherent pipeline.
2. **The LLM makes routing decisions.** Instead of writing complex if/else logic, you let the LLM understand intent.
3. **RAG lets you give an LLM knowledge it wasn't trained on.** Your recycling guides, your company's policies, your product docs — anything you can chunk and embed.
4. **Every "tool" is just a Python function.** Whether it calls an API, queries a database, or reads a file — the LLM decides *which* function to call, and your code executes it.
5. **Production systems use this exact architecture.** The tools get fancier, but the flow is the same one you built today.

---

## Bonus Challenge

If you finish early, try one or more of these:

- **Add a new tool**: Create a `lookup_product` function that calls `https://dummyjson.com/products/search?q={query}` and wire it into the classifier as a `PRODUCT_INFO` category.
- **Improve the classifier**: Add few-shot examples to the classifier prompt to make it more accurate.
- **Add more documents**: Upload additional `.md` files to the `docs/` folder, rebuild ChromaDB, and test that RAG finds the right answers.
- **Deploy to Hugging Face Spaces**: Follow the instructions below to get a permanent public URL for your bot.

---

## Bonus: Deploy to Hugging Face Spaces

You can host your bot for free on Hugging Face Spaces so anyone can use it from a permanent URL — no Colab notebook required.

### What you need

Two files are provided with this lab:

- `app.py` — your entire bot consolidated into one Python file
- `requirements.txt` — the Python packages Hugging Face needs to install

### Steps

1. **Create a Hugging Face account** at [huggingface.co](https://huggingface.co) (free)

2. **Create a new Space**
   - Go to [huggingface.co/new-space](https://huggingface.co/new-space)
   - Give it a name (e.g., `customer-support-bot`)
   - Select **Gradio** as the SDK
   - Choose **Public** (so others can see it) or **Private**
   - Click **Create Space**

3. **Add your API key as a secret**
   - In your new Space, go to **Settings → Variables and secrets**
   - Click **New secret**
   - Name: `GOOGLE_API_KEY`
   - Value: paste your Gemini API key
   - This keeps your key private — it won't be visible in your code

4. **Upload your files**
   - Upload `app.py` and `requirements.txt` to the root of the Space repo
   - Create a `docs/` folder and upload your `.md` files into it
   - Optionally upload `customer_support_tickets1.csv` (if you don't, the app creates sample data)

   Your Space repo should look like:
   ```
   your-space/
   ├── app.py
   ├── requirements.txt
   ├── customer_support_tickets1.csv   (optional)
   └── docs/
       └── spokane_recycling.md        (your .md files)
   ```

5. **Wait for it to build** — Hugging Face will install dependencies and start your app. This takes 2-3 minutes. Once it's live, you'll see the Gradio chat interface at `https://huggingface.co/spaces/YOUR_USERNAME/YOUR_SPACE_NAME`.

### What's different from the Colab version?

The `app.py` file is the same code from this lab, with two small changes:

- **API key**: reads from `os.environ["GOOGLE_API_KEY"]` instead of Colab's `userdata.get()`, since Hugging Face stores secrets as environment variables
- **Fallback data**: if the CSV or docs folder are missing, it creates sample data automatically so the app doesn't crash on first launch
