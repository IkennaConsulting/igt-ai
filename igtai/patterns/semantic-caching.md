
# Semantic caching

## The problem

Every time an application sends a prompt to an AI API, it incurs costs based on token consumption for both input and output. In production systems, many user queries are semantically similar or identical in meaning despite variations in phrasing. Traditional HTTP caching mechanisms depend on exact byte-for-byte matches of requests, which fails for natural language where users express the same intent in countless different ways. For example, "What is the capital of France?", "Tell me the capital city of France", and "Which city is France's capital?" all request identical information but would be treated as cache misses by conventional caching strategies.

Without semantic awareness, organizations pay full API costs for every variation of conceptually identical queries, leading to unnecessary expenses and increased latency. As AI usage scales across applications, this inefficiency compounds rapidly, particularly for common queries like frequently asked questions, standard customer support requests, or repetitive analytical questions.

## Forces

Several competing concerns make this problem challenging to solve:

- **Natural language variation**: Users express the same intent in countless ways using different vocabulary, sentence structures, word orders, and levels of formality. A caching solution must recognize semantic equivalence across these variations.

- **Similarity thresholds**: Determining when two prompts are "similar enough" requires balancing precision and recall. Too strict a threshold misses valid cache hits and wastes money. Too loose a threshold returns inappropriate cached responses and degrades quality.

- **Context sensitivity**: Some queries that appear similar may have subtle but important differences in context, intent, or required specificity. The caching system must avoid conflating genuinely distinct requests.

- **Cache invalidation complexity**: Unlike static content, AI responses may become outdated as underlying knowledge bases change, model versions update, or business context evolves. Determining appropriate cache lifetimes is non-trivial.

- **Computational overhead**: Computing semantic similarity requires generating vector embeddings for each prompt, which itself consumes resources. The cost of similarity computation must be significantly lower than the cost of the AI API call to justify the approach.

- **Freshness requirements**: Some use cases require unique or time-sensitive responses where caching is inappropriate regardless of semantic similarity. Others benefit from consistent answers to similar questions.

- **Variable response quality**: Cached responses bypass potential model improvements or updated knowledge, creating a trade-off between cost savings and response quality.

## Risk the pattern mitigates

This pattern primarily mitigates **[Runaway costs](../risks.md#runaway-costs)** from the IGT-AI financial risks category. Without proper controls, repetitive AI API consumption leads to significant and unexpected expenses as users naturally rephrase queries while requesting the same information repeatedly.

By recognizing semantically equivalent queries and serving cached responses instead of making new API calls, semantic caching directly reduces token consumption and associated costs. This is particularly important for high-traffic applications with predictable query patterns, such as customer support chatbots, internal knowledge assistants, or public-facing FAQ systems where certain questions dominate usage.

## How it works

Semantic caching extends traditional HTTP caching with natural language understanding to recognize when different prompts request conceptually similar information. The implementation requires several key components:

### Vector embedding generation

When a prompt arrives, the gateway generates a vector embedding—a high-dimensional numerical representation that captures the semantic meaning of the text. Similar meanings produce similar vectors regardless of specific word choices. For example:

- "What is the capital of France?" → [0.21, -0.45, 0.78, ...]
- "Tell me France's capital city" → [0.19, -0.43, 0.81, ...]
- "Best pizza in New York?" → [-0.67, 0.32, -0.15, ...]

The first two embeddings would be geometrically close in vector space, while the third would be distant, enabling similarity detection.

### Similarity threshold matching

The gateway compares the incoming prompt's embedding against cached embeddings using distance metrics like cosine similarity or Euclidean distance. If a cached embedding exceeds a configured similarity threshold (commonly 0.85-0.95 on a 0-1 scale), it's considered a cache hit.

```
Incoming: "Which city is the capital of France?"
Embedding similarity to cache:
  - "What is the capital of France?" → 0.94 (HIT)
  - "Population of Paris?" → 0.67 (MISS)
  - "Best hotels in Lyon?" → 0.31 (MISS)
```

### Cache storage and retrieval

Unlike traditional HTTP caches keyed by URL and headers, semantic caches use vector databases or specialized storage systems optimized for similarity search. The cache stores:

- **Vector embeddings** of prompts
- **Original prompt text** for transparency
- **Cached AI response** to return on cache hits
- **Metadata** including timestamp, TTL, hit count, and model version
- **Cache key parameters** such as model, temperature, and system context

### Contextual scoping

Semantic caching respects prompt context and parameters that affect responses. Cache keys incorporate:

- **Model identity**: Responses from GPT-4 and Claude are cached separately
- **System prompts**: Different system instructions produce different responses
- **Temperature and sampling parameters**: Creative vs. deterministic settings
- **User/tenant isolation**: Multi-tenant systems prevent cross-tenant cache sharing
- **Temporal context**: Time-sensitive queries may include date in cache key

### Cache invalidation strategies

Several approaches manage cache freshness:

- **Time-to-live (TTL)**: Cached responses expire after a configured duration (e.g., 1 hour for dynamic content, 24 hours for stable facts)
- **Model version tracking**: Cache invalidation when underlying models update
- **Manual purging**: Administrative controls to clear cache by pattern or tag
- **LRU eviction**: Least-recently-used entries removed when cache reaches capacity
- **Feedback-based invalidation**: User corrections or negative feedback trigger cache removal

### Transparent operation

Applications continue using standard AI API interfaces without modification. The gateway intercepts requests, checks the semantic cache, and either returns cached responses or forwards to the AI provider and caches the result for future use. From the application's perspective, the operation is identical but faster and cheaper for cache hits.

### Cost-benefit analysis

The pattern achieves cost savings when:
```
(cost of embedding generation + cost of similarity search) < (cost of AI API call)
```

For example:
- Embedding generation: ~$0.0001 per prompt
- Similarity search: ~$0.0001 per query
- AI API call: $0.01-$0.10 per prompt/response
- **Net savings per cache hit: ~99%**

The break-even point occurs at very low cache hit rates (~1-2%), making semantic caching economically favorable for most workloads.

## Solution checklist links

Implementation options include:

- **[AI Gateway solutions](../ai-gateway-definition.md#semantic-caching)**: Reference the semantic caching section for technical details on gateway implementation
- **Embedding model selection**: Choose appropriate embedding models (OpenAI Ada, Cohere Embed, open-source Sentence Transformers)
- **Vector database integration**: Implement storage using Pinecone, Weaviate, Milvus, Qdrant, or Redis with vector extensions
- **Similarity threshold tuning**: Conduct A/B testing to optimize threshold values for your use case
- **Cache analytics**: Monitor cache hit rates, cost savings, and response quality metrics

## Contraindications

Semantic caching is not appropriate when:

- **Uniqueness is required**: Each response must be novel or creative, such as generating unique marketing copy, creative writing, or personalized content where repetition is undesirable.

- **Real-time freshness is critical**: Responses depend on current data that changes frequently, such as stock prices, weather, breaking news, or time-sensitive information where even minutes-old cache entries are unacceptable.

- **Subtle distinctions matter**: Queries that appear semantically similar require meaningfully different responses based on nuanced differences. For example, medical or legal queries where small phrasing variations indicate different scenarios.

- **Streaming responses are essential**: Applications requiring token-by-token streaming may not benefit from cached responses, though some implementations support streaming cached content.

- **Privacy or compliance restrictions**: Regulatory requirements prohibit caching certain types of responses, or cross-user cache sharing violates data isolation policies.

- **Low query repetition**: If queries are genuinely unique with minimal semantic overlap, cache hit rates will be too low to justify the embedding and storage overhead.

- **Determinism is mandatory**: When reproducibility requires calling the live model with identical parameters every time, caching introduces unwanted variability in the execution path.

## References

- [Semantic caching in AI Gateway definition](../ai-gateway-definition.md#semantic-caching) - Technical details on implementation approaches
- [Runaway costs risk](../risks.md#runaway-costs) - Financial risks mitigated by this pattern
- [Token-aware rate limiting pattern](token-aware-rate-limiting.md) - Complementary cost control pattern
- [GPTCache: Semantic cache for LLMs](https://github.com/zilliztech/GPTCache) - Open-source implementation reference
- [Anthropic prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) - Provider-level caching approach
- [Vector databases for semantic search](https://www.pinecone.io/learn/vector-database/) - Storage infrastructure concepts
