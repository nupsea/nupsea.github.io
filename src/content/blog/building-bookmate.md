---
title: "Building Bookmate: My Journey Creating an Open-Source AI Library Assistant"
description: "A deep dive into building a self-hosted RAG application to converse with your personal book library, covering hybrid search, agents, MCP, and evaluation strategies."
pubDate: "Feb 11 2026"
---

I recently had the opportunity to speak at DataEngBytes about a project close to my heart: **Bookmate**. It's an AI assistant I built to help me "converse" with my personal library.

If you missed the talk, or if you just want to dive into the architecture and the lessons I learned building a full-stack LLM application, this post is for you.

## The "Why": Philosophy, Forgetfulness, and Privacy

The idea for Bookmate was born out of a specific frustration. I spend some of my spare time reading books. Books provide us the joy of learning and imagination.

The problem? **Books are hard. We tend to forget the details.** Months after reading, I might face a difficult situation and remember that Marcus Aurelius had a concept on how to deal with it, but I can't remember where it is or exactly how he phrased it.

While tools like ChatGPT or Google's NotebookLM exist, I wanted something different. I wanted a solution that was:

1. **Open Source**: So I could tweak the code and learn from the implementation.
2. **Self-Hosted**: So I could run it on my laptop without sending my personal data or private library into the cloud.

## The Tech Stack: Moving Beyond Simple Prompts

Bookmate isn't just a wrapper around an API; it's an engineering approach to the limitations of LLMs (hallucinations, frozen knowledge, and context limits). Here is how I architected the solution.

### The Power of Hybrid Search

One of the biggest takeaways from my testing was that neither keyword search nor vector search is enough on its own.

- **Keyword Search** is great for precision (e.g., searching for a specific name like "Cheshire Cat").
- **Vector Search** is excellent for semantic meaning (e.g., "that grinning cat").

For Bookmate, I implemented **Hybrid Search**. By combining both methods, I saw a **15-20% improvement** in hit rates and Mean Reciprocal Rank (MRR) compared to using them individually.

### Agents and the "Brain"

RAG (Retrieval Augmented Generation) gets you the data, but **Agents decide what to do with it**. I designed the Bookmate agent to be dynamic. When a user asks a question, the agent decides:

- Do I search a single book?
- Do I need to compare multiple books (e.g., "Compare suffering in Marcus Aurelius vs. Hegel")?
- **Retry Logic**: If a search yields zero results, the agent can self-correct, rephrase the query, and try again.

### Adopting MCP (Model Context Protocol)

A major headache in agentic workflows is the "M x N" connection problem: every LLM trying to connect to every different data source with custom APIs.

To solve this, I adopted Anthropic's **Model Context Protocol (MCP)**. It standardizes how the LLM communicates with the outside world (databases, tools, APIs). In Bookmate, the "Search Book" function is just a tool defined via MCP that the agent can call when it detects the need.

## The Hardest Part: Evaluation and Observability

Building the app was fun, but optimizing it was the hardest part. Because LLMs are non-deterministic, you can't just run a unit test and call it a day. You might change a prompt and break a feature without knowing it.

To tackle this, I focused heavily on observability:

- **Golden Datasets**: I generated ground truth data to measure "Hit Rate" (did we find the right chunk?) and "MRR" (was the right chunk at the top?).
- **LLM as a Judge**: I used an LLM to score the final answers on a scale of 0-5 based on relevance to the question.
- **Tracing**: I used Phoenix for distributed tracing. It allows me to see exactly what the agent is thinking: the tool calls, the retrieved chunks, and the latency, step by step.

## The Architecture

For those interested in the engineering side, Bookmate follows a clean separation of concerns:

- **Frontend**: A React/Gradio interface for chatting and ingesting books.
- **Ingestion**: When you upload a book, you can define a "Chapter Pattern" (like Roman numerals). The system chunks the text (with overlap) and stores embeddings in Qdrant.
- **Deployment**: The whole thing runs on Docker via a simple `make up` command.

## Future Plans

I'm currently working on expanding Bookmate to handle more formats. A major challenge right now is audiobooks and images: handling technical books with diagrams or audio files where we don't have the text.

## Try It Yourself

Bookmate is open source and available for you to run locally.

- **GitHub**: [github.com/nupsea/book-mate](https://github.com/nupsea/book-mate)
- **Watch the Talk**: You can see the full demo on the [DataEngBytes YouTube channel](https://www.youtube.com/@DataEngBytes).

If you're building your own RAG pipelines, I'd love to hear how you are handling evaluation and search optimization. Feel free to reach out!
