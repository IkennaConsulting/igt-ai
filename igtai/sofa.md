
# Risk models for AI API consumption

## SOFA risk model

Consuming AI APIs exposes organisations to the following categories of risks.

- **S**ecurity and privacy risks
- **O**utput Integrity and reliability risks
- **F**inancial and operational risks
- Silo **A**I risk

### Security and privacy risks

#### Credential leakage

API keys and other credentials used to access AI APIs can be leaked through
 code repositories or logs, allowing unauthorized access
 to AI services.

#### Sensitive data disclosure

When sending data to AI APIs, sensitive information may be inadvertently
 included in requests, leading to potential data breaches or compliance
 violations. Examples include Personally Identifiable Information (PII),
 Protected Health information (PHI), financial data, or proprietary business
 information.

#### Prompt injection

Malicious actors can manipulate prompts sent to AI APIs to produce harmful or
 unintended outputs, leading to misinformation by the model, offensive content,
 or security vulnerabilities.

### Output Integrity and reliability risks

#### Hallucination

AI models may generate incorrect or fabricated information, which can mislead
 users or result in poor decision-making.

#### Off-topic content

Attackers can send queries that are irrelevant to the business
 or out-of scope of the generative AI application.

#### Toxic, profane, and off-brand content

A model can produce outputs that are biased, offensive, or harmful, damaging the
 organisation's reputation and user trust.

#### Lack of resilience

An inference service may become unavailable due to high demand, outages, or
 other issues, impacting application functionality.

### Financial and operational risks

#### Denial of wallet attacks

Attackers can exploit AI APIs by sending a high volume of requests, leading to
 unexpected costs and potential service disruptions.

#### Runaway costs

Without proper monitoring and controls, the usage of AI APIs can lead to
 significant and unexpected expenses. This risk is also referred to as unbound consumption.

#### High application latency

AI APIs can introduce latency into applications, especially if the API
 provider experiences high demand or outages, impacting user experience. Delays
 between user input and model input can lead to user frustration and reduced
 engagement.

### Silo AI risks

#### Drift in policy enforcement

#### Fractured audit trails

#### Shadow AI

## Other AI consumption risk models

- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/)
- [Google Secure AI Framework](https://saif.google/secure-ai-framework/risks)
- [FINOS AI Governance Framework](https://air-governance-framework.finos.org/)
