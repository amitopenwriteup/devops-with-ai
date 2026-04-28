# AI Agent Workshop Guide
### Build Your First AI Agent — Step by Step

---

## Welcome to the Workshop

In this workshop, you will learn what an AI agent is, how it thinks, and how to build one from scratch using Python and the Anthropic API. By the end, you will have a working AI agent that can plan, use tools, and solve multi-step problems on its own.

**Prerequisites:** Basic Python knowledge, a computer with internet access, and curiosity!

**Duration:** 90–120 minutes

**What you will build:** A personal AI agent that can do math, analyze text, convert units, and answer multi-step questions — all by itself.

---

## How This Guide Works

Each section has three parts:

- **Concept** — What you need to understand
- **Activity** — What you will do
- **Checkpoint** — How to verify it worked

Read every concept before jumping to the activity. Do not skip checkpoints — they catch common mistakes early.

---

## Module 1 — What is an AI Agent?

### Concept

A regular AI (like a basic chatbot) takes one input and produces one output. You ask, it answers. Done.

An **AI agent** is different. It:

1. Receives a **goal** (not just a question)
2. **Plans** how to reach that goal
3. **Picks and uses tools** (like a calculator, web search, or a database)
4. **Observes** the result
5. **Decides** whether to use another tool or give a final answer
6. Repeats steps 2–5 until the goal is complete

Think of it like a human assistant who figures things out on their own, rather than waiting for instructions at every step.

### The Agent Loop

```
Goal → Think → Use Tool → Observe → Think Again → Answer
                  ↑_______________|  (repeats if needed)
```

### Activity 1.1 — Spot the Agent

Read each scenario and decide: is this a basic AI or an AI agent?

| Scenario | Basic AI or Agent? |
|---|---|
| You type "What is 2+2?" and the AI says "4" | ? |
| You say "Book me a flight to Delhi" and the AI searches flights, checks your calendar, and confirms a booking | ? |
| You ask "Summarize this email" and the AI summarizes it | ? |
| You say "Research the top 3 phones under ₹20,000 and compare their cameras" and the AI searches, reads specs, and produces a comparison | ? |

**Answers:** Basic AI, Agent, Basic AI, Agent.

### Checkpoint 1

Before moving on, answer in your notes:

- In your own words, what is the difference between a chatbot and an AI agent?
- Name two tools a real-world AI agent might use.

---

## Module 2 — Setting Up Your Environment

### Concept

To build an AI agent, you need:

1. **Python 3.9+** installed on your computer
2. The **Anthropic Python SDK** (the official library to talk to Claude)
3. An **API key** from Anthropic

The API key is like a password that lets your code talk to Claude. Keep it private — never share it or put it in public code.

### Activity 2.1 — Install Python and the SDK

Open your terminal (or command prompt on Windows) and run:

```bash
python --version
```

You should see something like `Python 3.11.4`. If not, download Python from [python.org](https://python.org).

Now install the Anthropic SDK:

```bash
pip install anthropic
```

Verify it installed correctly:

```bash
python -c "import anthropic; print('SDK ready!')"
```

You should see: `SDK ready!`

### Activity 2.2 — Set Your API Key

Never put your API key directly in code. Use an environment variable instead.

**On Mac/Linux:**

```bash
export ANTHROPIC_API_KEY="your-key-here"
```

**On Windows (Command Prompt):**

```cmd
set ANTHROPIC_API_KEY=your-key-here
```

**On Windows (PowerShell):**

```powershell
$env:ANTHROPIC_API_KEY="your-key-here"
```

### Activity 2.3 — Test the Connection

Create a new file called `test_connection.py` and paste this code:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=100,
    messages=[
        {"role": "user", "content": "Say hello in exactly 5 words."}
    ]
)

print(response.content[0].text)
```

Run it:

```bash
python test_connection.py
```

### Checkpoint 2

You should see a short 5-word greeting from Claude. If you get an error:

- `AuthenticationError` → your API key is wrong or not set
- `ModuleNotFoundError` → run `pip install anthropic` again
- `ConnectionError` → check your internet connection

---

## Module 3 — Giving the Agent Tools

### Concept

An agent without tools is just a chatbot. Tools are what make agents powerful.

In the Anthropic API, you define tools as structured descriptions. Claude reads these descriptions and decides when and how to use each tool. You write the actual tool logic in Python — Claude just tells you which tool to call and with what inputs.

This separation is important:
- **Claude decides** which tool to use
- **Your Python code** actually runs the tool
- **Claude receives** the result and decides what to do next

### Activity 3.1 — Define Your First Tool

Create a file called `tools.py`:

```python
# tools.py — All the tools our agent can use

import math
from datetime import datetime

def calculator(expression: str) -> str:
    """Safely evaluate a mathematical expression."""
    try:
        # Only allow safe math operations
        allowed = {
            "__builtins__": {},
            "abs": abs, "round": round, "pow": pow,
            "sqrt": math.sqrt, "floor": math.floor,
            "ceil": math.ceil, "pi": math.pi
        }
        result = eval(expression, allowed)
        return f"Result: {result}"
    except Exception as e:
        return f"Error: {str(e)}"


def get_current_datetime() -> str:
    """Return the current date and time."""
    now = datetime.now()
    return now.strftime("Date: %A, %d %B %Y | Time: %I:%M %p")


def analyze_text(text: str) -> str:
    """Analyze a piece of text and return basic statistics."""
    words = text.split()
    word_count = len(words)
    char_count = len(text)
    sentences = text.count('.') + text.count('!') + text.count('?')
    longest = max(words, key=len) if words else ""
    unique = len(set(w.lower().strip('.,!?') for w in words))

    return (
        f"Word count: {word_count}\n"
        f"Character count: {char_count}\n"
        f"Sentence count: {sentences}\n"
        f"Unique words: {unique}\n"
        f"Longest word: '{longest}' ({len(longest)} letters)"
    )


def convert_units(value: float, from_unit: str, to_unit: str) -> str:
    """Convert between common units."""
    conversions = {
        ("km", "miles"): lambda x: x * 0.621371,
        ("miles", "km"): lambda x: x * 1.60934,
        ("kg", "lbs"): lambda x: x * 2.20462,
        ("lbs", "kg"): lambda x: x * 0.453592,
        ("celsius", "fahrenheit"): lambda x: (x * 9/5) + 32,
        ("fahrenheit", "celsius"): lambda x: (x - 32) * 5/9,
        ("meters", "feet"): lambda x: x * 3.28084,
        ("feet", "meters"): lambda x: x * 0.3048,
        ("liters", "gallons"): lambda x: x * 0.264172,
        ("gallons", "liters"): lambda x: x * 3.78541,
    }
    key = (from_unit.lower(), to_unit.lower())
    if key in conversions:
        result = conversions[key](value)
        return f"{value} {from_unit} = {round(result, 4)} {to_unit}"
    return f"Sorry, conversion from {from_unit} to {to_unit} is not supported."
```

Test it in the terminal:

```bash
python -c "from tools import calculator; print(calculator('2 ** 10'))"
```

You should see: `Result: 1024`

### Activity 3.2 — Describe Tools for Claude

Claude needs tool descriptions in a specific format. Create `tool_definitions.py`:

```python
# tool_definitions.py — Descriptions Claude reads to know what tools exist

TOOLS = [
    {
        "name": "calculator",
        "description": (
            "Evaluate a mathematical expression. Use this for any arithmetic, "
            "percentages, powers, or financial calculations. "
            "Example expressions: '15 * 47.80 / 100', '1000 * (1.08 ** 10)'"
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "A valid Python math expression to evaluate."
                }
            },
            "required": ["expression"]
        }
    },
    {
        "name": "get_current_datetime",
        "description": "Get the current date and time. Use when the user asks about today's date, the current time, or needs time-based calculations.",
        "input_schema": {
            "type": "object",
            "properties": {}
        }
    },
    {
        "name": "analyze_text",
        "description": "Analyze a piece of text and return statistics like word count, character count, longest word, and number of unique words.",
        "input_schema": {
            "type": "object",
            "properties": {
                "text": {
                    "type": "string",
                    "description": "The text to analyze."
                }
            },
            "required": ["text"]
        }
    },
    {
        "name": "convert_units",
        "description": (
            "Convert a value from one unit to another. "
            "Supported: km/miles, kg/lbs, celsius/fahrenheit, meters/feet, liters/gallons."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "value": {
                    "type": "number",
                    "description": "The numeric value to convert."
                },
                "from_unit": {
                    "type": "string",
                    "description": "The unit to convert from."
                },
                "to_unit": {
                    "type": "string",
                    "description": "The unit to convert to."
                }
            },
            "required": ["value", "from_unit", "to_unit"]
        }
    }
]
```

### Checkpoint 3

Open `tool_definitions.py` and answer:

- What does the `input_schema` section tell Claude?
- Why does `get_current_datetime` have an empty `properties` object?
- Could Claude call `calculator` without you writing the `eval()` logic? (No — Claude only decides, your code executes.)

---

## Module 4 — Building the Agent Loop

### Concept

The core of any agent is a **loop**. The loop keeps running until Claude gives a final answer without calling any more tools.

Here is the logic in plain English:

```
1. Send the user's goal + all tool definitions to Claude
2. Claude responds with EITHER:
   a. A tool call  → run that tool, send the result back, go to step 2
   b. A text answer → the task is done, show the answer and stop
```

This loop can run many times for complex tasks. For simple tasks it may only loop once.

### Activity 4.1 — Build the Agent

Create `agent.py`:

```python
# agent.py — The AI Agent

import anthropic
from tools import calculator, get_current_datetime, analyze_text, convert_units
from tool_definitions import TOOLS

client = anthropic.Anthropic()

# Map tool names to actual Python functions
TOOL_FUNCTIONS = {
    "calculator": lambda inputs: calculator(inputs["expression"]),
    "get_current_datetime": lambda inputs: get_current_datetime(),
    "analyze_text": lambda inputs: analyze_text(inputs["text"]),
    "convert_units": lambda inputs: convert_units(
        inputs["value"], inputs["from_unit"], inputs["to_unit"]
    ),
}


def run_agent(goal: str, verbose: bool = True) -> str:
    """
    Run the agent loop until Claude produces a final answer.

    Args:
        goal: The task or question for the agent to solve.
        verbose: If True, print each step as it happens.

    Returns:
        The agent's final answer as a string.
    """

    messages = [{"role": "user", "content": goal}]
    step = 0

    if verbose:
        print(f"\n{'='*60}")
        print(f"GOAL: {goal}")
        print(f"{'='*60}")

    while True:
        step += 1

        # Send current conversation + tools to Claude
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1024,
            tools=TOOLS,
            messages=messages
        )

        if verbose:
            print(f"\n[Step {step}] Claude's decision: {response.stop_reason}")

        # If Claude is done, extract and return the final text answer
        if response.stop_reason == "end_turn":
            for block in response.content:
                if hasattr(block, "text"):
                    if verbose:
                        print(f"\nFINAL ANSWER:\n{block.text}")
                    return block.text
            return "Agent completed with no text response."

        # Claude wants to use a tool
        if response.stop_reason == "tool_use":

            # Add Claude's response to the conversation history
            messages.append({"role": "assistant", "content": response.content})

            # Process each tool call Claude requested
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    tool_name = block.name
                    tool_inputs = block.input

                    if verbose:
                        print(f"\n  → Calling tool: {tool_name}")
                        print(f"    Inputs: {tool_inputs}")

                    # Run the actual tool function
                    if tool_name in TOOL_FUNCTIONS:
                        result = TOOL_FUNCTIONS[tool_name](tool_inputs)
                    else:
                        result = f"Error: Tool '{tool_name}' not found."

                    if verbose:
                        print(f"    Result: {result}")

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # Send tool results back to Claude
            messages.append({"role": "user", "content": tool_results})

        else:
            # Unexpected stop reason
            break

    return "Agent stopped unexpectedly."


if __name__ == "__main__":
    # Quick test
    answer = run_agent(
        "What is 18% tip on a bill of ₹2,450? Show me the tip amount and the total."
    )
```

Run your agent:

```bash
python agent.py
```

### Checkpoint 4

Watch the terminal output carefully. You should see:

- The goal printed at the top
- Claude's decision at each step (`tool_use` or `end_turn`)
- The tool name and inputs Claude chose
- The tool result
- A final answer

If the loop runs more than 10 times without stopping, add this safety check inside your `while True:` loop:

```python
if step > 10:
    return "Agent stopped: too many steps."
```

---

## Module 5 — Testing Your Agent

### Concept

A good agent should handle a variety of tasks — including tasks that need multiple tools in one session. Now you will test it with different types of goals.

### Activity 5.1 — Run These Test Tasks

Create a file called `test_agent.py`:

```python
# test_agent.py — Test the agent with different tasks

from agent import run_agent

tasks = [
    # Single tool — calculator
    "What is the compound interest on ₹10,000 at 7% per year for 5 years?",

    # Single tool — text analysis
    "Analyze this sentence: 'Artificial intelligence is transforming every industry around the world today.'",

    # Single tool — unit conversion
    "Convert 42 kilometers to miles.",

    # Multiple tools in one task
    "Convert 100 pounds to kilograms AND calculate 15% of the result.",

    # Date + math
    "What is today's date? How many days are left in this year?",
]

for i, task in enumerate(tasks, 1):
    print(f"\n{'#'*60}")
    print(f"TEST {i}")
    run_agent(task)
    input("\nPress Enter to continue to the next test...\n")
```

Run it:

```bash
python test_agent.py
```

### Activity 5.2 — Observe and Record

While each test runs, note in your worksheet:

| Test | How many tool calls did the agent make? | Which tools did it use? | Did it give a correct answer? |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |

### Checkpoint 5

Test 4 (multiple tools) should show the agent calling two different tools in sequence. If it only calls one, re-read your goal — make sure the task genuinely requires both.

---

## Module 6 — Add Your Own Tool

### Concept

Real agents have many tools. The power of the tool-use pattern is that adding a new tool takes just two steps:

1. Write the Python function
2. Add a description to `TOOLS`

The agent loop does not change at all — Claude automatically learns to use new tools from their descriptions.

### Activity 6.1 — Build a Word Counter Tool (Warm-up)

Add this function to `tools.py`:

```python
def count_vowels(text: str) -> str:
    """Count vowels and consonants in a text."""
    vowels = set("aeiouAEIOU")
    v_count = sum(1 for c in text if c in vowels)
    c_count = sum(1 for c in text if c.isalpha() and c not in vowels)
    return f"Vowels: {v_count} | Consonants: {c_count} | Total letters: {v_count + c_count}"
```

Add this entry to the `TOOLS` list in `tool_definitions.py`:

```python
{
    "name": "count_vowels",
    "description": "Count the number of vowels and consonants in a piece of text.",
    "input_schema": {
        "type": "object",
        "properties": {
            "text": {
                "type": "string",
                "description": "The text to analyze."
            }
        },
        "required": ["text"]
    }
}
```

Add it to `TOOL_FUNCTIONS` in `agent.py`:

```python
"count_vowels": lambda inputs: count_vowels(inputs["text"]),
```

And import it at the top of `agent.py`:

```python
from tools import calculator, get_current_datetime, analyze_text, convert_units, count_vowels
```

Test it:

```bash
python -c "from agent import run_agent; run_agent('How many vowels are in the word Entrepreneurship?')"
```

### Activity 6.2 — Build Your Own Tool (Challenge)

Design and build a tool of your choice. Ideas:

- A **BMI calculator** (inputs: weight in kg, height in meters)
- A **simple password strength checker** (inputs: a password string)
- A **days between dates calculator** (inputs: two dates)
- A **percentage change calculator** (inputs: old value, new value)

Follow these steps:

1. Write the Python function in `tools.py`
2. Add the tool definition in `tool_definitions.py`
3. Register it in `TOOL_FUNCTIONS` in `agent.py`
4. Test it with a natural language goal

### Checkpoint 6

Run this command and make sure your new tool is used:

```bash
python -c "from agent import run_agent; run_agent('Use my new tool to [describe a task your tool can solve]')"
```

---

## Module 7 — Add Memory to the Agent

### Concept

Right now, each call to `run_agent()` starts fresh — the agent has no memory of previous conversations. For more advanced agents, you can maintain a **conversation history** across multiple calls.

### Activity 7.1 — Build a Persistent Chat Agent

Create `chat_agent.py`:

```python
# chat_agent.py — An agent that remembers the conversation

import anthropic
from tools import calculator, get_current_datetime, analyze_text, convert_units
from tool_definitions import TOOLS

client = anthropic.Anthropic()

TOOL_FUNCTIONS = {
    "calculator": lambda inputs: calculator(inputs["expression"]),
    "get_current_datetime": lambda inputs: get_current_datetime(),
    "analyze_text": lambda inputs: analyze_text(inputs["text"]),
    "convert_units": lambda inputs: convert_units(
        inputs["value"], inputs["from_unit"], inputs["to_unit"]
    ),
}

def chat():
    """Interactive chat loop with memory."""
    messages = []  # This persists across turns
    print("\nAI Agent Chat (type 'quit' to exit, 'clear' to reset memory)\n")

    while True:
        user_input = input("You: ").strip()

        if user_input.lower() == "quit":
            print("Goodbye!")
            break
        if user_input.lower() == "clear":
            messages = []
            print("[Memory cleared]\n")
            continue
        if not user_input:
            continue

        # Add user message to history
        messages.append({"role": "user", "content": user_input})

        # Agent loop
        while True:
            response = client.messages.create(
                model="claude-opus-4-5",
                max_tokens=1024,
                tools=TOOLS,
                messages=messages
            )

            if response.stop_reason == "end_turn":
                for block in response.content:
                    if hasattr(block, "text"):
                        print(f"\nAgent: {block.text}\n")
                        # Add agent reply to history
                        messages.append({
                            "role": "assistant",
                            "content": block.text
                        })
                break

            if response.stop_reason == "tool_use":
                messages.append({"role": "assistant", "content": response.content})
                tool_results = []
                for block in response.content:
                    if block.type == "tool_use":
                        result = TOOL_FUNCTIONS.get(
                            block.name,
                            lambda i: "Tool not found"
                        )(block.input)
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result
                        })
                messages.append({"role": "user", "content": tool_results})

if __name__ == "__main__":
    chat()
```

Run it and try a multi-turn conversation:

```bash
python chat_agent.py
```

Try this conversation:

```
You: Convert 5 miles to km
You: Now take that result and calculate how long it would take to walk it at 5 km/h
You: What percentage of a marathon (42.195 km) is that distance?
```

Notice how the agent remembers earlier answers!

### Checkpoint 7

Verify the agent remembers context by asking a follow-up question that only makes sense if it recalls the previous answer. For example: "What was the first number I asked you to convert?"

---

## Module 8 — Final Project

### Your Mission

Build an agent that solves the following real-world problem from start to finish:

> "I am planning a road trip from Mumbai to Pune (150 km). My car gets 12 km per liter and petrol costs ₹106 per liter. Calculate the fuel cost for the round trip. Also analyze this trip description and tell me how many words it has: 'The Mumbai to Pune expressway is one of the most scenic drives in Maharashtra, passing through the Western Ghats with stunning valley views.'"

Create `final_project.py` and run your agent on this goal. Your agent must:

- Use the calculator tool at least twice
- Use the text analysis tool once
- Produce a complete, well-formatted final answer

### Bonus Challenges

If you finish early, try these extensions:

- Add a `currency_converter` tool (use fixed exchange rates, no API needed)
- Add error handling so the agent retries if a tool returns an error
- Add a `step_limit` parameter to `run_agent()` so it stops after N steps
- Print a summary at the end: total steps taken, tools used, time elapsed

---

## What You Have Built

Congratulations! Here is what you created today:

| Component | File | What it does |
|---|---|---|
| Tools | `tools.py` | The actual functions the agent can call |
| Tool descriptions | `tool_definitions.py` | What Claude reads to know how to use tools |
| Agent loop | `agent.py` | The plan → act → observe → repeat cycle |
| Chat agent | `chat_agent.py` | An agent with multi-turn memory |

### The Agent Pattern — In Summary

```
┌─────────────────────────────────────────────────┐
│                   AI AGENT                      │
│                                                 │
│  Goal → [Claude] → tool_use? → [Your Code]     │
│            ↑                        │           │
│            └──── tool_result ───────┘           │
│                                                 │
│           → end_turn? → Final Answer            │
└─────────────────────────────────────────────────┘
```

---

## Key Takeaways

- An AI agent combines language model reasoning with real-world tool execution.
- Claude decides which tool to call and when — your code runs the actual tool.
- The agent loop repeats until Claude has enough information to give a final answer.
- Adding new capabilities is as simple as adding a new function + a description.
- Memory is just a list of messages you pass along with every API call.

---

## Going Further

Here are topics to explore next:

- **Multi-agent systems** — multiple agents working together, each with different roles
- **Web search tools** — connect your agent to live internet data
- **Code execution tools** — let your agent write and run Python code
- **Vector databases** — give your agent long-term memory across sessions
- **LangChain / LlamaIndex** — popular frameworks that implement these patterns for you

---

## Reference — Complete File Structure

```
my_agent/
├── tools.py               # Tool functions (calculator, text analysis, etc.)
├── tool_definitions.py    # Tool descriptions for Claude
├── agent.py               # Core agent loop
├── chat_agent.py          # Interactive chat with memory
├── test_agent.py          # Test tasks
└── final_project.py       # Your final project
```

---

*Workshop by AI Agent Lab | Suitable for beginners with basic Python knowledge*
