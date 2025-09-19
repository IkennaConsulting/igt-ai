
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

## Other AI consumption risk models

- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/)
- [Google Secure AI Framework](https://saif.google/secure-ai-framework/risks)
- [FINOS AI Governance Framework](https://air-governance-framework.finos.org/)
