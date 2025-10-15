# IGT-AI

# IGT-AI

A risk-based framework for evaluating runtime AI-API governance and integration efficiency. The risks mitigated by the IGT-AI framework are detailed in the
[IGT-AI risks document](risks.md).

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

## What risks does it mitigate?  
[See the risks section](igtai/risks.md)

# IGT-AI control patterns

## The patterns

### Security, privacy & compliance

- [Centralized credential management](centralized-credential-management.md)
- [Input & output guardrails](input-output-guardrails.md)
- [Prompt templating](prompt-templating.md)

### Cost management & optimization

- [Token-aware rate limiting and budgets](token-aware-rate-limiting.md)
- [Semantic caching](semantic-caching.md)
- [Dynamic model selection](dynamic-model-selection.md)

### Reliability & orchestration

- [Provider-agnostic API abstraction](provider-agnostic-api-abstraction.md)
- [Multi-model routing and failover](multi-model-routing-failover.md)
- [Streaming response management](streaming-response-management.md)
- [Context window management](context-window-management.md)

### Observability & governance

- [Audit trails and compliance logging](audit-trails-compliance-logging.md)
- [Model performance monitoring](model-performance-monitoring.md)
- [Distributed tracing](distributed-tracing.md)

### Developer experience

- [AI model catalog and playground](ai-model-catalog-playground.md)
- [Response validation and transformation](response-validation-transformation.md)
- [Docs and SDKs](docs-sdks.md)

## Checklists
- [See the checklist for evaluating AI gateways](igtai/checklists/checklist.md) 
