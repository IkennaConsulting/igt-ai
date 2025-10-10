
# Multi-model routing and failover

## The problem

Organizations deploying AI-powered applications face a critical challenge: **AI model and provider outages can completely block application functionality**, and **latency spikes from provider infrastructure can severely degrade user experience**. Unlike traditional API services where a single provider often suffices, AI workloads require access to multiple models with different capabilities, and provider reliability varies significantly. When an application depends on a single AI provider or model, any service degradation or outage directly translates to broken features and frustrated users.

Furthermore, different AI tasks require different model capabilities—code generation benefits from code-specialized models, vision tasks require multimodal models, and long-form content needs models with large context windows. Applications that hardcode dependencies on specific models cannot easily adapt when better models become available, when compliance requirements change, or when cost optimization demands switching providers.

## Forces

Several competing concerns make this problem challenging to solve:

- **Provider volatility**: AI providers experience outages, rate limiting, and capacity constraints more frequently than traditional cloud services. New providers enter the market regularly, and existing providers deprecate models or change pricing unexpectedly.

- **Model capability diversity**: Different models excel at different tasks. A model optimized for code generation may perform poorly at creative writing. Vision models handle images while language models process text. No single model serves all use cases equally well.

- **Varying availability and performance**: Provider service level agreements differ, geographic availability varies, and latency characteristics change based on load. A provider may be fast in one region but slow in another, or experience degraded performance during peak hours.

- **Cost constraints**: Routing all requests to the highest-capability model is expensive. Organizations need to balance quality requirements against budget limitations, routing simple tasks to cheaper models and complex tasks to premium models.

- **Consistency requirements**: Some applications require deterministic outputs or consistent model behavior across requests. Failover to different models may produce varying response styles or quality levels that impact user experience.

- **Compliance and data residency**: Regulatory requirements may restrict which providers can process certain data types. European customer data might require EU-hosted models, while healthcare data requires HIPAA-compliant providers.

- **Observability complexity**: Tracking which model served which request becomes difficult when routing is dynamic. Debugging issues, attributing costs, and analyzing performance require clear visibility into routing decisions.

## Risk the pattern mitigates

This pattern directly mitigates the following risks from the [IGT-AI risk model](../risks.md):

- **Lack of resilience**: By implementing failover and hedging strategies, applications become resilient to individual model or provider outages. When the primary provider fails, requests automatically route to backup models, preventing complete application failure.

- **High application latency**: Hedging strategies that send requests to multiple providers simultaneously and return the fastest response protect against latency spikes from any single provider. Intelligent routing can also direct requests to lower-latency providers based on real-time performance metrics.

## How it works

Multi-model routing and failover combines **capability-based routing** with **resilience strategies** to ensure reliable AI service delivery. The solution consists of two complementary mechanisms:

### Capability-based routing

The gateway analyzes incoming requests and routes them to appropriate models based on:

- **Task type classification**: Code generation requests route to code-specialized models like Claude Sonnet or GPT-4 with code capabilities, vision tasks route to multimodal models like GPT-4 Vision or Claude 3.5 Opus, and general text tasks route to language models optimized for conversation.

- **Complexity assessment**: Simple queries route to fast, cost-effective models while complex reasoning tasks route to more capable premium models. The gateway may classify complexity based on prompt length, keywords indicating difficulty, or explicit application hints.

- **Context requirements**: Requests with large conversation histories or long documents route to models with sufficient context window capacity, while short queries can use models with smaller context limits.

- **Compliance and governance rules**: Requests containing regulated data route only to compliant providers. For example, healthcare data routes to HIPAA-compliant models, European customer data routes to EU-hosted providers, and prompts requiring training opt-out route to providers that honor zero data retention.

- **Performance and cost targets**: Applications can specify latency budgets or cost constraints, and the gateway selects models that meet those requirements while satisfying capability needs.

Unlike generic load balancing that distributes requests evenly across identical backends, capability-based routing makes **content-aware decisions** that match requests to the most appropriate model based on semantic and business requirements.

### Failover and hedging strategies

When models fail or perform poorly, the gateway employs resilience patterns:

- **Automatic failover**: If the primary model returns an error, times out, or is unavailable, the gateway automatically retries the request with alternative models or providers. For example, if OpenAI returns a 503 service unavailable error, the gateway immediately retries with Anthropic, then Azure OpenAI, until a successful response is received or all alternatives are exhausted.

- **Hedging (parallel requests)**: For latency-sensitive applications, the gateway sends the same request to multiple models or providers **simultaneously** and returns whichever responds first. This protects against tail latency from any single provider. For instance, a request might be sent to both OpenAI and Anthropic at the same time, and the first complete response is returned to the client while the slower response is discarded.

- **Circuit breakers**: If a provider consistently fails or experiences degraded performance, the gateway temporarily removes it from the routing pool, preventing cascading failures and wasted retry attempts. After a cooldown period, the gateway tests the provider with limited traffic before fully restoring it.

- **Graceful degradation**: When all primary models fail, the gateway can fall back to lower-quality but more reliable models, return cached responses if available, or provide meaningful error messages to users rather than complete failure.

- **Health checking**: The gateway continuously monitors model availability, latency, and error rates, using this data to inform routing decisions and failover triggers in real time.

### Implementation variants

Organizations can implement this pattern at different levels of sophistication:

- **Static routing with manual failover**: Simplest approach where routing rules are predefined and failover happens only on explicit errors. Administrators manually configure which models handle which request types.

- **Dynamic routing with automatic failover**: The gateway uses runtime metrics and health checks to automatically select models and failover without manual intervention.

- **Intelligent routing with ML-based classification**: Advanced implementations use machine learning to classify request complexity, predict optimal model selection, and learn from past routing decisions to improve performance over time.

- **Cost-optimized hedging**: Instead of always hedging to multiple providers, the gateway hedges selectively based on request priority, latency requirements, or when provider performance is degraded.

## Solution checklist links

Implementation guidance for multi-model routing and failover can be found in:

- **AI Gateway Definition**: See sections on [Multi-model routing and load balancing](../ai-gateway-definition.md#multi-model-routing-and-load-balancing) and [Model failover and hedging](../ai-gateway-definition.md#model-failover-and-hedging) for technical implementation details.

- AI gateway product evaluations and implementation checklists (when available in the framework).

## Contraindications

This pattern should **not** be used when:

- **Cost of redundant calls is prohibitive**: Hedging strategies that send requests to multiple providers simultaneously double or triple the token consumption and costs. Organizations with tight budgets may find hedging too expensive for routine use.

- **Response consistency is critical**: Different models produce different response styles, tones, and quality levels. Applications requiring deterministic outputs or consistent brand voice across all responses may struggle with failover to alternative models that behave differently.

- **Request contains sensitive state**: If requests include conversation context or state that is specific to a particular model or provider, failing over to a different model may break conversation continuity or lose important context.

- **Single-provider compliance requirement**: Some regulatory or contractual obligations may mandate use of a specific provider or model version. In these cases, failover to alternatives violates compliance requirements.

- **Debugging and reproducibility needs**: When troubleshooting specific model behavior or reproducing exact outputs for testing, routing variability makes debugging more difficult. Development and testing environments often benefit from deterministic model selection rather than dynamic routing.

Alternative approaches for these scenarios:

- For cost-sensitive workloads, rely on automatic failover only (not hedging) and use semantic caching to reduce overall request volume.
- For consistency-critical applications, use a single primary model with failover only to equivalent model versions from the same provider.
- For stateful conversations, implement session affinity (sticky routing) to ensure all requests in a conversation use the same model.

## References

- **AI Gateway Definition**: [Multi-model routing and load balancing](../ai-gateway-definition.md#multi-model-routing-and-load-balancing) and [Model failover and hedging](../ai-gateway-definition.md#model-failover-and-hedging)
- **IGT-AI Risk Model**: [Lack of resilience](../risks.md#lack-of-resilience) and [High application latency](../risks.md#high-application-latency)
- [Martian AI Gateway](https://withmartian.com/) - Commercial implementation with multi-provider routing
- [Portkey AI Gateway](https://portkey.ai/) - Offers failover, load balancing, and hedging capabilities
- [LiteLLM Proxy](https://github.com/BerriAI/litellm) - Open-source proxy with multi-provider routing
- Fowler, Martin (2003). ["CircuitBreaker"](https://martinfowler.com/bliki/CircuitBreaker.html) - Classic pattern for handling service failures
- Dean, Jeff and Barroso, Luiz André (2013). ["The Tail at Scale"](https://research.google/pubs/pub40801/) - Google research on hedging and latency management
