# Centralized credential management

## The problem

Organizations consuming AI APIs face a growing challenge: API credentials scattered across multiple applications, teams, and deployment environments. Development teams embed API keys directly in application code, environment files, or configuration systems. As AI adoption accelerates, the number of credentials proliferates. Each new application requires keys for OpenAI, Anthropic, Cohere, or other providers. Teams duplicate credentials across development, staging, and production environments. When a key is compromised, identifying all affected systems becomes an archaeological exercise.

This fragmentation creates security exposure. Hardcoded keys leak into version control systems. Developers share credentials through insecure channels. Rotation becomes a coordination nightmare requiring synchronized updates across dozens of repositories. Access control is coarse-grained—either a team has a key or it doesn't, with no ability to enforce granular policies per user or use case. Audit trails are non-existent: when unexpected costs appear, tracing which application and which developer triggered the expense is impossible.

The problem extends beyond security. AI provider API keys have usage limits tied to billing accounts, requiring coordination between finance, security, and engineering teams. Different teams need different quotas and access scopes, but credential distribution makes enforcement difficult. When providers mandate key rotation or when security incidents require emergency revocation, the manual hunt across codebases and deployment environments delays response times.

## Forces

Several competing factors make centralized credential management challenging:

**Developer autonomy versus security control**: Developers need frictionless access to AI APIs for rapid prototyping and experimentation. Requiring approval workflows for every API call stifles innovation. But unrestricted credential distribution creates security vulnerabilities and cost overruns. The tension between velocity and governance pushes teams toward convenience, embedding keys directly in code for immediate access.

**Multiple AI providers with different authentication mechanisms**: OpenAI uses bearer tokens, Anthropic uses API keys with specific header formats, Azure OpenAI requires subscription keys and endpoint URLs, and self-hosted models may use OAuth or custom authentication. Normalizing these diverse mechanisms while maintaining provider-specific features adds complexity. Teams must abstract away differences without losing critical functionality.

**Rotation policies and key lifecycle management**: Security best practices mandate periodic credential rotation, but AI workloads are often stateless and distributed. Rotating a key requires coordinating updates across applications, ensuring zero-downtime transitions, and verifying that old credentials are fully decommissioned. The operational burden makes rotation a rare event instead of a routine process.

**Billing limits and quota management**: AI providers typically tie API keys to billing accounts with usage quotas. A single compromised or misconfigured key can exhaust an organization's monthly budget in hours. Teams need scoped access—research teams should not consume production budgets, and experimental projects should have strict limits. But distributing multiple keys reintroduces the fragmentation problem.

**Team and project isolation requirements**: In multi-tenant organizations, different business units require isolated access. Marketing should not see engineering's prompts in audit logs. Customer-facing applications should not share credentials with internal analytics tools. Achieving isolation while maintaining central visibility and control requires careful architectural design.

**Compliance and audit requirements**: Regulatory frameworks like SOC 2, GDPR, and HIPAA require demonstrating who accessed which AI services, when, and with what data. Scattered credentials make audit trails incomplete. Centralized management enables comprehensive logging but introduces a single point of monitoring and potential failure.

**Emergency revocation needs**: When a security incident occurs—a compromised key, a departing employee, or a policy violation—immediate revocation is critical. With distributed credentials, emergency response requires identifying all affected systems and deploying updates simultaneously. Delays create windows of exposure.

## Risk the pattern mitigates

This pattern directly addresses **[Credential leakage](../risks.md#credential-leakage)** from the IGT-AI risk model.

Credential leakage occurs when API keys and authentication tokens used to access AI services are exposed through code repositories, logs, error messages, or insecure storage. Once leaked, credentials enable unauthorized access to AI services, resulting in:

- **Unauthorized consumption**: Attackers use leaked credentials to consume AI services at the organization's expense, potentially exhausting quotas and generating unexpected costs (related to denial-of-wallet attacks).
- **Data exfiltration**: Compromised keys may provide access to conversation histories, stored prompts, or cached responses containing sensitive information.
- **Reputational damage**: Public disclosure of leaked credentials demonstrates poor security practices, damaging customer trust and potentially triggering regulatory scrutiny.
- **Service disruption**: Providers may suspend accounts when they detect suspicious activity from compromised credentials, impacting legitimate applications.

Centralized credential management reduces this risk by eliminating the need to distribute credentials to application code, enforcing key isolation, automating rotation, and providing rapid revocation capabilities.

## How it works

Centralized credential management establishes the AI gateway as the single point of credential ownership. Applications never possess provider API keys. Instead, the gateway stores credentials in a secure vault, authenticates applications through a separate mechanism, and injects provider credentials at request time.

### Architecture components

**Secure credential vault**: The gateway integrates with a hardened secrets management system such as HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, or Google Secret Manager. Provider API keys are stored encrypted at rest with access controls restricting retrieval to the gateway process only. The vault provides audit logs of credential access, versioning for rollback during incidents, and time-based automatic rotation hooks.

**Application authentication layer**: Applications authenticate to the gateway using OAuth 2.0, API keys managed by the gateway itself, mTLS certificates, or integration with enterprise identity providers like Okta or Azure AD. This separates application identity from provider credentials. The gateway validates application tokens, maps them to authorized scopes and policies, and determines which provider credentials to use based on application identity, team membership, and resource requirements.

**Credential injection at request time**: When an application sends a request to the gateway, it includes its own authentication token but no provider credentials. The gateway authenticates the application, retrieves the appropriate provider credential from the vault based on routing rules and team assignments, and injects the credential into the outbound request to the AI provider. Credentials never leave the gateway infrastructure, remaining invisible to applications.

**Multi-tenancy and key isolation**: Large organizations require multiple credentials for the same provider to enforce quotas, isolate billing, or meet compliance requirements. The gateway maintains a mapping between teams, projects, or environments and specific credentials. Production applications use production keys with high quotas and strict monitoring. Experimental projects use separate keys with aggressive rate limits. The gateway enforces this isolation transparently, routing requests through the correct credential based on policy.

### Rotation and lifecycle management

Centralized storage enables automated key rotation without application changes:

**Scheduled rotation**: The gateway defines rotation schedules per credential (e.g., every 90 days). Prior to rotation, the gateway provisions new credentials from the provider (if supported by the provider's API), stages them in the vault with a future activation date, and begins a phased rollout. During the transition period, the gateway accepts both old and new credentials to ensure zero downtime. Once the new credential is validated across all traffic, the old credential is revoked and purged.

**Emergency revocation**: When a security incident is detected, administrators revoke credentials through the gateway's control plane. The gateway immediately ceases using the compromised credential for all requests. If a replacement credential is available, traffic seamlessly shifts to the backup. If not, the gateway returns authentication errors to applications, providing fast fail instead of silent security compromise. Audit logs capture the revocation event, affected requests, and the administrator who initiated the action.

**Credential versioning**: The vault maintains historical versions of credentials, allowing rollback if a newly rotated key exhibits unexpected behavior or rate limiting. The gateway exposes controls to promote or demote credential versions without redeploying application code.

### Scoped access and quota enforcement

Centralized management enables fine-grained access control beyond traditional API keys:

**Per-team quotas**: Finance allocates a monthly budget to each department. The gateway maps departments to specific credentials with associated quotas. As teams consume tokens, the gateway tracks usage against their assigned budget. When a team approaches its limit, the gateway sends alerts to administrators and optionally throttles requests to prevent overruns. This decouples application deployment from financial governance.

**Per-user attribution**: Even within a team, the gateway can track individual user consumption. Applications pass user identifiers in requests (e.g., via JWTs). The gateway logs which user triggered which prompt, enabling cost attribution at the individual level for internal chargeback or identifying power users who may need training on cost-effective prompting.

**Use-case-specific credentials**: Organizations may negotiate different terms with AI providers for different use cases. A customer support chatbot might have unlimited access to a fast model, while internal analytics tools use a cheaper model with lower quotas. The gateway associates credentials with use case tags, routing requests through the appropriate credential based on application metadata.

### Audit and compliance

Centralized credential management creates comprehensive audit capabilities:

**Credential access logs**: The vault logs every time the gateway retrieves a credential, creating an immutable record of which gateway instance accessed which key at what time. Anomalies in access patterns (e.g., sudden spikes or access from unexpected geographic regions) trigger alerts for investigation.

**Request-to-credential mapping**: The gateway correlates each AI request with the specific credential used, the application that initiated it, the user context, and the timestamp. This end-to-end traceability supports compliance audits and security investigations. When regulators ask "who accessed this model with this data," the gateway provides a complete chain of evidence.

**Compliance enforcement**: Regulatory requirements may dictate that certain data types only be processed using specific credentials tied to compliant deployments (e.g., HIPAA-compliant Azure OpenAI instances). The gateway enforces these mappings, rejecting requests that attempt to use non-compliant credentials for regulated data based on automated PII detection or explicit application labeling.

## Solution checklist links

Implementation guidance for centralized credential management:

- [AI gateway feature checklist](../solutions/ai-gateway-features.md) (to be created)
- [Secrets management integration patterns](../solutions/secrets-management.md) (to be created)

Reference implementations:

- HashiCorp Vault integration for credential rotation
- AWS Secrets Manager with gateway examples
- Azure Key Vault integration guides
- OAuth 2.0 application authentication flows

## Contraindications

Centralized credential management introduces architectural dependencies and operational complexity that may not be justified in all contexts:

**Early-stage prototypes and experiments**: For initial AI experiments with small teams and low stakes, the overhead of configuring a gateway and secrets vault may exceed the risk of short-term credential distribution. Teams prototyping a proof-of-concept with a single developer might reasonably use a local environment variable for expediency, deferring centralization until the project approaches production.

**Edge and offline deployments**: Applications running on edge devices, mobile clients, or intermittent connectivity environments cannot rely on synchronous calls to a central gateway. These scenarios may require distributing scoped, short-lived credentials directly to endpoints, accepting higher leakage risk in exchange for operational viability. Alternatives include on-device credential management or gateway-issued JWT tokens with limited lifetimes.

**High-latency sensitivity**: The gateway adds a network hop between applications and AI providers. For latency-critical applications where every millisecond matters, the credential injection overhead may be unacceptable. In such cases, teams might use short-lived credentials directly distributed to applications with aggressive rotation schedules, trading some centralization benefits for performance.

**Third-party vendor integrations**: When using AI features embedded in third-party SaaS platforms (e.g., Salesforce Einstein, Zendesk AI), the vendor requires direct access to AI provider credentials for their internal operations. Organizations cannot interpose a gateway in these flows. In these scenarios, credential management must rely on vendor security practices and contractual guarantees rather than technical controls.

**Regulatory or compliance requirements for full decentralization**: Some regulatory environments mandate that no single system has access to all organizational credentials, requiring physical separation and multi-party authorization for credential retrieval. Centralized credential management may violate these requirements unless implemented with sufficient segmentation and access controls that effectively decentralize authority.

**Open-source or community-driven projects**: Projects without organizational backing or formal security requirements may find centralized credential management impractical. Individual contributors may need direct API access without complex infrastructure dependencies. In these contexts, documentation and training on secure credential handling may be more pragmatic than technical enforcement.

Despite these contraindications, most enterprise AI deployments benefit from centralized credential management. The pattern should be the default choice for production systems with multiple teams, compliance requirements, or significant cost exposure.

## References

**Industry standards and best practices:**

- [OWASP API Security Top 10 - API2:2023 Broken Authentication](https://owasp.org/API-Security/editions/2023/en/0xa2-broken-authentication/) - Authentication vulnerabilities applicable to API credential management
- [NIST SP 800-57 Part 1 Rev. 5 - Recommendation for Key Management](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final) - Federal standards for cryptographic key lifecycle management
- [CIS Controls v8 - Control 3: Data Protection](https://www.cisecurity.org/controls/data-protection) - Data protection controls including credential management

**Secrets management platforms:**

- [HashiCorp Vault](https://www.vaultproject.io/) - Industry-standard secrets management with dynamic credentials and rotation
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) - Cloud-native secrets management with automatic rotation
- [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/) - Microsoft's managed secrets and key management service
- [Google Cloud Secret Manager](https://cloud.google.com/secret-manager) - Google Cloud's secrets management service

**AI provider credential documentation:**

- [OpenAI API Authentication](https://platform.openai.com/docs/api-reference/authentication) - OpenAI bearer token authentication
- [Anthropic API Keys](https://docs.anthropic.com/en/api/getting-started) - Anthropic API key management and best practices
- [Azure OpenAI Authentication](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference) - Azure-specific authentication mechanisms

**Related patterns and frameworks:**

- [IGT-AI risk model - Credential leakage](../risks.md#credential-leakage) - Risk definition this pattern addresses
- [IGT-AI AI gateway definition - Centralized credential management](../ai-gateway-definition.md#centralized-credential-management) - Technical gateway capabilities
- [FINOS AI Governance Framework](https://air-governance-framework.finos.org/) - Broader AI governance context
