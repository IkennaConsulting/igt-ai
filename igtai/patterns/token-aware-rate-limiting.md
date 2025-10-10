
# Token-aware rate limiting

## The problem

Traditional API rate limiting mechanisms count requests per time window, assuming relatively uniform resource consumption across calls. A typical API gateway might enforce a limit of 1,000 requests per minute, treating each request as roughly equivalent regardless of payload size or processing complexity. This approach breaks down catastrophically for AI APIs where **resource consumption varies by orders of magnitude depending on token count**.

A simple AI query like "What's the capital of France?" might consume 20 input tokens and generate 5 output tokens, costing fractions of a cent. Meanwhile, a single request asking the model to "Summarize this 50-page legal document" could consume 40,000 input tokens and generate 5,000 output tokens, costing several dollars—literally thousands of times more expensive. Traditional request-based rate limiting treats these identically, allowing a handful of expensive requests to exhaust monthly budgets while reporting usage well within request limits.

Organizations face several critical challenges with token-based pricing models. A malicious user or misconfigured application can submit verbose prompts or request maximum-length responses, depleting budgets in hours rather than days. Legitimate users working with long documents or complex reasoning tasks consume dramatically more resources than those asking simple questions, creating fairness issues when request-based quotas apply uniformly. Cost attribution becomes impossible when rate limiting operates at the request level while billing operates at the token level, preventing finance teams from enforcing department budgets or implementing chargeback models.

Without token-aware controls, organizations experience **denial-of-wallet attacks** where attackers or careless users exhaust API quotas without triggering request-based rate limits. A single expensive prompt repeated 100 times might represent only 100 requests but consume an entire month's token allocation. Traditional monitoring systems show "low" request volumes while bills skyrocket, leaving teams blindsided by unexpected costs discovered only when invoices arrive.

## Forces

Several competing concerns make token-aware rate limiting challenging to implement effectively:

**Token counting complexity**: Accurate token counting requires model-specific tokenizers since different models (GPT-4, Claude, Llama) tokenize identical text differently. The text "Hello, world!" might be 4 tokens in one model and 3 in another. Organizations must maintain multiple tokenizers, keep them synchronized with provider updates, and apply the correct tokenizer based on which model will process each request. This complexity multiplies when supporting dozens of models across multiple providers.

**Input and output token asymmetry**: AI costs typically charge different rates for input tokens (prompt) and output tokens (generated response). Input tokens might cost $0.01 per 1,000 tokens while output tokens cost $0.03 per 1,000 tokens—a 3x difference. Rate limiting must account for both, but output token counts are unknown until after generation completes. This creates a chicken-and-egg problem: the system cannot accurately enforce output token limits without generating the response first, but generating a response that exceeds limits wastes resources.

**Streaming response challenges**: When models stream responses token-by-token, output token counts accumulate gradually rather than being known upfront. A streaming request might be within limits for the first 1,000 output tokens but exceed quotas at token 1,001. The gateway must decide whether to terminate mid-stream (disrupting user experience) or allow completion (violating limits). Neither choice is ideal, requiring organizations to balance strict enforcement against user satisfaction.

**Multi-dimensional quota management**: Organizations need quotas at multiple scopes—per user, per application, per team, per project, per model, and organization-wide. A single request might count against five different quota buckets simultaneously. The gateway must track and enforce all applicable limits, failing requests when any quota is exceeded. Managing these overlapping quotas without race conditions or double-counting requires sophisticated state management and coordination.

**Time window complexity**: Token consumption patterns are spiky and irregular. Unlike request-based limits that naturally smooth out over time windows, token usage can exhibit extreme variance—a quiet hour followed by several expensive document processing requests. Organizations need flexible time window definitions (per second, per minute, per hour, per day, per month) with different limits at each tier. A user might have 1,000 tokens per second, 100,000 per hour, and 5 million per month, requiring simultaneous enforcement across all windows.

**Cost vs. fairness trade-offs**: Simple token budgets disadvantage users working on legitimately complex tasks. A researcher analyzing long scientific papers naturally consumes more tokens than someone asking simple factual questions. Pure token-based limits penalize complex legitimate work, while unlimited access enables abuse. Organizations must balance fairness (allowing necessary token consumption) with cost control (preventing abuse and runaway spending).

**Budget enforcement timing**: Organizations typically operate on monthly billing cycles with fixed budgets, but token consumption happens continuously in real-time. The gateway must track cumulative monthly consumption across potentially millions of requests while maintaining accurate counts that align with provider billing. Achieving consistency between gateway-measured consumption and provider-reported usage requires careful token counting, provider reconciliation, and handling of edge cases like partial responses or failed requests.

**Graceful degradation needs**: When users approach token limits, hard rejections create poor experiences. Users working on important tasks suddenly encounter failures with cryptic "quota exceeded" errors, losing work and context. Better approaches provide warnings, suggest cost-saving alternatives (smaller models, summarization), or allow temporary overages with notifications. But these softer limits complicate enforcement logic and create opportunities for abuse.

## Risk the pattern mitigates

This pattern directly mitigates two critical risks from the [IGT-AI financial risks category](../risks.md):

**[Runaway costs](../risks.md#runaway-costs)**: Without proper monitoring and controls, the usage of AI APIs can lead to significant and unexpected expenses. Traditional request-based rate limiting provides no protection against token-based billing, allowing applications or users to submit expensive prompts that exhaust budgets while remaining within request quotas. Token-aware rate limiting enforces budgets at the granularity where costs actually accumulate—the token level—preventing uncontrolled spending by capping consumption before it reaches critical levels.

**[Denial-of-wallet attacks](../risks.md#denial-of-wallet-attacks)**: Attackers can exploit AI APIs by sending a high volume of expensive requests, leading to unexpected costs and potential service disruptions. By crafting verbose prompts or requesting maximum-length responses, malicious actors can exhaust organizational token budgets in hours, effectively mounting a financial denial-of-service attack. Token-aware rate limiting detects and blocks abnormally expensive requests based on token consumption patterns, protecting organizations from both intentional attacks and accidental misconfigurations.

The pattern also indirectly supports **[poor observability](../risks.md#poor-observability)** mitigation by providing visibility into token consumption patterns across users, applications, and models. This enables cost attribution, identifies optimization opportunities, and supports financial forecasting.

## How it works

Token-aware rate limiting operates at the AI gateway layer, tracking token consumption across multiple dimensions and enforcing quotas before requests reach AI providers. The implementation requires several coordinated components that work together to provide accurate, fair, and efficient token-based access control.

### Token counting and estimation

Before enforcing limits, the gateway must accurately count tokens consumed by each request. This involves multiple stages:

**Input token counting**: When a request arrives, the gateway uses model-specific tokenizers to count input tokens precisely. The tokenizer must match the target model's encoding scheme—GPT-4 uses a different tokenizer than Claude or Llama. The gateway maintains a library of tokenizers and selects the appropriate one based on model selection logic. For requests containing system prompts, conversation history, and new user input, the gateway counts all components to determine total input token consumption.

**Output token estimation**: Since output tokens are generated incrementally and their count is unknown at request time, the gateway must estimate expected output consumption. Several approaches enable this:

- **Maximum output limit parameter**: Most AI APIs allow specifying `max_tokens` to cap output length. The gateway can use this as a conservative estimate, assuming the worst case where models generate the maximum allowed tokens.
- **Historical patterns**: By tracking typical output token counts for similar requests, the gateway builds statistical models that predict output consumption based on input characteristics, request type, and model selection.
- **Sliding window calculations**: The gateway maintains per-user or per-application averages of recent output token consumption, using these averages as estimates for upcoming requests.
- **Safety margins**: To prevent quota violations, estimates include safety buffers, adding 10-20% to predicted output token counts.

**Output token reconciliation**: After response generation completes, the gateway counts actual output tokens and reconciles them against estimates. If actual consumption exceeded estimates, the gateway updates quota tracking retroactively, potentially placing users temporarily over limits until consumption normalizes. This eventual consistency model balances strict enforcement with operational practicality.

### Multi-dimensional quota tracking

The gateway maintains token consumption counters across multiple dimensions simultaneously:

**Per-user quotas**: Individual users receive token budgets that track their personal consumption. This enables fair distribution across teams and prevents single users from monopolizing resources. User identification comes from authentication tokens, API keys, or OAuth 2.0 claims.

**Per-application quotas**: Different applications or services receive independent token budgets. A customer-facing chatbot might have a large allocation while internal experimentation tools have smaller limits. This prevents low-priority applications from consuming budgets needed by critical production services.

**Per-team or organization quotas**: In multi-tenant systems, different business units, departments, or customer organizations receive isolated budgets. Marketing, engineering, and finance teams each have separate allocations that don't interfere with each other. This enables organizational cost management and chargeback models.

**Per-model quotas**: Organizations may allocate different budgets for expensive vs. economical models. Premium models like GPT-4 or Claude Opus have tighter limits while cheaper models like GPT-3.5 or Claude Haiku allow higher consumption. This encourages cost-conscious model selection while permitting premium model access for high-value use cases.

**Global organization quotas**: An overall organizational limit caps total token consumption across all users, applications, teams, and models. This provides ultimate protection against budget overruns regardless of how consumption is distributed internally.

When evaluating a request, the gateway checks all applicable quotas. If any quota is exceeded, the request is rejected with a detailed error message indicating which limit triggered the rejection. This multi-dimensional enforcement ensures comprehensive cost control without complex logic in client applications.

### Time window management

Token quotas operate over various time windows, each serving different control objectives:

**Per-second limits**: Prevent sudden spikes or burst attacks by capping instantaneous consumption. For example, 10,000 tokens per second prevents a single user from overwhelming shared resources with a flood of expensive requests.

**Per-minute limits**: Smooth out short-term variability while allowing legitimate bursts. A user might legitimately submit several expensive requests in quick succession as part of a workflow, but sustained high consumption over minutes indicates potential issues.

**Per-hour limits**: Support workload patterns where usage concentrates in specific hours (business hours, batch processing windows) while preventing sustained abuse throughout the day.

**Per-day limits**: Align with typical business cycles and allow teams to front-load usage early in the day while ensuring budgets last through the entire day.

**Per-month limits**: Match billing cycles and organizational budgets. Finance teams allocate monthly budgets to departments, and monthly token limits enforce these allocations. This is the most common quota type for financial control.

The gateway maintains separate counters for each time window and enforces all of them simultaneously. A request is allowed only if it stays within **all applicable limits**. For example, a user might be under their monthly limit but exceed their per-second limit due to a burst of requests, triggering rejection even though monthly budget remains available.

Time windows use **sliding window algorithms** rather than fixed boundaries to prevent quota reset gaming. Instead of resetting counters at midnight, sliding windows track consumption over rolling time periods. This ensures smooth, continuous enforcement without sudden resets that attackers could exploit.

### Request evaluation and enforcement workflow

When a request arrives, the gateway follows a systematic evaluation process:

1. **Authentication and identification**: Extract user identity, application identity, team/organization affiliation, and other metadata needed to determine which quotas apply.

2. **Model selection and tokenization**: Determine which model will process the request (based on routing logic or explicit specification) and select the appropriate tokenizer for accurate counting.

3. **Input token counting**: Tokenize the complete input including system prompts, conversation history, and new user content, producing an exact input token count.

4. **Output token estimation**: Predict expected output token consumption based on maximum limits, historical patterns, or conservative estimates.

5. **Total consumption calculation**: Sum input tokens and estimated output tokens to determine total expected consumption for the request.

6. **Quota checking**: Query all applicable quota counters (per-user, per-app, per-team, per-model, organization-wide) across all relevant time windows (per-second, per-minute, per-hour, per-day, per-month).

7. **Limit enforcement decision**:
   - If **all quotas** have sufficient remaining capacity: Allow the request, decrement all quota counters by the estimated consumption, and forward to the AI provider.
   - If **any quota** is exceeded: Reject the request immediately with a detailed error response indicating which quota was exceeded, current consumption, quota limit, and reset time.

8. **Response processing and reconciliation**: After the AI provider returns a response, count actual output tokens. Update quota counters with actual consumption, adjusting for any differences between estimated and actual token counts.

9. **Async reconciliation**: For quotas that were based on estimates, the gateway may retrospectively adjust counters. If actual consumption significantly exceeded estimates, the gateway may temporarily lock the user's access until consumption falls back within limits.

### Graceful degradation strategies

Hard quota rejections create poor user experiences, particularly for legitimate users approaching limits due to complex but necessary work. The gateway can implement softer enforcement approaches:

**Warning thresholds**: When users consume 70-80% of their quotas, the gateway adds warning headers to responses indicating approaching limits, remaining capacity, and suggestions for reducing consumption. Applications can display these warnings to users proactively.

**Overflow buffers**: Allow temporary quota overages up to a small percentage (e.g., 10% over limit) to handle estimation errors and prevent disruption for users genuinely close to limits. Overages trigger notifications but don't immediately block access, allowing users to finish critical work while administrators address the situation.

**Priority tiers**: Classify requests by priority (critical customer-facing vs. internal experimentation) and enforce stricter limits for low-priority traffic while allowing critical requests to continue even when approaching limits. This ensures production services remain operational while reducing less important consumption.

**Dynamic model downgrading**: When users approach token limits, the gateway can automatically route requests to cheaper models rather than rejecting them outright. For instance, downgrade from GPT-4 to GPT-3.5 or from Claude Opus to Claude Haiku, preserving functionality while reducing per-request costs.

**Rate limit buyback**: In enterprise scenarios, allow applications to request temporary quota increases through approval workflows. This provides escape valves for legitimate exceptional circumstances while maintaining default controls.

### Cost attribution and visibility

Beyond enforcement, token-aware rate limiting provides comprehensive cost visibility:

**Real-time dashboards**: Display current token consumption, remaining quotas, consumption trends, and cost projections across all dimensions (users, applications, teams, models). Finance and engineering teams gain insight into where token budgets are spent.

**Consumption analytics**: Break down token usage by prompt type, model selection, time of day, and request characteristics. Identify which use cases drive costs, which users are heavy consumers, and where optimization opportunities exist.

**Budget forecasting**: Project when quotas will be exhausted based on current consumption trends, providing early warnings that enable proactive quota adjustments or usage optimization before limits are hit.

**Chargeback support**: Generate detailed reports attributing token consumption and costs to specific users, teams, or applications. Finance teams use these reports for internal billing, department budget management, and cost accountability.

**Anomaly detection**: Identify unusual consumption patterns that might indicate misconfiguration, abuse, or unintended behavior. Sudden spikes, sustained increases, or consumption from unexpected sources trigger alerts for investigation.

### Provider reconciliation

Since providers charge based on their own token counting, the gateway must ensure its consumption tracking aligns with provider billing:

**Provider token reporting**: Most AI providers include token counts in API responses (input tokens, output tokens, total tokens). The gateway captures these provider-reported counts and compares them to its own calculations.

**Discrepancy handling**: When gateway counts and provider counts differ (due to tokenizer version mismatches, edge cases in encoding, or calculation errors), the gateway flags discrepancies for investigation. Persistent discrepancies trigger tokenizer updates or algorithm adjustments to maintain accuracy.

**Billing reconciliation**: At the end of billing periods, the gateway compares cumulative token consumption tracked internally against provider invoices. Any mismatches indicate bugs in counting logic or provider errors, enabling correction before costs surprise finance teams.

**Multi-provider aggregation**: Organizations using multiple AI providers (OpenAI, Anthropic, Google, Azure) receive separate invoices with different billing formats. The gateway normalizes and aggregates consumption across providers, presenting unified token tracking and cost attribution regardless of which providers served requests.

## Solution checklist links

Implementation guidance for token-aware rate limiting can be found in:

- **[AI Gateway Definition](../ai-gateway-definition.md#token-aware-rate-limiting)**: Technical specifications for token-based rate limiting, budget enforcement, and circuit breakers
- **[Token-aware rate limiting implementation checklist](../checklists/token-aware-rate-limiting.md)**: Detailed evaluation criteria for assessing AI gateway token limiting capabilities

Organizations implementing token-aware rate limiting should consider:

**Tokenizer management**:
- Deploy tokenizers for all supported models (OpenAI tiktoken, Anthropic, Cohere, HuggingFace)
- Keep tokenizers synchronized with provider updates
- Implement fallback tokenizers for unsupported models
- Cache tokenization results for repeated prompts

**Quota configuration**:
- Define per-user, per-application, per-team, and organization-wide token budgets
- Configure time windows (per-second, per-minute, per-hour, per-day, per-month)
- Set warning thresholds at 70-80% of limits
- Establish overflow buffers for graceful degradation
- Define priority tiers for critical vs. experimental traffic

**Storage and state management**:
- Select high-performance distributed counters (Redis, Memcached, DynamoDB)
- Implement counter persistence to survive gateway restarts
- Design for horizontal scalability as request volumes grow
- Handle clock skew in distributed environments

**Monitoring and alerting**:
- Track token consumption in real-time across all dimensions
- Alert when quotas approach exhaustion (80%, 90%, 95%)
- Monitor discrepancies between gateway counts and provider reports
- Visualize consumption trends and forecasts

**User communication**:
- Return detailed error messages indicating which quota was exceeded
- Include remaining capacity and reset times in responses
- Provide API endpoints for applications to query current quota status
- Offer self-service interfaces for users to monitor their consumption

## Contraindications

Token-aware rate limiting is not appropriate in several scenarios:

**Early prototypes and research environments**: When developers are exploring AI capabilities and iterating rapidly on prompts, strict token limits introduce friction that hinders experimentation. Research environments benefit from permissive or unlimited access with comprehensive logging rather than hard enforcement, allowing creativity while maintaining visibility into consumption patterns.

**Dedicated single-purpose deployments**: If an organization dedicates an AI budget exclusively to one application or use case with no cost-sharing concerns, complex multi-dimensional token limiting provides minimal value over simpler provider-level billing. The administrative overhead of configuring and maintaining token quotas exceeds the benefit when only one consumer exists.

**Trusted internal low-volume systems**: Internal applications used by small, trusted teams processing low volumes of requests may not justify token rate limiting complexity. If total consumption naturally remains well below budgets and abuse risk is negligible, simpler request-based limits or no limits at all reduce operational complexity.

**Flat-rate or unlimited pricing arrangements**: Organizations with negotiated flat-rate pricing or unlimited usage contracts with AI providers don't face direct financial risk from token overconsumption. While token limiting might still provide fairness or capacity management benefits, the primary cost control motivation disappears under flat-rate pricing.

**Use cases requiring maximum context**: Applications that routinely process long documents, extensive conversation histories, or large knowledge bases naturally consume many tokens per request. Aggressive token limiting forces these legitimate use cases to artificially constrain context, degrading quality. For document analysis, legal review, or research applications, generous token budgets aligned with actual workload needs are more appropriate than restrictive limits.

**Real-time collaborative editing or streaming**: Applications where multiple users collaborate on shared AI-generated content or where token consumption is unpredictable due to streaming behavior may struggle with strict per-request limits. These scenarios benefit more from aggregate monthly quotas that allow flexibility in how tokens are consumed across individual requests.

Alternative approaches for these scenarios include:

- **Monitoring without enforcement**: Track token consumption for visibility and reporting while not actively blocking requests, relying on organizational policies and user trust rather than technical controls
- **Aggregate quotas only**: Enforce only monthly or organization-wide limits, skipping per-user or per-application granularity to reduce complexity
- **Alert-based governance**: Send notifications when consumption exceeds thresholds but allow continued usage, delegating enforcement to manual review processes
- **Provider-level limits**: Rely on rate limits and budget alerts provided by AI providers themselves rather than implementing gateway-level controls

## References

**AI-specific rate limiting research**:
- [Token-Based Resource Management for LLM APIs](https://arxiv.org/abs/2305.10762) - Research on efficient token quota algorithms
- [Cost-Aware Scheduling for Large Language Models](https://arxiv.org/abs/2308.12715) - Strategies for managing variable-cost AI workloads
- [Fair Resource Allocation in Multi-Tenant AI Systems](https://arxiv.org/abs/2310.15348) - Fairness considerations for token-based rate limiting

**Rate limiting algorithms and implementation**:
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket) - Classic rate limiting algorithm applicable to token consumption
- [Redis for Rate Limiting](https://redis.io/glossary/rate-limiting/) - High-performance distributed counters for quota tracking
- [GCRA Algorithm](https://brandur.org/rate-limiting) - Generic Cell Rate Algorithm for smooth rate limiting

**AI provider documentation**:
- [OpenAI Rate Limits](https://platform.openai.com/docs/guides/rate-limits) - OpenAI's token-based rate limiting approach
- [Anthropic Rate Limits](https://docs.anthropic.com/en/api/rate-limits) - Claude API rate limiting and token quotas
- [Azure OpenAI Rate Limits](https://learn.microsoft.com/en-us/azure/ai-services/openai/quotas-limits) - Token quotas and provisioned throughput

**AI gateway implementations**:
- [LiteLLM Budgets](https://docs.litellm.ai/docs/proxy/budget) - Open-source token budget management
- [Portkey AI Gateway](https://portkey.ai/features/ai-gateway) - Commercial gateway with token-aware rate limiting
- [Kong AI Gateway](https://konghq.com/products/kong-ai-gateway) - Enterprise gateway with token management capabilities

**Related IGT-AI framework resources**:
- [IGT-AI risk model - Runaway costs](../risks.md#runaway-costs) - Primary financial risk addressed by this pattern
- [IGT-AI risk model - Denial-of-wallet attacks](../risks.md#denial-of-wallet-attacks) - Attack pattern mitigated by token-aware limiting
- [AI Gateway definition - Token-aware rate limiting](../ai-gateway-definition.md#token-aware-rate-limiting) - Technical implementation details
- [Dynamic model selection pattern](dynamic-model-selection.md) - Complementary pattern for cost optimization through model routing
- [Semantic caching pattern](semantic-caching.md) - Complementary pattern for reducing token consumption through caching
