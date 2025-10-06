
# IGT-AI control patterns

## The patterns

Patterns are marked as Must-Have (MH), Should-Have (SH), or Nice-to-Have (NH) based on their criticality for AI API governance.

### Security, privacy & compliance

- Centralized credential management (MH)
- PII detection and redaction (MH)
- Input & output guardrails (MH)
- Prompt templating and injection prevention (SH)

### Cost governance & optimization

- [Token-aware rate limiting and budgets](token-aware-rate-limiting.md) (MH)
- Semantic caching (SH)
- Dynamic model selection (SH)

### Reliability & orchestration

- Provider-agnostic API abstraction (MH)
- Multi-model routing and failover (MH)
- Streaming response management (SH)
- Context window management (SH)

### Observability & governance

- Audit trails and compliance logging (MH)
- Model performance monitoring (SH)
- Hallucination detection (NH)

### Developer experience

- AI model catalog (NH)
- Response validation and transformation (SH)
- AI model playground (NH)

## Pattern template

We use the following template for describing each pattern.

- **Name** of the pattern
- **The problem**: A description of the problem or design issue the pattern
 addresses.
- **Forces**: The various reasons or drivers that make the problem hard 
 to solve.
- **Risk the pattern mitigates**: This should be linked to one of the [IGT-AI
 risk model items](../risks.md).
- **How the solution(s) work**: An answer to the problem, describing how the
 solution works and any solution variants that exist. Includes any illustrative
 diagrams.
- **Solution checklist links**: A link to an AI-gateway checklist or any other 
 implementation solution checklist in the ch
- **Contraindications**: When not to use the pattern.
- **References**: Links to relevant resources describing the pattern.

