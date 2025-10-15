# IGT-AI canvas checklist

A checklist for a basic evaluation of AI gateways for runtime AI governance of
AI-API consumption. This checklist focuses on using an AI gateway to mitigate
the [IGT-AI risks](../risks.md).

## 1. AI-API workflow

- How easy is it to get started with the AI gateway?
- Does it support GitOps workflow?
- If the organisation is a Kubernetes shop, is the AI gateway
  Kubernetes-native?
- If it's a cloud platform, how easy is it to integrate
  with inference services not on its platform?

## 2. Security, privacy, and compliance

- How does the AI gateway support centralised management for AI provider
  API credentials?
- How does the AI gateway provide basic internal guardrails for PII detection
  and regex filters for disallowed keywords and phrases? (See
  [The IGT-AI PII list](../pii.md))
- How does the AI gateway support integration with external guardrails?
  - If so, how many, and which ones?
  - How easy is it to integrate with external guardrails?
- Does it provide an air-gapped deployment option?
  An air-gapped environment is one that's secure and isolated from
  the internet and other external networks,
  making it resistant to external
  <!-- vale Vale.Spelling = NO -->cyber attacks.<!-- vale Vale.Spelling = YES -->
- Does it support prompt templating to avoid
  prompt injection attacks, and facilitate versioning of prompt logic?

## 3. Cost optimisation and performance

- How does the AI gateway support **token-aware** rate limiting?
- Does the AI gateway provide configurable, token-based cost monitoring?
  - Does it provide dashboards for cost monitoring?
  - How does it provide alerting for cost monitoring?
- Does it provide semantic caching support?
  - For third-party AI service providers, if this is a cloud platform?
- Does it provide OTel-based observability (logs, metrics, traces)?

## 4. Developer experience and collaboration

- Does it provide an out-of-the-box OpenAI-compatible API for inference?
  - OpenAI API is now the de facto standard for AI model inference APIs.
- Does it provide an out-of-the-box SDK for developer integration?
- How does it support abstracting away the model from the calling client?
  (For cases where the team wants to switch between models in the same
  family, or between model providers, without changing the client)
- Does it provide an AI model catalogue or inventory so developers can see
  what models they can access?
