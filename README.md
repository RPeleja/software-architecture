# Understanding Reranker Scores

Azure AI Search Semantic Ranker provides scores on a 0-4 scale:

- **3.5-4.0**: Excellent match - highly relevant results
- **2.5-3.5**: Good match - relevant results with minor gaps
- **1.5-2.5**: Fair match - partially relevant, may need refinement
- **0.0-1.5**: Poor match - low relevance, definitely needs refinement

**Recommended threshold**: `2.5` (default) balances quality and performance.


# MCP (Model Context Protocol) Integration

## Overview
MCP in this solution is a **real-time context augmentation layer** that extends the RAG (Retrieval-Augmented Generation) system by connecting to external data sources through the Model Context Protocol SDK. It runs in parallel with Azure AI Search to provide fresh, live data alongside static knowledge base results.

## Architecture Flow

1. **Query Refinement** → The user question is refined first
2. **Parallel Retrieval** → Two simultaneous queries:
   - `query_knowledge_base`: Queries Azure AI Search (static indexed docs)
   - `query_mcp_sources`: Queries MCP servers (real-time data)
3. **Merge** → `merge_knowledge_sources` combines both result sets
4. **Rerank & Response** → Results are reranked and used for generation

## How MCP Query Works

**Connection Process:**
- Uses the official `mcp` Python SDK with `StdioServerParameters`
- Spawns MCP server processes via stdio (standard input/output)
- Establishes `ClientSession` for bidirectional communication
- Each server runs as a separate process with environment variables injected

**Query Execution:**
```
User Query → MCP Server Tool Discovery → Tool Selection → Search → File Fetching → Result Formatting
```

**Server Configuration Example:**
```yaml
mcp_servers:
  - name: Microsoft PromptFlow Repository
    command: /usr/local/bin/github-mcp-server
    args: [stdio]
    default_repo: microsoft/promptflow
```

## Key Features

**1. Tool Discovery & Prioritization**
- After connecting, lists all available tools on the MCP server
- For GitHub MCP: prioritizes `search_code` > `search_issues` > `search_repositories`
- Automatically selects the best search tool available

**2. Enhanced Code Search**
- When using `search_code`, fetches **full file contents**
- Makes secondary call to `get_file_contents` tool
- Returns complete file text instead of just snippets

**3. Parallel Querying**
- Supports multiple MCP servers queried simultaneously via `ThreadPoolExecutor`
- Configurable: `mcp_parallel_search: true` enables parallel mode
- Improves response time when querying multiple sources

**4. Graceful Degradation**
- `mcp_fallback_enabled: true` allows system to continue if MCP fails
- Returns empty results instead of crashing
- Logs errors but doesn't block the RAG pipeline

### Result Merging Strategy

**Priority System:**
- MCP results get **priority weight of 2**
- Azure Search gets **priority weight of 1**
- MCP results appear **first** in merged results (higher priority = real-time data)

**Deduplication:**
- Uses **cosine similarity** on text embeddings
- Threshold: `0.95` similarity = duplicate
- Removes redundant results from both sources
- Preserves higher-priority results (MCP over Azure Search)

**Result Limiting:**
- Per server: `mcp_result_limit: 5` results max
- After merge: `max_merged_results: 15` total documents
- Prevents context overflow while including diverse sources

## Configuration Parameters

```yaml
use_mcp: true                      # Enable/disable MCP
mcp_result_limit: 5                # Max results per server
mcp_timeout_seconds: 15            # Query timeout
mcp_fallback_enabled: true         # Continue on MCP failure
mcp_priority: 2                    # Priority weight (vs Azure Search: 1)
max_merged_results: 15             # Total results after merge
deduplicate_results: true          # Enable deduplication
dedup_similarity_threshold: 0.95   # Similarity threshold for duplicates
```

## Result Format
MCP results are normalized to match Azure Search structure:
```python
{
    "text": "<content>",
    "metadata": {
        "mcp_server": "server_name",
        "mcp_tool": "tool_name",
        "github_item": {...}  # Original GitHub data
    },
    "source": "<URL>",
    "sourceType": "mcp",
    "title": "<title>"
}
```

## Real-World Example Flow

When a user asks: "How does promptflow handle retries?"

1. **Refined Query**: "promptflow retry handling"
2. **MCP Query**: Searches `microsoft/promptflow` repo using GitHub MCP's `search_code`
3. **Results**: Fetches actual Python files containing retry logic
4. **Merge**: Combines with Azure indexed docs, MCP results ranked higher
5. **LLM Response**: Generated using both static docs + live code examples

**Key Advantage:** The RAG system gets **up-to-date code and documentation** directly from GitHub repositories, ensuring responses reflect the current state of the codebase rather than outdated indexed content.
