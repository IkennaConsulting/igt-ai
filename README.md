# IGT-AI


A risk-based framework for evaluating runtime AI-API governance and integration efficiency.
It is a collection of feature patterns and checklists for evaluating [AI gateways](igtai/ai-gateway-definition.md). 
The risks mitigated by the IGT-AI patterns are detailed in the
[IGT-AI risks document](igtai/risks.md).

## What runtime AI-API governance means

Runtime AI-API governance is the set of policies, controls,
and monitoring mechanisms
applied at the point where AI systems are invoked through APIs. That is, while
AI-APIs are being consumed in production. It's the governance in operation in
real time when an AI model or service is called, rather than only during
design or development.

AI–APIs are the AI capabilities exposed via APIs, and governance must happen at
that API boundary. Runtime governance focuses on what happens during live
interactions with AI models (inference requests, responses, usage patterns).
APIs are the ideal control point for enforcing runtime governance policies as
every AI request/response passes through the API layer.

## What integration efficiency means

Integration efficiency is about reducing friction, complexity,
and time taken to integrate AI-APIs into
applications while ensuring governance policies are effectively enforced.
It is also reducing the friction and time it takes to integrate the governance
mechanisms themselves with other systems.

## IGT-AI control patterns

### Security, privacy & compliance

- [Centralized credential management](igtai/patterns/centralized-credential-management.md)
- [Input & output guardrails](igtai/patterns/input-output-guardrails.md)
- [Prompt templating](igtai/patterns/prompt-templating.md)

### Cost management & optimization

- [Token-aware rate limiting and budgets](igtai/patterns/token-aware-rate-limiting.md)
- [Semantic caching](igtai/patterns/semantic-caching.md)
- [Dynamic model selection](igtai/patterns/dynamic-model-selection.md)

### Reliability & orchestration

- [Provider-agnostic API abstraction](igtai/patterns/provider-agnostic-api-abstraction.md)
- [Multi-model routing and failover](igtai/patterns/multi-model-routing-failover.md)
- [Streaming response management](igtai/patterns/streaming-response-management.md)

### Observability 

- [Audit trails and compliance logging](igtai/patterns/audit-trails-compliance-logging.md)
- [Model performance monitoring](igtai/patterns/model-performance-monitoring.md)
- [Distributed tracing](igtai/patterns/distributed-tracing.md)

### Developer experience

- [AI model catalog and playground](igtai/patterns/ai-model-catalog-playground.md)
- [Response validation and transformation](igtai/patterns/response-validation-transformation.md)
- [Docs and SDKs](igtai/patterns/docs-sdks.md)

## Checklists
- [See the checklist for evaluating AI gateways](igtai/checklists/checklist.md) 
