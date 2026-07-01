# From Confusion to Clarity: My Journey Understanding Multi-Agent Orchestration

*A data scientist's honest questions, wrong assumptions, and the moment it all clicked.*

---

## Why I Started Asking These Questions

I kept hearing the phrase "multi-agent orchestration" everywhere. FAANG engineers, LinkedIn posts, AI newsletters, all talking about how companies are implementing multi-agent systems. I had no idea what it actually meant in practice, so I decided to figure it out by building something.

This post documents every question I had, every wrong assumption I made, and how my understanding evolved until I finally knew what I was actually building and why.

---

## Question 1: Can I build a data analysis project with a dashboard using multi-agent orchestration?

This was my starting question. I had VS Code, Copilot CLI installed, and a vague idea that "multi-agent" and "data analysis" should go together somehow.

The short answer: yes, you can. But the more important answer is that multi-agent orchestration and data analysis are not naturally the same thing. The architecture works well for certain use cases and adds unnecessary complexity for others.

What I initially imagined was something like this:

```
Orchestrator
   │
   ├── Data ingestion agent
   ├── Analysis agent
   ├── Visualization agent
   └── Dashboard agent
```

That seemed reasonable. But I didn't yet understand why you'd bother wrapping these into "agents" instead of just writing a script.

---

## Question 2: What project should I implement?

I considered five options:

1. Automated Financial Report Generator
2. Stock/Crypto Market Analyzer
3. Customer Churn Analysis Pipeline
4. Multi-Source Data Aggregator
5. Survey/Review Sentiment Dashboard

I landed on **Customer Churn Analysis Pipeline**, and specifically the bank customer churn dataset (10,000 customers, binary target: did they leave the bank or not?).

The reason this worked as a learning project: the agent roles are genuinely distinct. Ingestion, EDA, modeling, and insight generation are meaningfully different jobs. It's not artificial to separate them.

---

## Question 3: What data should I use?

I learned about the IBM Telco dataset (the standard beginner choice, 7,043 customers, no Kaggle login needed) and the bank churn dataset (10,000 customers, features like CreditScore, Balance, Geography, NumOfProducts, target is `Exited`).

I went with the bank churn dataset for a simple reason: it maps to the fintech domain. If you're building a portfolio project, domain relevance matters.

Key thing to know about this dataset upfront: only about 20% of customers churned. That class imbalance means you cannot use plain accuracy as a metric. A model that predicts "nobody ever churns" scores 80% accuracy and is completely useless. You need precision, recall, and ROC-AUC.

---

## Question 4: I found this gist. Is this what I should implement?

I found Burke Holland's "Ultralight Orchestration" gist: a set of four AI agents (Orchestrator, Planner, Coder, Designer) that live inside VS Code Copilot and write code for you.

This is where my understanding started splitting in a useful direction.

**The gist is not the same thing as building a multi-agent application.**

They're both called "multi-agent orchestration," but they're operating at completely different levels.

---

## The Realization: Two Completely Different Things Share the Same Name

This was the most important clarification of my entire learning journey.

**Bullet 1: An application made of agents (what you build)**

This is a Python program that runs on its own. The "agents" are specialized components inside your application. When you run it, they execute in sequence, passing data to each other.

```
python orchestrator.py
        │
        ▼
IngestionAgent.run()  →  loads and cleans data
        │
        ▼
EDAAgent.run()        →  computes churn rates by segment
        │
        ▼
ModelingAgent.run()   →  trains classifier, generates predictions
        │
        ▼
InsightAgent.run()    →  (the one LLM agent) writes plain-English findings
        │
        ▼
DashboardAgent.run()  →  Streamlit dashboard opens
```

These agents are part of the product. They live in your `.py` files permanently. Every time you run the program, they all run again.

**Bullet 2: AI agents that build your project (what the gist is)**

These are AI assistants in VS Code. Their job is to write your code for you. You chat with the Orchestrator, it delegates to the Planner and Coder, and a `.py` file appears in your project.

```
You (in VS Code chat): "Build the ingestion component"
        │
        ▼
Orchestrator AI  →  routes to Planner
        │
        ▼
Planner AI       →  creates implementation plan
        │
        ▼
Coder AI         →  writes agents/ingestion.py
        │
        ▼
File appears in your project. AI agents go idle. They are done.
```

These agents are not part of your churn project. They're scaffolding, a construction crew. Once the house is built, they leave.

**The analogy that made it stick:**

Bullet 2 is the construction crew. Bullet 1 is the house they build. The house operates on its own after the crew leaves. You can hire a crew to build a house, or build it yourself, but either way what you end up with is a house, not a crew.

Burke Holland's gist is someone else's bullet 1 project (a set of coding agents) that got polished and shared. Other people now use it as a bullet 2 tool (to build their own things). Your churn pipeline agents could eventually become someone else's reusable ML pipeline toolkit. Same concept, different domains.

---

## Question 5: If we're doing a code approach, why do we need Copilot CLI?

You don't need it. It's a productivity tool. The code is the same either way.

With or without Copilot, you end up with a Python class. Copilot reduces the typing. You're still the one who decides what the agent should do, reviews what it wrote, catches mistakes, and wires the pieces together.

For learning orchestration specifically, writing the first few agents by hand is actually better. The concepts stick when you type them. Use Copilot for the boilerplate-heavy parts once the pattern is clear in your head.

---

## Question 6: Shouldn't agents be rules instead of code?

This is a sharp question because it gets at when you need an LLM versus when you need a function.

The answer is: it depends on the step.

```
Is this step deterministic? Known inputs → known outputs?
        │
        ├── YES → write code
        │         faster, cheaper, reliable, testable, auditable
        │
        └── NO, requires judgment or reasoning → use an LLM with a prompt
                  handles ambiguity, synthesizes meaning, more expensive
```

The gist's agents are prompts because their job is reasoning. The Planner needs to look at a codebase and decide what to build. That requires judgment. A prompt makes sense.

Your churn pipeline agents are mostly code because their job is deterministic. Loading a CSV does not require reasoning. `pd.read_csv()` does it the same way every time, reliably, cheaply, instantly.

**The pipeline breakdown:**

| Agent | Type | Why |
|---|---|---|
| IngestionAgent | Code | Loading and cleaning data is deterministic |
| EDAAgent | Code | Computing groupby stats is deterministic |
| ModelingAgent | Code | Training sklearn models is deterministic |
| InsightAgent | LLM | Turning numbers into plain English requires reasoning |
| DashboardAgent | Code | Rendering Streamlit components is deterministic |

This is not a compromise. This is the correct production pattern. You use an LLM exactly where human-like reasoning is genuinely needed, and nowhere else.

---

## Question 7: If I'm writing Python code, is this even orchestration? Am I just engineering?

This feeling is legitimate and points to something real.

A simple sequential pipeline (call agent 1, pass output to agent 2, repeat) does feel like "just calling functions." And honestly, at that stage, it mostly is.

What makes it real orchestration is when the orchestrator starts making decisions:

```python
# Sequential pipeline (just calling functions)
stats = EDAAgent().run(X, y)
model = ModelingAgent().run(X, y)

# Actual orchestration (coordinator makes decisions)
stats = EDAAgent().run(X, y)

if stats["churn_rate"] > 30:
    stats = EDAAgent().run_deep(X, y)   # escalate to deeper analysis

result = ModelingAgent().run(X, y)

if result["roc_auc"] < 0.75:
    result = ModelingAgent().run(X, y, model="xgboost")   # retry with better model
```

Now the orchestrator is coordinating, not just sequencing. And when you replace the orchestrator with LangGraph, it gets a proper state machine with conditional routing, parallel execution, and retry logic.

The spectrum looks like this:

```
Notebook script  →  Agent classes + sequential orchestrator  →  LangGraph + LLM agents
(what you know)     (where you start)                           (what you're building toward)
```

You start in the middle and build right. That is the correct learning path.

---

## The FAANG Data Science Manager Perspective

At this point I shared what I was building with a data science manager at a FAANG company. His feedback reframed everything:

> "Using agents to build full models might not be the best first thing to do with multi-agent orchestration. You should look into use cases where agents work together to solve a customer problem or provide an agentic interface to customers. It can have prompts and deterministic rules available through tools and MCP."

He was right. My project had the right architecture but the wrong use case emphasis.

Building a churn model has a fixed sequence. The data pipeline doesn't need to reason about what to do next because the steps are always the same. A regular Python script handles this fine. Wrapping it in agents doesn't add much.

The use cases where multi-agent orchestration genuinely earns its complexity are the ones where:

- The input is ambiguous (customer questions, unstructured documents)
- The steps are not fixed in advance (what you do next depends on what you just learned)
- The output needs to be personalized or conversational
- Multiple specialized capabilities need to combine dynamically

---

## What is MCP and Why Does It Matter?

MCP stands for Model Context Protocol. Anthropic created it.

Before MCP, connecting an AI agent to an external tool (a database, an API, your ML model) required custom integration code every single time. Every connection was a one-off. It didn't scale.

MCP standardizes these connections. Think of it as USB for AI agents.

```
Without MCP:
   AI agent ──custom code──► your churn model
   AI agent ──custom code──► customer database
   AI agent ──custom code──► CRM system
   (every connection built from scratch)

With MCP:
   AI agent ──MCP──► your churn model
   AI agent ──MCP──► customer database
   AI agent ──MCP──► CRM system
   (one standard protocol, plug anything in)
```

In the context of the architecture I'm building: your churn model, your EDA functions, and your database queries become MCP tools. Any LLM agent can call them through the standard protocol. The deterministic code becomes a tool the reasoning agent uses when it needs it.

---

## Why the Reasoning Agent Exists: The Problem It Solves

This is the question that connected everything.

After a churn model is built and deployed, here is what actually happens in most companies today:

**The analyst workflow (today, without a reasoning agent):**

1. Analyst opens the dashboard, sees churn rate is 20%
2. Wants to know why German customers churn more
3. Emails the data scientist
4. Waits two days
5. Gets a chart
6. Presents to manager
7. Manager asks a follow-up question
8. Repeat from step 3

The model's intelligence is locked inside a static dashboard that can only answer the questions it was built to answer. Every new question requires a data scientist.

**The customer service workflow (today):**

1. Rep gets a list of "high risk customers" every Monday
2. Calls them with a generic retention script
3. Has no idea why each specific customer is at risk
4. Can't personalize the conversation
5. Customer churns anyway

**With a reasoning agent:**

```
Analyst asks: "Why are German customers churning more 
               and is it getting worse?"

Reasoning agent:
  → calls tool: get_churn_by_geography()
  → calls tool: get_churn_trend(segment="Germany", periods=4)
  → calls tool: get_feature_importance(segment="Germany")
  → answers: "German customers churn 2x more, driven by higher
              proportion of month-to-month contracts. The gap
              widened in Q3. Recommend targeting with contract
              upgrade offers."

Time: 30 seconds. No data scientist involved.
```

```
Rep asks: "I'm about to call customer 4821, what should I know?"

Reasoning agent:
  → calls tool: get_customer_profile(4821)
  → calls tool: run_churn_model(4821)        ← 73% churn probability
  → calls tool: get_churn_drivers(4821)      ← inactive 8 months, 1 product
  → calls tool: get_retention_offers(4821)
  → answers: "73% churn probability. Inactive for 8 months,
              only has a checking account. Lead with the savings
              account offer. Similar profiles retained at 40%."

Rep has a personalized script before the call connects.
```

The model's value was always there. The reasoning agent unlocks it at scale, for every analyst, every rep, every customer, simultaneously.

---

## The Architecture I'm Actually Building

This is what the full system looks like:

```
                    ┌─────────────────────────────┐
                    │      REASONING AGENT         │
                    │  (LLM, decides what to ask)  │
                    └──────────────┬──────────────┘
                                   │ calls tools via MCP
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
  run_churn_model()      get_customer_profile()    get_segment_stats()
  (your ML model)        (database query)          (your EDA functions)
  deterministic code     deterministic code        deterministic code
```

The bottom layer is the tool layer: all the deterministic Python code, the churn model, the EDA functions, the data queries. This is what I'm building this weekend.

The top layer is the reasoning agent: the LLM that decides which tools to call and synthesizes the results into an answer. This is what I'm building next weekend.

MCP is the protocol connecting them.

---

## Build Plan

**Phase 1: The tool layer (this weekend)**

Build five Python agent classes:
- `IngestionAgent` — loads and cleans the bank churn data
- `EDAAgent` — computes churn rates by segment, flags high-risk groups
- `ModelingAgent` — trains a classifier with class-weight balancing, reports ROC-AUC
- `InsightAgent` — calls an LLM, translates model output into plain English
- `DashboardAgent` — Streamlit app with KPI cards, segment charts, feature importance

Wire them in `orchestrator.py` with a simple sequential coordinator first, then swap in LangGraph for real conditional routing.

**Phase 2: The reasoning layer (next weekend)**

Add a conversational agent on top. Expose the churn model and EDA functions as MCP tools. Build a simple interface where an analyst or a rep can ask a question in plain English and get a personalized answer powered by the underlying model.

This is the part that maps directly to what companies are actually building.

---

## Question 8: What is LangGraph and why do I need it?

LangGraph is a framework for building multi-agent systems where the orchestrator makes decisions about what to do next, not just follows a fixed sequence.

The name comes from "graph." Instead of a linear pipeline, you define your agents as nodes in a graph, and edges between them represent possible paths. The orchestrator traverses the graph based on state, not a hardcoded order.

**My current orchestrator (linear, dumb):**

```
Ingestion → EDA → Modeling → Insight → Dashboard
(always the same path, no decisions made)
```

**LangGraph orchestrator (graph, smart):**

```
Ingestion → EDA ──── churn < 20% ──────────────► Modeling
                │                                     │
                └─── churn > 20% ──► Deep EDA ───────┘
                                                      │
                                         ROC-AUC < 0.75?
                                          │           │
                                         YES          NO
                                          │           │
                                       XGBoost    Insight
                                          │           │
                                          └─────────┘
                                               │
                                           Dashboard
```

The orchestrator reads the state after each node and decides which node to go to next. It can loop back, skip steps, run things in parallel, or escalate based on what it finds.

Without LangGraph, my orchestrator is just a script calling functions in order. With LangGraph, it becomes a real coordinator that reasons about what to do next based on what each agent returned.

**Three concrete things LangGraph adds:**

Conditional routing: if model accuracy is low, retry with a different algorithm. If churn rate is unusually high, run deeper segment analysis before modeling.

Parallel execution: run EDA and feature engineering simultaneously instead of sequentially, cutting runtime.

State management: every agent reads from and writes to a shared state object. No manual passing of variables between agents. The orchestrator always knows exactly where things stand.

This is the upgrade that happens in phase 2 of my build. Week one I write the sequential baseline so the concepts are clear. Week two I swap in LangGraph so the orchestrator becomes a genuine decision-maker, not a script.

---

## Question 9: How is this different from RAG? Isn't RAG also built for deterministic results?

This is a sharp question because on the surface both look similar: LLM plus some data source plus an answer comes out. But they solve completely different problems.

**What RAG is:**

RAG stands for Retrieval Augmented Generation. It is a pattern for answering questions about documents the LLM was not trained on.

```
User asks: "What does our Q3 risk policy say about 
            loan defaults over $50k?"

RAG pipeline:
  1. Convert question to a vector embedding
  2. Search document store for relevant chunks
  3. Retrieve top matching passages
  4. Pass them into the LLM context
  5. LLM generates an answer grounded in the retrieved text

Output: an answer grounded in your documents
```

RAG is fundamentally about finding relevant text and using it to answer a question. Retrieval is the core mechanism.

**What my multi-agent system is:**

```
User asks: "Which customers should we call this week 
            and what should we say to each one?"

Multi-agent pipeline:
  1. Reasoning agent understands the question
  2. Decides it needs churn predictions, customer profiles,
     and retention offer data
  3. Calls tool: run_churn_model()     → gets probabilities
  4. Calls tool: get_customer_profiles() → gets demographics
  5. Calls tool: get_retention_offers()  → gets available deals
  6. Synthesizes a personalized recommendation per customer

Output: an action plan grounded in live model outputs
```

My system is about reasoning over live data and tool outputs to decide what to do. Tools are the core mechanism.

**The key differences:**

| | RAG | Multi-Agent System |
|---|---|---|
| Core mechanism | Retrieve text chunks | Call tools, execute code |
| Data source | Documents, PDFs, text | Databases, ML models, APIs |
| Output | An answer from text | An action or decision |
| State | Stateless, one shot | Stateful, agents pass context |
| Routing | Fixed pipeline | Dynamic, conditional |
| Best for | "What does X say about Y?" | "What should we do about Z?" |

**The analogy that made it click:**

RAG is like giving someone a library card and asking them to research a question. They find the relevant books, read the relevant pages, and summarize the answer. The intelligence is in finding and synthesizing the right text.

A multi-agent system is like hiring a team of specialists who can run experiments, pull live reports, query systems, and coordinate with each other to solve a problem. The intelligence is in deciding what to do and coordinating who does it.

**On the "deterministic" point:**

RAG is not deterministic the way a SQL query is. The retrieval step is fuzzy (vector similarity search), and the LLM generation step is probabilistic. The answer can vary between runs. It is more reliable than a pure LLM answer because it is grounded in your documents, but it is not deterministic in any strict sense.

My deterministic code agents (the Python classes) are genuinely deterministic. Same input, same output, every single time. That is a stronger guarantee than RAG gives you, which is exactly why I keep the ML model and data queries as code tools rather than RAG pipelines.

**Where they overlap in practice:**

In a real production system you often use both together. The reasoning agent might call a RAG tool as one of its available options alongside the deterministic tools:

```
Reasoning agent gets question: "Why did customer 4821 churn?"

→ calls tool: run_churn_model(4821)         ← deterministic code
→ calls tool: get_customer_history(4821)     ← database query
→ calls tool: search_support_tickets(4821)   ← RAG over text documents
→ synthesizes all three into one answer
```

The RAG system is just one tool the agent uses. The agent orchestrates it alongside other tools. That is why people conflate them: RAG often lives inside a multi-agent system as one component, but it is not the same thing as the system itself.

The one-line summary: RAG finds relevant text to answer a question. Multi-agent orchestration coordinates specialized components to take action. One is about retrieval and synthesis. The other is about reasoning and coordination.

---

## What I Actually Learned

The word "multi-agent orchestration" gets used for at least two different things that look similar from the outside but are completely different in practice.

One is a set of AI coding assistants that build software for you. The other is a software system where specialized components coordinate to solve a problem. The first is a development tool. The second is a product.

The use cases where the second genuinely earns its complexity are the ones involving ambiguous input, dynamic routing, and conversational interfaces, not fixed data pipelines with known steps.

The right pattern in production: deterministic code for steps with known logic, LLM agents for steps requiring judgment, MCP to connect them, an orchestrator to coordinate. The skill is knowing which is which for each step.

And finally: the value of a churn model has never been the model itself. It's always been the model's ability to answer questions, surface risk, and enable action. The reasoning agent is what finally unlocks that value at scale, without a data scientist in the loop for every question.

That's why companies are building these systems. And that's what I'm building.

---

*Next post: building the tool layer, five agents, one orchestrator, and the first time it all runs end to end.*
