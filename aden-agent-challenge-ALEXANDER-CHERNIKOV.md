# Build Your First Agent Challenge - Alexander Chernikov

## Part 1: Agent Fundamentals

### Task 1.1: Core Concepts

1. A node is a self-contained, executable unit in Aden's agent graph that can be scheduled, observed, and connected to other nodes at runtime. Unlike a traditional function, a node is stateful at the agent level, participates in routing, and exposes metadata for tooling, memory, and safety.

2. SDK-wrapped node concept: every node automatically gets:
   - Structured input/output schema validation
   - Built-in memory access (shared, STM, LTM)
   - Tool execution and tracing hooks
   - Observability and error-handling lifecycle events (retries, fallbacks, logging)

3. Differences:
   - Coding Agent vs Worker Agent: Coding Agent modifies system logic or graphs; Worker Agent executes a specific task within the graph.
   - Goal-driven vs workflow-driven: goal-driven focuses on final outcomes with dynamic routing; workflow-driven follows predefined steps.
   - Predefined edges vs dynamic connections: predefined edges are static links; dynamic connections are created at runtime based on context or goals.

4. Aden generates connection code to enable dynamic, context-aware routing and to optimize execution paths based on runtime data, rather than forcing a fixed graph that may be inefficient or too rigid for real-world variability.

### Task 1.2: Memory Systems

1. Three types of memory:
   - Shared Memory: global, cross-agent knowledge accessible by all agents in a system.
   - STM (Short-Term Memory): per-session working memory for the current task and recent context.
   - LTM (Long-Term Memory / RLM): persistent memory for learned patterns and feedback across sessions.

2. Usage:
   - Shared Memory: common facts, brand guidelines, or company context.
   - STM: current job details, transient execution context, intermediate results.
   - LTM: lessons learned, failure patterns, preferred outputs, historical feedback.

3. Session Local memory isolation means each run has its own STM namespace; data is not leaked across sessions unless explicitly promoted to Shared or LTM, preventing contamination and improving privacy and correctness.

### Task 1.3: Human-in-the-Loop

1. Human intervention is triggered by confidence thresholds, policy violations, brand risk, or explicit workflow checkpoints (e.g., pre-publication).
2. If no response within timeout, the system follows configured fallback behavior: pause, auto-retry, or fail safe by halting publication.
3. Essential HITL scenarios:
   - Legal or compliance-sensitive claims
   - Brand tone or messaging approval
   - Conflicting data sources or ambiguous facts

## Part 2: Agent Design

### Task 2.1: Content Marketing Multi-Agent System

1. Agent diagram (ASCII):

```
News Ingest -> Research Agent -> Writer Agent -> Editor Agent -> Human Approval -> Publisher Agent
                      |                |              |                 |
                      +---> Memory ----+--------------+-----------------+
```

2. Agent descriptions:

- Research Agent
  - Role: gather context and facts for each news item
  - Inputs: news item, keywords
  - Outputs: research summary, source links
  - Tools: web_search, knowledge_base_search
  - Failure: missing sources, low confidence; fallback to request human input

- Writer Agent
  - Role: draft blog post
  - Inputs: news item, research summary, brand guide
  - Outputs: draft post with title, outline, CTA
  - Tools: llm_generate, style_checker
  - Failure: off-brand, incomplete; retry with stricter prompt or escalate

- Editor Agent
  - Role: fact-check and edit for tone
  - Inputs: draft post, sources
  - Outputs: edited post, approval status
  - Tools: fact_check, grammar_check
  - Failure: detected factual error; send back to Research or Writer

- Publisher Agent
  - Role: publish approved content
  - Inputs: approved post, metadata
  - Outputs: live URL, status
  - Tools: cms_publish, image_uploader
  - Failure: API errors; retry with backoff, notify human if repeated

3. Human checkpoints:
   - After Editor Agent approves but before publish
   - When Research Agent cannot verify critical facts
   - When Writer Agent repeatedly fails tone or policy checks

4. Self-improvement:
   - Store rejection feedback and failed outputs in LTM
   - Update prompt templates and style constraints
   - Coding Agent can modify routing rules based on recurring failure types

### Task 2.2: Goal Definition

Goal:
Create a system that monitors our internal news feed, drafts a brand-compliant blog post for each item, runs fact checks and tone editing, gets marketing approval before publishing to WordPress, and records feedback to improve future drafts. Success is defined by publishing accurate posts within 24 hours. If drafting or editing fails, the system should retry with stricter constraints and escalate to a human reviewer.

### Task 2.3: Test Cases

| Test Case | Input | Expected Output | Success Criteria |
| --- | --- | --- | --- |
| Happy Path | Standard news item with sources | Published blog post | Post live and approved |
| Edge Case 1 | News item with minimal details | Draft with clarifying questions | Human approves or provides missing info |
| Edge Case 2 | News item contradicts knowledge base | Escalation to human | No publish until resolved |
| Failure 1 | CMS API timeout | Retry and queue for publish | No data loss; eventual publish or alert |
| Failure 2 | Writer output off-brand | Rewritten draft or human review | Revised draft passes editor |

## Part 3: Practical Implementation

### Task 3.1: Agent Pseudocode

```python
class ContentWriterAgent:
    """
    Agent that takes news items and writes blog posts.
    """

    def __init__(self, config):
        self.llm = config.llm_client
        self.memory = config.memory
        self.tools = config.tools
        self.brand_guide = self.memory.shared.get("brand_guide")

    async def execute(self, input_data):
        news = input_data["news_item"]
        research = input_data["research_summary"]

        # Read short-term context for current run
        stm_context = self.memory.stm.get("run_context", default={})

        prompt = f"""
        You are a content writer for {stm_context.get('company_name', 'our company')}.
        Write a blog post based on this news: {news}
        Use these facts: {research}
        Follow this brand guide: {self.brand_guide}
        Output: title, 3-section outline, body, CTA.
        """

        try:
            draft = await self.llm.generate(prompt)
            self.memory.stm.set("latest_draft", draft)
            return {"draft": draft, "status": "ok"}
        except Exception as error:
            return await self.handle_failure(error, {"news": news, "research": research})

    async def handle_failure(self, error, context):
        # Log failure to LTM with context
        self.memory.ltm.append("writer_failures", {"error": str(error), "context": context})

        # If rate limited, retry with backoff
        if "rate limit" in str(error).lower():
            await self.tools.sleep(seconds=5)
            return await self.execute(context)

        # Otherwise escalate for human review
        return {"status": "failed", "reason": str(error), "needs_human": True}

    async def learn_from_feedback(self, feedback):
        # Store feedback and update prompt heuristics
        self.memory.ltm.append("writer_feedback", feedback)
        self.memory.shared.set("style_notes", feedback.get("style_notes", []))
```

### Task 3.2: Prompt Engineering

SYSTEM PROMPT:
You are a professional content writer for {company_name}. Write accurate, on-brand blog posts. Avoid speculation and include citations when provided.

TASK PROMPT:
Given the following news item:
{news_content}

Context and sources:
{research_summary}

Write a blog post with a clear title, 3-section outline, body copy under 800 words, and a CTA. Follow this brand guide:
{brand_guide}

FEEDBACK PROMPT:
Your previous post was rejected with this feedback:
{feedback}

Identify what failed (tone, accuracy, completeness, structure) and propose a revised draft outline and updated writing rules.

### Task 3.3: Tool Definitions

```python
tools = [
    {
        "name": "knowledge_base_search",
        "description": "Search internal knowledge base for relevant context",
        "parameters": {
            "query": "string - search query",
            "limit": "int - max results (default 5)"
        },
        "returns": "List of documents with title, snippet, url",
        "example": "knowledge_base_search(query='product launch', limit=3)"
    },
    {
        "name": "fact_check",
        "description": "Validate claims against trusted sources",
        "parameters": {
            "claims": "list[string] - statements to verify"
        },
        "returns": "List of claim results with verdict and evidence",
        "example": "fact_check(claims=['We launched X on Jan 5'])"
    },
    {
        "name": "cms_publish",
        "description": "Publish a blog post to the CMS",
        "parameters": {
            "title": "string - post title",
            "body": "string - HTML or markdown content",
            "tags": "list[string] - post tags",
            "schedule_time": "string - ISO timestamp or 'now'"
        },
        "returns": "Publish result with status and URL",
        "example": "cms_publish(title='Launch', body='...', tags=['news'], schedule_time='now')"
    }
]
```

## Part 4: Advanced Challenges

### Task 4.1: Failure Evolution Design

1. Failure classification:
   - LLM failures: rate limit, content filter, hallucination
   - Tool failures: API down, invalid response, timeout
   - Logic failures: wrong output format, missing data
   - Human rejection: off-brand, factual error, unclear CTA

2. Learning storage:
   - LLM: prompt, model, error type, retry count
   - Tool: tool name, inputs, response codes, latency
   - Logic: validation errors, schema mismatches
   - Human: feedback text, reviewer role, revision notes

3. Evolution strategy:
   - Coding Agent aggregates failure stats weekly
   - Update prompt templates and routing rules
   - Adjust model selection and tool retries

4. Guardrails:
   - Keep a stable baseline prompt and A/B changes
   - Roll back if quality drops or error rate increases
   - Enforce validation schemas and human approval gates

### Task 4.2: Cost Optimization

1. Model selection:
   - GPT-4 for final drafts and complex edits
   - GPT-3.5 or Claude Haiku for summaries and triage
2. Caching strategy:
   - Cache research summaries, brand guide, and repeated FAQ snippets
3. Batching:
   - Batch multiple news summaries for research and tagging
4. Budget rules:
   - Daily token budget per agent
   - Escalate to human if budget exceeded
   - Use cheaper model when confidence is high

### Task 4.3: Observability Dashboard

1. Performance metrics:
   - End-to-end latency per post
   - LLM response time
   - Tool call latency
   - Retry rate
   - Queue backlog size

2. Quality metrics:
   - Human approval rate
   - Fact-check pass rate
   - Rewrite count per post

3. Cost metrics:
   - Tokens per post
   - Cost per publish
   - Cache hit rate

4. Alert conditions:
   - Approval rate drops below threshold
   - Repeated tool failures over 15 minutes
   - Cost per post exceeds budget
