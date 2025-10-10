
# Dynamic model selection

## The problem

Applications that hardcode specific AI model choices cannot adapt to changing cost constraints, workload complexity, or performance requirements. When every request routes to the same model regardless of query complexity, organizations face two inefficiencies: simple queries waste money on expensive, high-capability models, while budget-conscious teams manually implement model selection logic in each application, creating scattered, inconsistent optimization strategies.

This problem intensifies as AI providers expand model offerings. A simple factual question might cost fractions of a cent with a small fast model, while the same query sent to a frontier reasoning model could cost 100 times more. Without intelligent routing, applications either overspend on every request or sacrifice quality by always choosing the cheapest option.

Traditional API gateways route based on URL paths or headers, but AI workloads require content-aware routing that understands query semantics, estimates complexity, and matches requests to appropriate model capabilities. Application developers should focus on building features, not implementing cost optimization algorithms.

## Forces

Several competing forces make model selection challenging:

- **Cost variation across models**: Pricing differences between models can exceed 100x for the same request. Frontier models with advanced reasoning capabilities charge premium rates, while smaller models optimized for speed and efficiency cost significantly less per token.

- **Query complexity differences**: User queries range from simple factual lookups that any model can handle to complex reasoning tasks requiring advanced capabilities. The complexity is not always obvious from request metadata alone and may require semantic analysis.

- **Performance requirements vary by use case**: Customer-facing chatbots need sub-second latency, while background summarization jobs can tolerate slower, more thorough processing. Different models offer different latency-quality trade-offs.

- **Model capability heterogeneity**: Models differ in context window size, supported features like function calling or vision, training data recency, and domain expertise. A coding question benefits from a code-specialized model, while general knowledge queries work with any capable model.

- **Budget constraints per tenant or team**: Multi-tenant systems must enforce different cost limits for different users. Enterprise customers might have access to premium models, while free-tier users route to economical options.

- **Quality expectations cannot be sacrificed**: Cost optimization must not degrade output quality below acceptable thresholds. Users expect consistent, correct responses regardless of which model processes their request.

- **Application code should remain model-agnostic**: Embedding model selection logic in applications couples code to specific providers and models, making updates difficult and preventing centralized optimization improvements.

## Risk the pattern mitigates

This pattern directly mitigates **runaway costs** from the [IGT-AI risk model](../risks.md#runaway-costs). Without intelligent model selection, organizations face unbound consumption where every query consumes maximum-cost resources regardless of need.

The pattern also indirectly reduces **denial of wallet attacks** by automatically routing expensive or suspicious requests to cheaper models or applying additional scrutiny before invoking premium capabilities. Attackers attempting to drain budgets through verbose or repeated queries encounter cost barriers.

## How it works

An AI gateway with dynamic model selection intercepts incoming requests and analyzes them to determine the most appropriate model based on configured routing rules. The selection process considers multiple factors:

### Request analysis and routing

The gateway examines the incoming request to extract routing signals:

- **Prompt length and token count**: Short queries likely represent simple questions suitable for fast, economical models. Long, detailed prompts with extensive context may require models with larger context windows and better reasoning capabilities.

- **Semantic complexity classification**: The gateway can use a small, fast classifier model to categorize request complexity. For example, factual lookups, creative writing tasks, and complex multi-step reasoning problems route to different model tiers.

- **Required capabilities detection**: Requests that include images route to vision-capable models. Function calling requirements trigger routing to models that support tool use. Queries needing recent information might prefer models with more current training data.

- **User or tenant tier**: Authentication metadata determines which models are available. Free tier users route to economical options, while premium subscribers access frontier models.

- **Historical performance data**: The gateway tracks which models performed well for similar queries in the past, learning from usage patterns to improve routing decisions over time.

### Routing strategies

The gateway supports multiple routing approaches:

**Rule-based routing** applies explicit conditions configured by administrators. Examples include:

- Prompts under 200 tokens route to GPT-3.5 Turbo
- Coding questions identified by keywords route to code-specialized models
- Enterprise tier users always access Claude Opus
- Off-peak hours use more expensive models to balance load and cost

**Intelligent routing** uses machine learning to classify queries and predict optimal models. A lightweight classifier analyzes semantic features and estimates whether a query requires advanced reasoning, domain expertise, or simple pattern matching. This approach adapts to changing workload patterns without manual rule updates.

**Cost-constrained routing** enforces budget limits by automatically downgrading model selection when token budgets approach limits. If a user has consumed 80% of their monthly allocation, subsequent requests route to cheaper models to extend budget runway.

**Performance-optimized routing** prioritizes latency for time-sensitive requests while allowing background tasks to use slower, more thorough models. User-facing chatbot queries might route to fast models even at higher cost, while batch document processing uses economical options.

**Fallback cascades** attempt requests with a preferred model but automatically fall back to alternatives if the primary option is unavailable, rate-limited, or experiencing high latency.

### Model selection workflow

1. **Request interception**: The gateway receives an API call from a client application using a provider-agnostic interface or standard format like the OpenAI API schema.

2. **Context extraction**: The gateway extracts routing signals including prompt content, token counts, user identity, required capabilities, and any application-provided hints like priority flags.

3. **Routing decision**: Based on configured strategy, the gateway selects the target model. This decision may involve running a lightweight classifier, evaluating rule conditions, checking budget availability, or consulting a model recommendation cache.

4. **Request translation**: The gateway transforms the request into the target model's API format, handling differences in parameter names, authentication, and endpoint structure.

5. **Response handling**: After the model responds, the gateway transforms the response back to the client's expected format, ensuring the model choice remains transparent to the application.

6. **Feedback loop**: The gateway logs routing decisions and outcomes, enabling continuous improvement of selection logic through analysis of cost, latency, and quality metrics.

### Transparency and control

While dynamic selection operates transparently, the gateway can expose model selection information to applications and users:

- Response headers indicate which model processed the request
- Monitoring dashboards show routing distribution across models
- Applications can provide routing hints as preferences rather than hard requirements
- Override mechanisms allow forcing specific models for testing or debugging

## Solution checklist links

Dynamic model selection is implemented through AI gateway capabilities described in the [AI Gateway Definition](../ai-gateway-definition.md#dynamic-model-selection).

Implementation checklist items:

- Configure routing rules based on prompt characteristics and user tiers
- Deploy model capability metadata catalog for routing decisions
- Implement request classification logic for complexity assessment
- Set up cost tracking to inform routing decisions
- Define model tier hierarchies with automatic fallback chains
- Enable routing decision logging for analysis and optimization
- Establish override mechanisms for testing and debugging
- Configure response headers to expose selected model information
- Implement budget-aware routing thresholds
- Set up A/B testing infrastructure for routing strategy validation

## Contraindications

Dynamic model selection is not appropriate in these scenarios:

**Consistent model behavior is critical**: Applications that depend on specific model behaviors, formatting quirks, or capabilities should not use dynamic routing. If code relies on a particular model's response structure or reasoning style, switching models can break functionality.

**Quality variance is unacceptable**: High-stakes decisions requiring maximum accuracy and reliability should not tolerate model downgrading. Medical diagnoses, legal advice, financial recommendations, and safety-critical systems need consistent access to the most capable models regardless of cost.

**Reproducibility and auditability require model pinning**: Scientific research, compliance audits, or scenarios requiring exact reproducibility of results must use fixed model versions. Dynamic routing introduces variability that makes reproducing specific outputs difficult.

**Latency requirements are extremely strict**: The overhead of request analysis and routing decision logic adds milliseconds to processing time. Applications with single-digit millisecond latency budgets may not tolerate this overhead.

**User expectations include model awareness**: If users explicitly choose or expect specific models, transparent routing violates those expectations. Applications marketing "powered by GPT-4" cannot silently route to different models without user consent.

**Complex multi-turn conversations risk inconsistency**: Switching models mid-conversation can create jarring changes in response style, capability, or context understanding. Conversational applications may prefer consistent model selection throughout a session.

**Model-specific features are required**: Requests that use provider-specific capabilities like Claude's extended thinking mode or GPT's custom instructions cannot be transparently routed to models lacking those features.

## References

- [AI Gateway Definition - Dynamic Model Selection](../ai-gateway-definition.md#dynamic-model-selection)
- [IGT-AI Risk Model - Runaway Costs](../risks.md#runaway-costs)
- [Token-Aware Rate Limiting Pattern](token-aware-rate-limiting.md)

