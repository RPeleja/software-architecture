# Understanding Reranker Scores

Azure AI Search Semantic Ranker provides scores on a 0-4 scale:

- **3.5-4.0**: Excellent match - highly relevant results
- **2.5-3.5**: Good match - relevant results with minor gaps
- **1.5-2.5**: Fair match - partially relevant, may need refinement
- **0.0-1.5**: Poor match - low relevance, definitely needs refinement

**Recommended threshold**: `2.5` (default) balances quality and performance.
