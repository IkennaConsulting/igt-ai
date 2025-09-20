# IGT-AI canvas

## 1. AI-API workflow

- Does it support GitOps workflow?
- If the organisation is a Kubernetes shop, is the AI gateway
  Kubernetes-native?
- How easy is it to get started?
- If it's a cloud platform, how easy is it to integrate
  with inference services not on its platform?

## 2. Security, privacy, and compliance

- How does the AI gateway support centralised management for AI provider
  API credentials?
- How does the AI gateway provide basic internal guardrails for PII detection and
  regex filters for disallowed keywords and phrases? (List essential PII)
- How does the AI gateway support integration with external guardrails?
  - If so, how many, and which
    ones?
  - How easy is it to integrate with external guardrails?
- Do the guardrails work for a streaming API?
- Does it provide an air-gapped deployment option?
- Does it support prompt templating to avoid
  prompt injection attacks, and facilitate versioning of prompt logic?

## 3. Cost optimisation and performance

- How does the AI gateway support **token-based** rate limiting?
- Does the AI gateway provide configurable, token-based cost monitoring?
  - Does it provide dashboards?
  - How does it provide alerting?
- Does it provide semantic caching support?
  - For third-party AI service providers, if this is a cloud platform?
- Does it provide OTel-based observability (logs, metrics, traces)?
- Does it support conditional routing scenarios:
  - Routing by cost?
  - Fallback mechanism, for example, if an AI provider is down?

## 4. Developer experience and collaboration

- Does it provide an out-of-the-box OpenAI compatible API for inference?
- Does it provide an out-of-the-box SDK for developer integration?
- How does it support extending the unified API?
- How does it support abstracting away the model from the calling client?
- Does it provide an AI model catalogue?
- Is it open-source or commercial? If open-source, is there an active
  community around it?
