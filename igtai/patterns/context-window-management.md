
# Context window management

## The problem

AI models impose strict token limits on each request, known as context windows, which constrain the combined size of the system prompt, conversation history, and new user input. When applications engage in multi-turn conversations or accumulate long interaction histories, token counts grow continuously until they exceed the model's maximum limit, causing request failures and disrupting user experience.

Unlike traditional stateless APIs where each request stands alone, AI conversations are inherently stateful. Applications must track and send conversation history with each request to maintain coherence across turns. A customer support chatbot might start with a simple 200-token exchange, but after ten back-and-forth interactions, the accumulated context can reach 5,000 tokens. Without active management, the eleventh turn fails when it breaches the model's context window, terminating the conversation abruptly.

Different models have vastly different context limits—ranging from 4,000 tokens for older models to 200,000 tokens for modern large-context models—creating additional complexity. Applications must either hardcode assumptions about token limits, implement custom truncation logic scattered across codebases, or risk failures when context inevitably overflows. The problem intensifies in multi-tenant systems where thousands of concurrent conversations each accumulate context independently, making manual oversight impossible.

## Forces

Several competing tensions make context window management challenging:

- **Model diversity and varying limits**: Context windows range from 4,096 tokens in legacy models to 200,000 tokens in frontier models, and applications often switch between models dynamically. Hardcoding limits couples applications to specific models and breaks when routing changes.

- **Conversation continuity needs**: Users expect coherent multi-turn conversations where the model remembers previous context. Aggressive pruning saves tokens but breaks conversational flow when the model forgets earlier discussion points, frustrating users.

- **Token cost implications**: Larger contexts consume more input tokens per request, directly increasing costs. A 10-turn conversation with full history might repeat thousands of tokens across requests, while intelligent pruning could reduce costs by 60-80% without quality loss.

- **Relevance decay over time**: Early conversation turns often contain information that becomes irrelevant as discussions progress. Initial pleasantries or off-topic remarks consume tokens without adding value to current context, but determining what remains relevant requires semantic understanding.

- **System prompt preservation**: System prompts that define model behavior and instructions must persist throughout conversations, but they consume a fixed portion of the context window regardless of conversation length. Complex system prompts can use 1,000+ tokens before user messages even begin.

- **Unpredictable user input length**: Applications cannot predict how much context the next user message will consume. A sudden long message can push total tokens over the limit, requiring reactive rather than proactive management.

- **Multi-modal context complexity**: Models that process images, audio, or other media consume context tokens in ways that differ from text. A single image might represent 1,000+ tokens, making mixed-media conversations particularly prone to overflow.

- **Latency from large contexts**: Beyond cost, large contexts increase processing time. Models take longer to process 50,000-token requests than 5,000-token requests, impacting user experience even when within limits.

## Risk the pattern mitigates

This pattern primarily mitigates **lack of resilience** from the [IGT-AI operational and performance risks](../risks.md#lack-of-resilience) category. Without context window management, applications experience request failures when accumulated conversation history exceeds model token limits, disrupting user sessions and degrading reliability.

The pattern also significantly reduces **runaway costs** from the [financial risks](../risks.md#runaway-costs) category. Unmanaged context windows cause applications to repeatedly send thousands of tokens of conversation history with every request, multiplying token consumption and costs unnecessarily. A 20-turn conversation without pruning can consume 10 times more input tokens than intelligently managed context.

## How it works

Context window management operates transparently within the AI gateway, tracking token consumption across conversation turns and applying pruning strategies before requests reach model limits. The implementation involves several coordinated mechanisms:

### Token accounting and tracking

The gateway maintains accurate token counts for all conversation elements:

- **Cumulative counting**: Tracks total tokens across system prompts, conversation history, and new user messages. Token counts use model-specific tokenizers since different models tokenize text differently.

- **Per-message accounting**: Records token consumption for each conversation turn, enabling granular decisions about which messages to retain or prune.

- **Headroom calculation**: Monitors remaining capacity by comparing current token count against the target model's context window limit, maintaining a safety buffer to prevent unexpected overflows from response generation.

- **Multi-conversation isolation**: In multi-tenant systems, tracks token counts independently for each conversation session, preventing cross-conversation interference.

### Threshold-based triggering

The gateway activates pruning when token counts approach dangerous levels:

- **Soft thresholds**: Triggers non-disruptive optimization at 70-80% of context limit, applying gentle pruning strategies that remove clearly irrelevant content while preserving conversation flow.

- **Hard thresholds**: Enforces aggressive truncation at 90-95% of context limit, ensuring enough headroom for model responses regardless of user input length.

- **Model-specific limits**: Adjusts thresholds based on the target model's context window, handling model switches automatically when dynamic routing changes which model processes requests.

### Pruning strategies

The gateway implements multiple approaches for reducing context while preserving coherence:

**Sliding window retention** keeps only the most recent N conversation turns, discarding older messages. This simple strategy works well for conversations where recent context dominates relevance. For example, keeping the last 10 turns ensures immediate context remains available while automatically dropping distant history.

**Summarization-based compression** uses a small, fast model to generate concise summaries of older conversation segments. For instance, 20 early conversation turns consuming 3,000 tokens might be summarized into a 300-token digest: "User asked about product features, preferred cloud deployment, and requested pricing for 100 users." This preserves essential information while reclaiming context space.

**Semantic importance ranking** analyzes conversation turns for relevance to current topics and prunes low-relevance messages first. Messages containing key decisions, user preferences, or essential context receive higher scores and persist longer, while small talk and confirmations are pruned early.

**Role-based pruning** preserves system prompts and key instructional messages while pruning user-assistant exchanges. System prompts that define model behavior must remain intact, while intermediate conversation turns can be reduced.

**Hybrid approaches** combine multiple strategies for optimal results. A typical implementation might preserve the 5 most recent turns, summarize turns 6-20, and discard turns older than 20, creating a tiered pruning strategy that balances recency, relevance, and context preservation.

### System prompt preservation

Context window management always preserves system prompts that define model behavior, instruction templates, and operational guidelines. These essential instructions occupy context tokens but must not be pruned, as removing them would degrade response quality or violate application requirements.

When context limits are extremely tight, the gateway can:

- **Compress system prompts** by removing redundant instructions or consolidating verbose phrasing while maintaining semantic meaning
- **Split system prompts** into essential core instructions that always persist and optional guidance that can be pruned under pressure
- **Warn administrators** when system prompts consume excessive context space, indicating a need for optimization

### Prevention of overflow failures

Before forwarding requests to AI providers, the gateway performs final validation:

- **Pre-flight token checks** ensure the complete request including system prompt, pruned history, and new user message fits within model limits with buffer space for responses
- **Emergency truncation** applies last-resort cutting if user input alone exceeds remaining capacity
- **Graceful degradation** returns informative error messages to applications when context cannot be safely managed, enabling fallback behaviors rather than cryptic provider errors

### Transparent operation and state management

Applications continue using standard AI API patterns without implementing token counting or pruning logic:

- **Stateful session management**: The gateway tracks conversation state across requests, associating messages with session IDs to maintain coherent context
- **Client-side transparency**: Applications send full conversation history or reference session IDs, and the gateway applies pruning transparently before provider forwarding
- **Metadata exposure**: Response headers indicate token consumption, pruning actions taken, and remaining context capacity, enabling applications to monitor context health
- **Manual overrides**: Applications can provide hints about message importance or pruning preferences, though the gateway handles defaults automatically

### Cost optimization through context reduction

Beyond preventing failures, context window management delivers substantial cost savings:

```
Example: 20-turn conversation without management
Request 1: 100 tokens → Cost: $X
Request 10: 2,000 tokens → Cost: $20X
Request 20: 5,000 tokens → Cost: $50X
Total input cost: ~500X

Same conversation with sliding window (keep last 10 turns)
Request 1: 100 tokens → Cost: $X
Request 10: 2,000 tokens → Cost: $20X
Request 20: 2,500 tokens → Cost: $25X
Total input cost: ~250X

Cost reduction: 50%
```

By removing irrelevant historical context, the gateway reduces token consumption per request, lowering cumulative costs across long conversations while maintaining quality.

## Solution checklist links

Context window management is implemented through AI gateway capabilities described in the [AI Gateway Definition](../ai-gateway-definition.md#context-window-management).

Implementation checklist items:

- Deploy model-specific tokenizers for accurate token counting
- Configure context window limits for each supported model
- Set soft and hard threshold percentages for pruning triggers
- Implement sliding window retention with configurable turn limits
- Integrate summarization models for conversation compression
- Establish semantic importance scoring for intelligent pruning
- Configure system prompt preservation rules
- Set up session state management for multi-turn conversations
- Enable token usage monitoring and alerting
- Implement pre-flight validation before provider forwarding
- Configure metadata exposure in response headers
- Define emergency truncation fallback behaviors
- Establish cost tracking for context optimization impact

## Contraindications

Context window management is not appropriate in these scenarios:

**Full conversation history is legally required**: Compliance regulations or audit requirements may mandate complete, unmodified conversation records. Industries like healthcare or finance often prohibit any alteration or summarization of user interactions. In these cases, context management conflicts with retention obligations, and applications must either use models with extremely large context windows or implement external storage with selective context loading.

**Absolute determinism is mandatory**: Pruning introduces variability in what context the model receives, making exact response reproduction difficult. If applications require bit-perfect reproducibility—such as scientific research or debugging specific model behaviors—context management adds unwanted non-determinism. Fixed, complete conversation histories are necessary for reproducible outputs.

**Conversation structure is critical**: Some applications analyze or process complete conversation graphs where message relationships and sequencing carry semantic meaning. Pruning or summarization loses structural information that might be essential. For example, analyzing negotiation patterns or conversational dynamics requires intact message sequences.

**Context windows are extremely large**: When using models with 200,000+ token context windows for typical conversations that rarely exceed 20,000 tokens, the overhead of context management provides minimal benefit. Applications naturally operating well below context limits can avoid the complexity of active management.

**Real-time summarization latency is unacceptable**: Compression strategies that use summarization models add processing time to each request. Applications with strict single-digit millisecond latency budgets may not tolerate the overhead, preferring simple truncation or no management at all.

**Cost is not a constraint**: Organizations with unlimited budgets for AI consumption might prioritize maximum context retention over cost optimization, preferring to send complete histories regardless of token consumption.

**Single-turn interactions only**: Applications that never maintain conversation state—such as one-off classification tasks, independent document processing, or stateless question answering—have no context accumulation problem to solve. Context window management addresses multi-turn statefulness.

## References

- [Context window management in AI Gateway definition](../ai-gateway-definition.md#context-window-management) - Technical details on implementation approaches
- [Lack of resilience risk](../risks.md#lack-of-resilience) - Operational risks mitigated by preventing context overflow failures
- [Runaway costs risk](../risks.md#runaway-costs) - Financial risks reduced through context optimization
- [Token-aware rate limiting pattern](token-aware-rate-limiting.md) - Complementary pattern for managing token consumption budgets
- [Dynamic model selection pattern](dynamic-model-selection.md) - Related pattern that considers context window size when routing to models
- [OpenAI tokenizer documentation](https://platform.openai.com/docs/guides/text-generation/managing-tokens) - Model-specific token counting guidance
- [Anthropic prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) - Provider-level context optimization approach
