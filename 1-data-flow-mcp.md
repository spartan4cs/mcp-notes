
We’ll build this structure:

```
calculator_app/
│
├── main.py
├── llm/
│     └── client.py
├── mcp/
│     └── orchestrator.py
├── tools/
│     └── calculator.py
└── schemas/
      └── calculator_schema.py
```

This separates:

* LLM communication
* Tool definitions
* Tool execution
* Orchestration logic

This is how production agents are structured.

---

# 🧠 First — Diagram of Data Flow

Here is the architectural data flow:

```
                ┌─────────────────────┐
                │        User         │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │       main.py       │
                │  (Entry Point)      │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  MCP Orchestrator   │
                │ (Reason + Act Loop) │
                └──────────┬──────────┘
                           │
          ┌────────────────┴────────────────┐
          ▼                                 ▼
┌─────────────────────┐            ┌─────────────────────┐
│     LLM Client      │            │   Tool Execution    │
│   (Groq Cloud ☁️)   │            │  calculator.py      │
└─────────────────────┘            └─────────────────────┘
```

---

# 🏗 Now Let’s Build It

---

## 🟢 1️⃣ tools/calculator.py

```python
def calculator(expression: str) -> str:
    try:
        result = eval(expression)
        return str(result)
    except Exception as e:
        return f"Error: {str(e)}"
```

This is your **MCP server tool layer**.

---

## 🟢 2️⃣ schemas/calculator_schema.py

```python
calculator_schema = {
    "type": "function",
    "function": {
        "name": "calculator",
        "description": "Perform mathematical calculations",
        "parameters": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "Mathematical expression to evaluate"
                }
            },
            "required": ["expression"],
        },
    },
}
```

This is what the LLM sees.

---

## 🟢 3️⃣ llm/client.py

```python
from groq import Groq

def get_llm_client():
    return Groq()

def call_llm(client, messages, tools=None):
    return client.chat.completions.create(
        model="llama3-8b-8192",
        messages=messages,
        tools=tools,
        tool_choice="auto" if tools else None,
    )
```

This isolates your LLM layer.

If tomorrow you switch to OpenAI, you change only this file.

---

## 🟢 4️⃣ mcp/orchestrator.py

This is the brain.

```python
import json
from llm.client import call_llm
from tools.calculator import calculator

def run_agent(client, user_input, tools):

    messages = [
        {"role": "system", "content": "Use calculator tool for math problems."},
        {"role": "user", "content": user_input},
    ]

    response = call_llm(client, messages, tools)
    message = response.choices[0].message

    if message.tool_calls:
        tool_call = message.tool_calls[0]
        args = json.loads(tool_call.function.arguments)

        result = calculator(args["expression"])

        messages.append(message)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": result
        })

        final_response = call_llm(client, messages)
        return final_response.choices[0].message.content

    return message.content
```

This is your **MCP client loop**.

---

## 🟢 5️⃣ main.py

```python
from llm.client import get_llm_client
from mcp.orchestrator import run_agent
from schemas.calculator_schema import calculator_schema

def main():
    client = get_llm_client()

    while True:
        user_input = input("Ask: ")
        answer = run_agent(client, user_input, [calculator_schema])
        print("Answer:", answer)

if __name__ == "__main__":
    main()
```

---

# 🚀 Run It

From project root:

```bash
python main.py
```

Try:

```
What is 45 * 2 + 10?
```

---

# 🧠 Now Let’s Map Components Clearly

| Component     | Lives Where     | Responsibility               |
| ------------- | --------------- | ---------------------------- |
| main.py       | Entry point     | Handles user interaction     |
| orchestrator  | MCP Client      | Controls tool-calling loop   |
| llm/client.py | LLM Layer       | Talks to Groq                |
| calculator.py | MCP Server Tool | Executes math                |
| Groq          | Cloud LLM       | Decides whether to call tool |

---

# 🏢 Production-Level Architecture Diagram

If deployed in cloud:

```
User (Browser)
      ↓
FastAPI Server
      ↓
MCP Orchestrator
      ↓
LLM API (Groq/OpenAI)
      ↓
Tool Microservices
      ↓
Database / APIs
```

---

# 🧠 Important Concept You Just Learned

We separated:

* Reasoning (LLM)
* Orchestration (MCP client)
* Execution (Tools)
* Interface (main)

This is clean architecture.

---

# ⚡ Next Level Options

Now we can:

1️⃣ Add multiple tools
2️⃣ Make it a FastAPI REST API
3️⃣ Add logging to see tool calls live
4️⃣ Add memory + conversation state
5️⃣ Replace eval with safe math parser

Pick one — we scale it properly 🚀
