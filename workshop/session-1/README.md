# Session 1: Basic CLI Client

## Objective

Build a command line application that connects to the AISHE web server, sends a question from the user, and prints the answer.

## Prerequisites

Before starting this session, ensure you have:

1. **AISHE Server Running**: The AISHE server must be running on `http://localhost:8000`
   ```bash
   # From the project root directory
   docker-compose up -d aishe
   ```

## Implementation Overview

### Command-Line Interface

- Accepts questions in the command line

### API Communication


- **Endpoint**: `POST http://localhost:8000/api/v1/ask`
- **Request Format**:
  ```json
  {
    "question": "Your question here"
  }
  ```
- **Response Format**:
  ```json
  {
    "answer": "The generated answer",
    "sources": [
      {
        "number": 1,
        "title": "Wikipedia Article Title",
        "url": "https://en.wikipedia.org/wiki/..."
      }
    ],
    "processing_time": 2.45
  }
  ```

You can test the AISHE API directly using curl:

```bash
curl -X POST http://localhost:8000/api/v1/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the capital of France?"}'
```

---

## STRETCH TASK: modern AISHE architecture

Today, AISHE always requests 3 Wikipedia articles before Ollama summarizes them and responds.

Modern AI agents have access to tools. If an agent decides it needs extra information,
it calls an MCP server on its own. The information is fed back into the LLM in a loop.

The goal of this task is to adjust AISHE's server implementation to support tool-calling.
Pass Wikipedia tools to Ollama and let it decide what info it fetches via the Wikipedia MCP.

You may find AISHE's server implementation inside [/src/](/src/).

```
         ┌─────────────┐
         │    User     │
         │  question   │
         └──────┬──────┘
                │
                ▼
    ┌───────────────────────┐
    │       AISHE           │
    │  (agent / server)     │◀──────────────┐
    └───────────┬───────────┘               │
                │                           │
                │  prompt + available       │  tool result
                │  tools (wiki search,      │  (article text)
                │  wiki fetch, ...)         │
                ▼                           │
        ┌───────────────┐                   │
        │    Ollama     │                   │
        │     LLM       │                   │
        └───────┬───────┘                   │
                │                           │
     ┌──────────┴──────────┐                │
     │                     │                │
  has enough           needs more           │
  info?                info? (tool call)    │
     │                     │                │
     ▼                     ▼                │
┌─────────────┐     ┌─────────────┐         │
│   Final     │     │ Wikipedia   │         │
│   answer    │     │    MCP      │         │
│  + sources  │     │   server    │         │
└──────┬──────┘     └──────┬──────┘         │
       │                   │                │
       │                   └────────────────┘
       ▼
┌─────────────┐
│    User     │
└─────────────┘
```

### Before proceeding

You need a working session 1 client before proceeding. Complete the regular task
first, then revisit this one.

You also need to deploy your own local AISHE server via Docker or Nix:

- For Docker deployment, see the [prerequisites section above](#prerequisites)
- For Nix deployment, see the [repository README.md](/README.md)

This task involves modifying AISHE's server codebase. It's best to get familiar
with it before making changes.

If you're not entirely familiar with the concept of tool calling, check out
[this brief introduction](https://medium.com/@yasir_siddique/tool-calling-for-llms-a-detailed-tutorial-a2b4d78633e2).

Feel free to use AI agents to fulfill this task. Any LLM (gpt, claude, composer, grok, etc.) will work.

### How to use AI to implement this task

Below is a simple workflow you can use to implement this task in 40-50 minutes,
even if you're unfamiliar with AI concepts.

Pay attention to the *development process* itself:
- We start by gathering needed context (information related to our topic).
- Then we let AI narrow down our analysis into a targeted implementation.

Using your favorite AI agent:

1. **Ask** an AI agent to explain unknown concepts and AISHE's codebase to you:

> What is a RAG pipeline and how is it used in AISHE?

2. Have it make a **plan** to implement Wikipedia tool calls:

> Come up with a step-by-step plan to give tools to Ollama so it can decide itself what information to look up

3. Prompt AI agent to **implement** the produced plan:

> Okay, now that we've refined the plan, implement it inside my codebase.
> Then give me instructions on how to verify it works.

4. **Debug** any issues:

```bash
# Prompt AISHE and inspect Ollama & server logs
# Assert Ollama triggers an MCP request
./your-cli "What is the capital of France?"
```

> Hey, when I ask AISHE about the capital of France, it responds with Paris but doesn't output Wikipedia sources?
> Where did we make a mistake and how can we fix it?
