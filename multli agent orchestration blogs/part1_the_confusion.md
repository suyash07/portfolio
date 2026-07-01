# Part 1: The Confusion (Why I Started)

*This is Part 1 of an 8-part series on building a multi-agent churn intelligence system from scratch. I document every question I had, every wrong assumption I made, and what I actually learned.*

---

I kept hearing "multi-agent orchestration" everywhere. Engineering blogs, LinkedIn posts, AI newsletters, conference talks. Everyone was saying companies were implementing these systems. A few people I respected were calling it the most important shift in how software gets built.

I had no idea what it actually meant in practice.

Not in the "I need to read a few more articles" way. In the "I genuinely cannot picture what this looks like running on a computer" way.

So I decided to figure it out by building something. This is the story of how that went, what I got wrong, what clicked, and what I ended up building.

---

## My First Question: Can I Just Build a Data Analysis Project?

My first instinct was to take something I already knew, data science, and wrap it in "agents." Build agents that do EDA, feature engineering, modeling, and dashboard. Call it multi-agent orchestration. Done.

So I started sketching something like this:

```
Orchestrator
   │
   ├── Data ingestion agent
   ├── Analysis agent  
   ├── Visualization agent
   └── Dashboard agent
```

That seemed reasonable. But I immediately hit a question I could not answer: why would you bother wrapping these into agents instead of just writing a script? What does the agent part actually add?

I did not have a good answer. So I kept going anyway.

---

## Picking a Project

I needed a concrete use case. I considered five options: a financial report generator, a stock market analyzer, a customer churn pipeline, a multi-source data aggregator, and a sentiment dashboard.

I landed on customer churn. Specifically, bank customer churn using a dataset of 10,000 customers, each with a binary label: did they leave the bank or did they stay?

The reason this worked as a learning project: the agent roles are genuinely distinct. Loading data, running EDA, training a model, and generating insights are meaningfully different jobs. It is not artificial to separate them.

---

## The First Thing I Got Wrong: Confusing Two Completely Different Things

While researching, I found something called the Ultralight Orchestration gist. A set of four AI agents, an Orchestrator, a Planner, a Coder, and a Designer, that live inside VS Code and write code for you. You describe what you want to build, the Orchestrator delegates to the other agents, and files appear in your project.

My first reaction: is this what I should be building?

Sitting with it for a while, I realized these are two completely different things that share the same name.

**Thing 1: An application made of agents.**

This is a Python program that runs on its own. The agents are specialized components inside the application. When you run it, they execute in sequence, passing data to each other.

```
python orchestrator.py
        │
        ▼
IngestionAgent    loads and cleans 10,000 customers
        │
        ▼
EDAAgent          computes churn rates by segment
        │
        ▼
ModelingAgent     trains a classifier, generates predictions
        │
        ▼
InsightAgent      writes plain-English findings
        │
        ▼
DashboardAgent    opens a Streamlit dashboard
```

These agents are part of the product. They live in your `.py` files permanently. Every time you run the program, they all run again.

**Thing 2: AI agents that build your project.**

These are AI assistants in your editor. Their job is to write code for you. You chat with them, they produce `.py` files, and then they go idle. They are not part of your application. They are scaffolding.

The analogy that made it click: Thing 2 is the construction crew. Thing 1 is the house they build. The house operates on its own after the crew leaves. You can use a crew to build the house faster, but what you end up with is a house, not a crew.

---

## The Question That Reframed Everything

At some point I was explaining my churn pipeline idea to someone and they pushed back. The feedback went something like this:

Building agents to run a model pipeline might not be the most interesting first application of multi-agent orchestration. The more compelling use case is agents that work together to solve a customer problem, or an agentic interface that customers or analysts can actually talk to.

That landed hard.

Because they were right. A churn pipeline has a fixed sequence. Load data, clean data, run EDA, train model, show dashboard. The steps do not change based on what the data says. A regular Python script handles this fine. Wrapping it in agents does not make it meaningfully better.

The use cases where multi-agent orchestration genuinely earns its complexity are different:

The input is ambiguous. A customer types a question. An analyst asks something nobody anticipated. The question changes every time.

The steps are not fixed in advance. What you do next depends on what you just learned. If the model accuracy is low, try a different algorithm. If a customer segment looks unusual, dig deeper before moving on.

The output needs to be personalized. A list of high-risk customers is useful. A personalized explanation of why each specific customer is at risk, with a tailored retention suggestion, is far more useful.

Multiple specialized capabilities need to combine dynamically. Not one agent doing everything. Several agents, each owning one skill, coordinating on the fly to answer a question nobody pre-programmed.

---

## What MCP Is and Why It Matters

Understanding the better use case raised a new question: how does a reasoning agent actually call my model or my database? How does it connect to the real world?

The answer is MCP, the Model Context Protocol. Anthropic created it.

Before MCP, connecting an AI agent to an external tool required custom integration code every single time. Every connection was a one-off. It did not scale.

MCP standardizes these connections. Think of it as USB for AI agents.

```
Without MCP:
   AI agent → custom code → your churn model
   AI agent → custom code → customer database
   AI agent → custom code → CRM system
   (every connection built from scratch)

With MCP:
   AI agent → MCP → your churn model
   AI agent → MCP → customer database  
   AI agent → MCP → CRM system
   (one standard protocol, plug anything in)
```

Your churn model, your EDA functions, your database queries become MCP tools. Any LLM agent can call them through the standard protocol without you writing custom glue code each time.

---

## How This Is Different From RAG

Another thing that confused me early: how is this different from RAG?

RAG stands for Retrieval Augmented Generation. It is a pattern for answering questions about documents the LLM was not trained on.

```
User asks: "What does our Q3 risk policy say about loan defaults?"

RAG pipeline:
  1. Convert question to a vector embedding
  2. Search document store for relevant chunks
  3. Retrieve top matching passages
  4. Pass them into the LLM context
  5. LLM generates answer grounded in retrieved text
```

RAG is about finding relevant text and using it to answer a question. Retrieval is the core mechanism.

A multi-agent system is different:

```
User asks: "Which customers should we call this week 
            and what should we say to each one?"

Multi-agent pipeline:
  1. Reasoning agent understands the question
  2. Decides it needs churn predictions, profiles, and offers
  3. Calls tool: run_churn_model()
  4. Calls tool: get_customer_profiles()
  5. Calls tool: get_retention_offers()
  6. Synthesizes a personalized recommendation per customer
```

The core mechanism here is tool use and coordination, not retrieval. The agent decides what to look up, calls the right tools, and synthesizes the results.

They are not mutually exclusive. In a real production system, a RAG tool might be one of several tools a reasoning agent can call. But they solve different problems. RAG answers "what does the document say?" Multi-agent answers "what should we do?"

---

## Where I Landed

After all of this, the project I decided to build has two phases.

**Phase 1: The tool layer.**

Five Python agents (ingestion, EDA, modeling, insight, dashboard) wired together by an orchestrator. This is not really multi-agent orchestration yet. It is a well-structured modular pipeline with one LLM-powered summarization step. But it is the foundation. Every function you build here becomes a tool the reasoning agent calls in phase 2.

**Phase 2: The intelligence layer.**

A LangGraph orchestrator that makes decisions based on what each agent returns. A conversational reasoning agent that answers analyst and customer questions in plain English by calling the phase 1 functions as tools via MCP.

The phase 1 pipeline is the tool layer. Phase 2 is what turns it into an actual agentic system.

That is the architecture. That is why I built it in this order. And that is what the rest of this series walks through.

---

## The Honest Level Spectrum

Before moving to Part 2, here is an honest framing of what "multi-agent orchestration" looks like at different levels. This spectrum took me a while to understand and nobody writes it down clearly.

```
Level 1: Modular pipeline
  Functions organized as classes, fixed sequence
  What phase 1 is

Level 2: Orchestrated pipeline  
  Coordinator makes decisions, conditional routing, state management
  What LangGraph adds

Level 3: Agentic system
  LLM reasons about what to do next, calls tools dynamically,
  responds to open-ended questions, handles novel situations
  What the full phase 2 becomes

Level 4: Autonomous agent network
  Multiple LLM agents with memory, planning, tool use,
  feedback loops, human-in-the-loop escalation
  What FAANG teams build over months
```

By the end of this series you will have a working Level 3 system, with a clear understanding of what Level 4 requires and why it is genuinely hard.

That is the honest version of what you are about to read.

---

*Part 2 covers the full system architecture before a single line of code. If you want to follow along, the full codebase is at github.com/suyash07/multi_agent_orchestration and my portfolio is at suyash07.github.io/portfolio.*
