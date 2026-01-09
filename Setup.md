# Building a Local Token Optimization System with Claude Code

Claude Code is ideal for this because it can generate and iterate the infrastructure you need. The process involves building three linked components: a local storage system, a retrieval MCP server, and a summarization workflow. You run these locally, then interact with Claude through them, drastically cutting context size.

## Phase 1: Set Up Local Storage Architecture

Start with a simple file-based system. You'll build this out, but the foundation is straightforward JSON records organized by namespace.

**Directory structure:**

```
local-memory/
├── storage/
│   ├── zettelkasten/        # Interconnected notes with tags
│   ├── summaries/           # Auto-generated conversation summaries
│   ├── workflows/           # Completed task records
│   └── metadata/            # Index files for quick retrieval
├── mcp-server.js            # MCP server (communicates with Claude)
├── retriever.js             # Retrieval logic
├── summarizer.js            # Summarization functions
└── config.json              # Configuration and policies
```

Use Claude Code to generate the initial structure. Give it this prompt:

"Build a local token optimization system. Create a directory structure with Node.js files for: (1) a file-based storage layer using JSON, (2) an MCP server that responds to retrieval queries, (3) a retrieval module that finds relevant records by tag/keyword, and (4) a configuration file. Include TypeScript types for storage records."

Claude Code will generate working files. This takes 10 minutes and handles the boilerplate.

## Phase 2: Implement Zettelkasten Storage

This is where your structured memory lives. Each record is a lightweight JSON object with tags, timestamps, and summaries rather than raw data.

**Record structure:**

```json
{
  "id": "proj_2024_q4_analysis_001",
  "type": "workflow_summary",
  "source": "customer_cohort_analysis",
  "created": "2024-11-15T14:30:00Z",
  "tags": ["customer_retention", "q4_2024", "cohort_analysis", "churn_risk"],
  "summary": "Analyzed 5,200 customers from Q3. Retention rate 87%. High-value segment (>$5k LTV) shows 94% retention. Low-touch email cadence outperforms weekly emails by 12 percentage points.",
  "key_entities": ["retention_rate", "ltv_segmentation", "email_frequency"],
  "related_ids": ["proj_2024_q3_analysis_001", "workflow_email_test_002"],
  "token_estimate": 52,
  "source_token_cost": 3200
}
```

The critical line: `token_estimate: 52`. You're trading 3,200 tokens (the full analysis) for 52 tokens (the summary). You keep both but only load the summary into Claude's context by default.

**Claude Code prompt:**

"Create a storage module in Node.js that can write, read, and query these Zettelkasten records. The module should support: (1) writing new records with auto-generated IDs, (2) reading by ID or tag filters, (3) full-text search within summaries, and (4) returning only metadata for large result sets. Each record has a 'token_estimate' field. Include a function that returns the cheapest (smallest token) matching records first."

This generates your `storage.js`. It handles the actual file I/O and indexing.

## Phase 3: Build the MCP Server

Claude needs to query this storage system. The MCP server sits between Claude and your local storage, handling requests like "retrieve customer retention insights from Q4 2024" and returning just the relevant summaries.

**Example MCP request (what Claude sends):**

```json
{
  "method": "retrieve",
  "tags": ["customer_retention", "q4_2024"],
  "limit": 5,
  "max_tokens": 500
}
```

**Example response:**

```json
{
  "records": [
    {
      "id": "proj_2024_q4_analysis_001",
      "summary": "Analyzed 5,200 customers from Q3. Retention rate 87%...",
      "token_estimate": 52
    }
  ],
  "total_tokens_retrieved": 143,
  "alternative_available": false
}
```

Claude Code prompt:

"Build an MCP server using the Anthropic SDK that implements these methods: (1) 'retrieve' - queries local storage by tags and returns lightweight summaries, (2) 'store' - adds new records to Zettelkasten, (3) 'summarize_and_store' - takes raw text, summarizes it, and stores the summary, (4) 'list_available' - shows all tags and record counts. The server should run locally on port 3000 and handle concurrent requests. Use TypeScript."

This generates your actual MCP integration. Claude Code builds the server scaffolding and error handling.

## Phase 4: Implement Smart Retrieval

Retrieval logic determines what gets pulled from storage for any given request. This is where you save tokens.

Simple retrieval strategy:

1. **User submits request to Claude.** Example: "Analyze Q4 customer retention trends and suggest email cadence changes."
    
2. **Claude calls your MCP retrieve method** with tags like `["customer_retention", "q4_2024", "email_strategy"]`.
    
3. **Your retriever finds matching records**, ranks by relevance score (based on tag overlap and recency), and returns the top N summaries that fit within a token budget (say, 300 tokens max).
    
4. **Claude gets the summaries + the user request**, totaling ~500 tokens instead of ~3,000 if you'd loaded raw analysis.
    

More sophisticated approach (token-aware retrieval):

```javascript
const retrieved = storage.query({
  tags: ["customer_retention", "q4_2024"],
  maxTokens: 300,
  strategy: "highest_relevance"  // Other options: "most_recent", "lowest_cost"
});
```

Claude Code prompt:

"Create a retrieval module that: (1) accepts tag-based queries, (2) ranks records by tag overlap and recency, (3) respects a max token budget, (4) returns records ranked by relevance score, (5) logs retrieval queries and their token savings. Include a 'smart_retrieve' function that uses embeddings (you can use a lightweight local embedding model) to match semantic intent when exact tags don't exist."

This is the leverage point. Good retrieval means Claude gets exactly what it needs and nothing it doesn't.

## Phase 5: Set Up Automatic Summarization

As you interact with Claude over multiple turns, summaries should be generated and stored automatically. This prevents context bloat in long workflows.

Summarization trigger points:

1. **Decision checkpoints**: When Claude completes an analysis or reaches a major conclusion, auto-summarize the work and store it.
2. **Token threshold**: When the active conversation hits 1,500 tokens, summarize the last N turns and offload them to storage.
3. **Manual trigger**: You explicitly tell Claude "save current insights" and it summarizes and stores.

**Workflow example:**

Turn 1: User asks Claude to analyze Q4 data (500 tokens). Turn 2: Claude requests clarification (200 tokens). Turn 3: User provides data (800 tokens). Turn 4: Claude produces preliminary findings (600 tokens).

At this point, turns 1-3 become stale. Before Turn 5, invoke the summarizer:

```javascript
const summary = await summarize({
  turns: [turn1, turn2, turn3],
  maxTokens: 80
});

await storage.store({
  type: "conversation_summary",
  source: "q4_analysis_session",
  summary: summary,
  key_insights: ["q4_data_loaded", "preliminary_findings_generated"],
  tags: ["q4_analysis", "active_session"]
});
```

Claude Code prompt:

"Build a summarization module that: (1) takes an array of conversation turns, (2) extracts key insights, entities, and decisions, (3) calls Claude with a short prompt to summarize those turns into 60-80 tokens, (4) returns a summary object with extracted entities and key facts. The summary should be regenerable from the original turns but must be significantly smaller."

This is a recursive use of Claude (Claude summarizing its own work), which feels meta but works. You trade a small cost (one summary call) for much larger token savings downstream.

## Phase 6: Integrate with Your Workflow

Now you need a command-line interface or wrapper that coordinates everything. When you use Claude Code, you'll reference your MCP:

```bash
node mcp-server.js &  # Start the MCP server locally
```

Then, in your interactions with Claude Code, you configure it to use this MCP. The setup command in Claude Code:

```
tools:
  - type: mcp
    name: local-memory
    command: node
    args: [mcp-server.js]
```

Every time you ask Claude Code to work on a task, it can call your MCP to retrieve relevant context. Instead of pasting 5,000 tokens of past work, Claude Code retrieves a 150-token summary.

**Example workflow:**

```
User: "Claude Code, build on the customer cohort analysis from Q4 and add demographic breakdowns."

Claude Code queries MCP: retrieve tag=["customer_retention", "q4_2024"]

MCP returns: 
  - Q4 retention analysis summary (52 tokens)
  - Cohort methodology note (28 tokens)
  - Previous demographic attempt summary (35 tokens)

Claude Code gets user request + 115 tokens of retrieved context, then adds demographic logic.
Total: ~800 tokens instead of 3,500.
```

## Phase 7: Token Accounting and Optimization

Build a logging layer that tracks token savings. Every retrieval logs:

```json
{
  "timestamp": "2024-11-20T10:30:00Z",
  "query_tags": ["customer_retention", "q4_2024"],
  "records_retrieved": 3,
  "tokens_in_summary": 115,
  "equivalent_raw_tokens": 2800,
  "tokens_saved": 2685,
  "retrieval_time_ms": 45
}
```

Over time, you see exactly where you're saving tokens and which retrieval strategies work best. Adjust tag hierarchies and summarization aggressive if you're still bloated.

Claude Code prompt:

"Add a logging module that tracks every retrieval and summarization operation. Log should include: (1) tags queried, (2) records returned, (3) token estimate for retrieval, (4) token estimate for equivalent raw data, (5) savings, (6) retrieval latency. Export summaries weekly showing top savings opportunities and slowest retrievals."

## Making It Work

Start small. Build Phase 1-2 (storage + Zettelkasten) first using Claude Code. Get comfortable with the file structure and record format. Add the MCP server (Phase 3) next. Then plug in retrieval (Phase 4).

Don't try to build the whole system at once. Claude Code is iterative by design. Build, test, refine.

The payoff: once your MCP is running, Claude Code (and your Claude interactions generally) drop from 3,000-5,000 token requests to 800-1,500. For long-running projects or frequent interactions, that's 60-75% cost reduction.

The constraint that makes this feasible: you're only storing summaries and metadata, not raw data. Storage stays compact (a year of summaries might be 50-100MB). Retrieval stays fast (local, no network overhead). Claude gets smarter context, not just more context.
