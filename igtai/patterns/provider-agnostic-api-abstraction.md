# Provider-agnostic API abstraction

## The problem

Applications calling AI providers directly create vendor lock-in. Switching from OpenAI to Anthropic requires code changes throughout your application—every API call, authentication method, and response handler must be updated. This makes it expensive and risky to change providers even when better models, lower costs, or improved availability emerge. Organizations cannot experiment with alternatives, implement failover, or negotiate from positions of strength.

## Forces

- **API format diversity**: Each provider uses different request/response structures, parameter names, and authentication schemes that require translation
- **Feature availability gaps**: Advanced capabilities (function calling, vision, JSON mode) exist on some providers but not others, with varying implementations
- **Performance trade-offs**: Abstraction adds latency overhead while enabling resilience and flexibility

## Risk the pattern mitigates

**[Lack of resilience](../risks.md#lack-of-resilience)**: Abstraction enables provider switching and failover without code changes. When one provider fails, traffic automatically routes to alternatives.

The pattern also supports **[Runaway costs](../risks.md#runaway-costs)** mitigation through dynamic provider routing and reduces **[Credential leakage](../risks.md#credential-leakage)** by centralizing credentials in the gateway.

## How it works

The gateway exposes a unified API (typically OpenAI-compatible format) that applications call regardless of backend provider. When requests arrive, the gateway routes to appropriate providers, translates parameters and authentication to provider-specific formats, forwards requests with secure credentials, and translates responses back to the standard format. Applications remain unaware of which provider processed their request.

For advanced features, the gateway translates function calling definitions, routes vision requests to capable providers, and handles structured output variations. When providers fail, automatic failover retries with alternatives while maintaining idempotency.

## Solution checklist links

- [AI Gateway Definition - Provider abstraction](../ai-gateway-definition.md#provider-agnostic-api-abstraction)
- Choose unified API format (OpenAI compatibility recommended)
- Implement translation layer for each supported provider
- Configure routing rules and secure credential storage
- Document feature parity across providers

## Contraindications

- **Provider-specific features critical**: Applications depending on unique capabilities (Anthropic's extended thinking, OpenAI's DALL-E parameters) lose functionality
- **Extreme performance requirements**: Abstraction adds latency unsuitable for sub-100ms response time requirements
- **Single provider commitment**: No resilience or flexibility needs and no plans to change providers

## References

- [AI Gateway Definition - Provider abstraction](../ai-gateway-definition.md#provider-agnostic-api-abstraction)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [LiteLLM](https://github.com/BerriAI/litellim) - Open source multi-provider proxy
- [Portkey AI Gateway](https://portkey.ai/)
- [Kong AI Gateway](https://konghq.com/products/kong-ai-gateway)
