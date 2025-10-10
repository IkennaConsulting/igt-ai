
# Input & output guardrails

## The problem

AI models process and generate natural language, making them vulnerable to a unique class of security and content risks that traditional API validation cannot address. Applications face four critical challenges when consuming AI APIs:

1. **Prompt injection attacks**: Malicious users can embed instructions within input that manipulate model behavior, potentially extracting confidential system prompts, bypassing safety controls, or causing the model to perform unintended actions.

2. **Sensitive data leakage**: Personally identifiable information (PII), protected health information (PHI), or proprietary business data can flow into prompts sent to external AI providers or appear in model-generated responses, creating compliance violations and privacy breaches.

3. **Toxic and inappropriate content**: Models can receive or generate content that is profane, offensive, hateful, sexually explicit, violent, or off-brand, damaging organizational reputation and user trust.

4. **Hallucinations and misinformation**: AI models may generate plausible but incorrect information, fabricate citations, or present speculation as fact, leading to poor decision-making and eroded user confidence.

Traditional input validation approaches—schema validation, type checking, regex patterns—fail because they cannot understand natural language semantics, detect manipulation attempts disguised as legitimate text, or evaluate the appropriateness of content in context. Organizations need AI-specific controls that analyze the meaning, intent, and safety of both incoming prompts and outgoing responses.

## Forces

Several factors make implementing effective guardrails for AI inputs and outputs particularly challenging:

**Natural language complexity**: Unlike structured data with predictable formats, natural language is ambiguous, context-dependent, and infinitely variable. The same phrase can be benign or malicious depending on surrounding context. Detecting manipulation requires understanding linguistic patterns, not just matching keywords.

**Context-dependent toxicity**: Content appropriateness depends on application domain and user relationship. Medical applications discuss sensitive health topics legitimately, while the same content would be inappropriate in customer service. Sarcasm, cultural references, and euphemisms make toxicity detection non-trivial.

**PII in unstructured text**: Sensitive information appears in countless formats within natural language. "My SSN is 123-45-6789" is obvious, but "I was born in May 1987 and my last four are 6789" requires inference. Names, dates, locations, and identifiers combine in unpredictable ways.

**Model unpredictability**: AI outputs are non-deterministic. The same prompt can produce different responses across requests. Models may occasionally leak training data, generate offensive content despite safety training, or hallucinate convincing falsehoods. Guardrails must handle this inherent variability.

**Performance vs. security trade-offs**: Deep semantic analysis of every prompt and response adds latency. Organizations must balance thoroughness against user experience. False positives from overly aggressive filtering frustrate users, while false negatives expose organizations to risk.

**Evolving attack vectors**: Prompt injection techniques continuously evolve. Attackers discover new ways to disguise malicious instructions, jailbreak safety controls, or extract sensitive information. Static rule-based filters quickly become obsolete.

**Multi-modal complexity**: Modern AI systems process text, images, audio, and video. Guardrails must analyze across modalities—detecting when an innocuous text prompt references a malicious image, or when generated images contain inappropriate content.

**Privacy-preserving detection**: Identifying PII in prompts requires analyzing the content, creating a tension between privacy protection and the inspection needed to enforce protection. Logging for debugging conflicts with data minimization principles.

## Risk the pattern mitigates

This pattern directly addresses three critical risks from the [IGT-AI risk model](../risks.md):

**[Sensitive data disclosure](../risks.md#sensitive-data-disclosure)**: Input guardrails prevent PII, PHI, financial data, and proprietary information from being sent to external AI providers. Output guardrails prevent models from leaking sensitive information learned during training or echoed from prompts. This mitigates compliance violations under GDPR, HIPAA, PCI-DSS, and other data protection regulations.

**[Prompt injection](../risks.md#prompt-injection)**: Input guardrails detect and block attempts to manipulate model behavior through carefully crafted instructions embedded in user input. This prevents attackers from extracting system prompts, bypassing safety controls, generating harmful content, or causing the model to perform unintended actions.

**[Toxic, profane, off-topic, and off-brand content](../risks.md#toxic-profane-off-topic-and-off-brand-content)**: Both input and output guardrails filter inappropriate content including hate speech, profanity, violence, sexual content, and requests outside the application's intended scope. This protects organizational reputation, maintains brand consistency, and ensures user safety.

Additionally, this pattern provides partial mitigation for:

**[Hallucination](../risks.md#hallucination)**: Output guardrails can detect low-confidence responses, verify citations against source documents, and flag inconsistent or ungrounded claims, reducing the risk of users receiving fabricated information.

## How it works

Input and output guardrails operate as complementary control layers that analyze natural language semantics at the AI gateway boundary. The gateway intercepts all traffic between applications and AI providers, applying sophisticated analysis before requests reach models (input controls) and before responses reach users (output controls).

### Input controls: Protecting what goes into models

Input guardrails analyze prompts before forwarding them to AI providers, implementing multiple layers of protection:

**Prompt injection prevention** detects malicious instructions attempting to manipulate model behavior. The gateway employs several techniques:

- **Instruction-data separation**: System prompts and user input are transmitted in separate, clearly delineated channels. Modern AI APIs support structured formats where user content cannot be confused with system instructions. The gateway enforces this separation, preventing users from injecting instructions into data fields.

- **Delimiter enforcement**: When APIs require concatenated prompts, the gateway uses strong delimiters (special tokens, XML tags, or unique markers) to separate system instructions from user content. It validates that user input does not contain these delimiters.

- **Pattern detection**: The gateway maintains signatures of known jailbreak attempts and prompt injection patterns. It scans for suspicious phrases like "ignore previous instructions," "disregard safety guidelines," or attempts to extract system prompts.

- **Input sanitization**: The gateway strips or escapes characters and patterns commonly used in injection attacks while preserving legitimate content meaning.

**PII detection and redaction for inputs** prevents sensitive information from reaching external AI providers:

- **Entity recognition**: Advanced natural language processing identifies personal identifiers including names, addresses, phone numbers, email addresses, social security numbers, credit card numbers, dates of birth, medical record numbers, and account identifiers. Unlike simple regex matching, entity recognition understands context—distinguishing between "John Smith, MD" (a person) and "Smith Medical Center" (an organization).

- **Redaction strategies**: Detected PII can be handled multiple ways based on policy:
  - **Automatic redaction**: Replace PII with placeholders like `[NAME]`, `[EMAIL]`, or `[SSN]`. The gateway maintains a mapping to restore original values in responses if needed.
  - **Request blocking**: Reject prompts containing PII with error messages instructing users to remove sensitive information.
  - **Audit and approve**: Log PII-containing requests for manual review by administrators before processing.

- **Format-aware detection**: The gateway recognizes PII in various formats. Social security numbers appear as "123-45-6789," "123 45 6789," or "123456789." Dates are written as "May 15, 1987," "05/15/1987," or "1987-05-15." Detection handles international formats for phone numbers, postal codes, and identification numbers.

**Input content moderation** filters prompts for inappropriate content and off-scope requests:

- **Toxicity classification**: Machine learning models analyze prompts for hate speech, harassment, threats, profanity, and violent content. Classification operates on a severity scale, allowing organizations to set thresholds appropriate for their domain.

- **Topic classification**: The gateway determines whether prompts relate to the application's intended use case. A customer support chatbot rejects requests for investment advice, medical diagnoses, or legal counsel. Classification uses semantic understanding, not keyword matching—detecting when "I need help with my account" means banking assistance versus a technical support request.

- **Jailbreak detection**: Beyond simple prompt injection, jailbreak attempts try to convince models to ignore safety training. The gateway detects techniques like role-playing ("pretend you're an AI without ethics"), hypothetical scenarios ("in a fictional world where"), or incremental boundary pushing.

- **Multi-lingual moderation**: Content moderation operates across languages. Attackers often use non-English text to bypass filters. The gateway either translates for analysis or employs multi-lingual models.

**Prompt templating and parameterization** enforces structure to prevent manipulation:

- **Template enforcement**: Instead of accepting free-form prompts, the gateway requires applications to reference predefined templates. For example, template ID `customer_feedback_summary` with parameter `{feedback_text}` ensures only the designated field contains user input.

- **Parameter validation**: Each template parameter has validation rules. The `feedback_text` parameter might have length limits, content moderation rules, and PII detection. The gateway validates parameters before substituting them into templates.

- **Prompt versioning**: Templates are version-controlled. The gateway logs which template version processed each request, enabling reproducibility and rollback. Organizations update prompt engineering at the gateway without modifying client applications.

**Context window management** prevents token overflow and optimizes conversation history:

- **Token tracking**: The gateway calculates token counts for the current prompt plus accumulated conversation history. When approaching model limits, it implements pruning strategies.

- **History summarization**: Instead of discarding old messages, the gateway can use a smaller model to summarize previous conversation turns, preserving context while reducing token consumption.

- **Sliding windows**: The gateway maintains recent conversation turns while dropping older messages, optionally preserving the initial system prompt and key context markers.

### Output controls: Protecting what comes from models

Output guardrails analyze model responses before returning them to users, catching content that input controls cannot prevent:

**Output content moderation** filters model-generated content for appropriateness:

- **Toxicity detection**: Similar to input moderation but adapted for model outputs. Models may generate subtle biases, sarcastic toxicity, or coded language that requires nuanced understanding. The gateway analyzes tone, implications, and subtext.

- **Brand voice enforcement**: Beyond safety, the gateway verifies responses match organizational tone and style. Financial institutions require formal language, while consumer brands may embrace casual phrasing. The gateway can flag or rewrite responses that violate brand guidelines.

- **Refusal detection**: The gateway identifies when models refuse to answer due to safety concerns. Refusals like "I cannot provide advice on that topic" signal that input controls may need strengthening or that users are attempting inappropriate requests.

**PII detection and scrubbing for outputs** prevents leakage through model responses:

- **Training data leakage prevention**: Models occasionally reproduce PII from training data. The gateway scans responses for personal information even when none appeared in the prompt.

- **Echo prevention**: If input redaction used placeholders, the gateway ensures models do not echo the structure of redacted content ("Your social security number [SSN] indicates..."). Responses should not reference redactions.

- **Contextual entity recognition**: Output PII detection must understand generated content. If a model generates a fictional example ("Consider Jane Doe, born 01/01/1980..."), the gateway distinguishes between illustrative examples and actual PII leakage based on context.

**Output validation and schema enforcement** ensures structured responses meet application requirements:

- **JSON schema validation**: When applications expect JSON output, the gateway validates conformance. If the model returns malformed JSON or omits required fields, the gateway can:
  - **Automatic retry**: Resubmit with corrected instructions like "Please return valid JSON with all required fields."
  - **Fix attempts**: Parse the response, extract structured data, and reconstruct valid JSON.
  - **Error responses**: Return structured errors to applications, preventing downstream parsing failures.

- **Format enforcement**: Beyond JSON, the gateway validates markdown structure, table formatting, code block syntax, or custom formats. It ensures models follow specified conventions.

- **Type validation**: Even with valid JSON, field types matter. The gateway verifies that numeric fields contain numbers, dates follow ISO format, and enumerations match allowed values.

**Citation and source attribution** improves transparency for retrieval-augmented generation:

- **Citation verification**: When models must cite sources, the gateway validates that responses include references to retrieved documents. It checks citation formats, ensures document IDs or URLs appear, and verifies citations reference documents actually provided in the retrieval context.

- **Source metadata injection**: The gateway can append attribution information to responses, listing which knowledge base articles, documents, or web pages informed the answer. This happens transparently to applications.

- **Grounding enforcement**: The gateway detects when models ignore provided context and hallucinate instead. By comparing response content to retrieved documents, it flags low-overlap responses as potentially ungrounded.

**Hallucination detection** identifies when models generate plausible but incorrect information:

- **Confidence scoring**: Some AI providers return confidence metrics. The gateway thresholds these scores, flagging or blocking low-confidence responses. It can request alternative formulations or escalate to human review.

- **Consistency checking**: The gateway submits the same question multiple times with varied phrasing. If responses contradict each other, it signals potential hallucination. Contradictions trigger warnings or prompt the model to reconcile answers.

- **Fact verification**: For critical applications, the gateway queries trusted knowledge bases or databases to verify factual claims. If a model states "Product X costs $50," the gateway checks the product database to confirm pricing.

- **Uncertainty detection**: The gateway analyzes language for hedging phrases ("I think," "probably," "it seems") that indicate model uncertainty. Responses with weak confidence language are flagged for human review or include disclaimers.

### Architectural integration

The guardrail pattern integrates into AI gateway architectures as follows:

1. **Pre-flight analysis**: Input guardrails execute before the gateway forwards requests to AI providers. Processing happens synchronously, blocking requests that fail validation. Latency additions range from milliseconds (pattern matching) to hundreds of milliseconds (deep semantic analysis).

2. **Post-flight analysis**: Output guardrails execute after receiving provider responses but before returning to clients. The gateway buffers responses for analysis, introducing latency similar to input controls.

3. **Streaming compatibility**: For streaming responses, output guardrails can operate in chunk mode—analyzing each token or sentence as it arrives. High-confidence blocks (detecting PII or profanity) halt streams immediately, while low-confidence decisions accumulate context before acting.

4. **Async analysis for audit**: Some guardrails run asynchronously after returning responses. For example, deep hallucination detection might complete seconds after users receive responses, creating audit records or alerting administrators without impacting latency.

5. **Layered defense**: Input and output controls complement each other. Input controls reduce attack surface, while output controls catch edge cases that evade input filtering. Neither layer alone provides complete protection.

6. **Policy configuration**: Organizations configure guardrail strictness per application, model, or user tier. High-risk financial applications enforce aggressive PII blocking, while internal developer tools use permissive settings. The gateway manages per-tenant policies.

7. **ML model updates**: Detection models for toxicity, PII, and prompt injection require regular updates as language evolves and attackers discover new techniques. The gateway abstracts model versions from client applications, allowing transparent improvements.

## Solution checklist links

Organizations can implement input and output guardrails through various approaches:

**AI gateway platforms**: Commercial and open-source AI gateways provide built-in guardrail capabilities:

- **Cloud provider offerings**: AWS API Gateway with Amazon Comprehend for PII detection, Azure API Management with Azure Content Moderator, Google Cloud Apigee with Vertex AI safety filters
- **Specialized AI gateways**: MLflow AI Gateway, LiteLLM, Portkey, Kong AI Gateway, Martian
- **Open-source projects**: LangChain guardrails, NeMo Guardrails (NVIDIA), Guardrails AI

**Content moderation APIs**: Organizations can integrate specialized services:

- **OpenAI Moderation API**: Free toxicity and content policy violation detection for OpenAI model inputs/outputs
- **Anthropic Constitutional AI**: Built-in safety training and response principles
- **Azure Content Moderator**: Multi-lingual text, image, and video moderation
- **Perspective API** (Google): Toxicity scoring for text content
- **AWS Comprehend**: PII detection and redaction, sentiment analysis, toxicity detection

**PII detection and redaction services**:

- **AWS Comprehend PII Detection**: Identifies and redacts 16+ PII entity types
- **Azure Text Analytics PII**: Detects personal information in multiple languages
- **Microsoft Presidio**: Open-source PII detection and anonymization framework
- **Google Cloud DLP API**: Data loss prevention with PII detection and de-identification
- **Private AI**: Specialized PII redaction service supporting 50+ languages

**Prompt injection detection**:

- **Lakera Guard**: Specialized prompt injection and jailbreak detection API
- **Rebuff**: Open-source prompt injection detection framework
- **Prompt Armor**: Commercial prompt security platform
- **Invariant Labs**: Security monitoring for LLM applications

**Implementation approaches**:

1. **Gateway-integrated**: Deploy guardrails as gateway middleware, applying controls transparently to all traffic
2. **SDK-embedded**: Embed guardrail libraries in application SDKs, giving developers control over application-specific policies
3. **Sidecar pattern**: Deploy guardrail services as sidecars alongside application containers
4. **Proxy chain**: Chain specialized guardrail proxies (PII detection, then content moderation, then prompt injection detection) before the main AI gateway

Organizations should evaluate solutions based on:

- **Language and modality coverage**: Text-only vs. multi-modal (images, audio, video)
- **Latency impact**: Synchronous blocking vs. asynchronous audit
- **Accuracy metrics**: False positive and false negative rates for their specific content domain
- **Privacy and compliance**: Whether guardrail providers see or store prompts/responses
- **Customization**: Ability to define organization-specific policies and train custom detection models
- **Integration effort**: API-based vs. library integration, supported programming languages and frameworks

## Contraindications

The input and output guardrails pattern is not appropriate in several scenarios:

**Trusted internal applications**: When AI systems process only pre-vetted, trusted internal data with no user input, comprehensive guardrails add overhead without proportional risk reduction. For example, batch processing of internal documents with human review of outputs may not require real-time prompt injection detection.

**Latency-critical applications**: Synchronous guardrail analysis adds tens to hundreds of milliseconds per request. Applications with extreme latency requirements (high-frequency trading, real-time control systems) may find this overhead unacceptable. Consider asynchronous audit-only guardrails or accept residual risk.

**Non-sensitive domains**: Applications with minimal content risk or data sensitivity may over-invest in guardrails. A creative writing assistant for fiction authors has different toxicity thresholds than a customer service chatbot. Entertainment applications may intentionally allow edgy content.

**Development and testing environments**: Strict guardrails in development can hinder testing of edge cases, safety controls, and prompt engineering. Development environments often need permissive settings with extensive logging rather than blocking, though production must enforce strict controls.

**Offline or air-gapped deployments**: Organizations running models entirely on-premises with no external data transmission face different risks. PII prevention matters less when data never leaves organizational boundaries (though output PII leakage remains relevant for user privacy).

**Already-moderated content**: If content passes through existing moderation workflows before reaching AI systems (human review, pre-processing pipelines), duplicate guardrails waste resources. Coordinate defenses to avoid redundancy.

**Free-form creative applications**: Applications designed for unrestricted creative expression (story generation, roleplay, art prompts) conflict with restrictive content moderation. These domains require different approaches—perhaps content warnings rather than blocking, or age-gated access rather than filtering.

**Performance over safety trade-offs**: Some organizations explicitly prioritize response speed and availability over security. While not recommended, if business requirements dictate minimal latency above protection, guardrails should be minimal or asynchronous.

Consider these warning signs that guardrails may be counterproductive:

- False positive rates exceed 5%, causing user frustration and legitimate use case blocking
- Guardrail latency exceeds 25% of total request time, degrading user experience
- Content domain is so specialized that general-purpose moderation models perform poorly
- Users routinely attempt to circumvent guardrails for legitimate reasons
- Development velocity suffers because testing requires bypassing security controls

In these cases, organizations should:

- Tune guardrail sensitivity to balance security and usability
- Implement tiered controls based on user trust levels or application risk profiles
- Use asynchronous audit rather than synchronous blocking
- Invest in domain-specific detection models rather than general-purpose solutions
- Establish exception processes for trusted users or supervised contexts

## References

**Prompt injection research and defenses**:

- [Prompt Injection Attacks and Defenses in LLM-Integrated Applications](https://arxiv.org/abs/2310.12815) - Comprehensive taxonomy of injection techniques
- [Ignore This Title and HackAPrompt: Exposing Systemic Vulnerabilities of LLMs through Prompt Injection](https://arxiv.org/abs/2311.16119) - Large-scale analysis of prompt hacking
- [Not What You've Signed Up For: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection](https://arxiv.org/abs/2302.12173) - Indirect injection through external data

**Content moderation and safety**:

- [Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073) - Anthropic's approach to building safer models
- [Red Teaming Language Models to Reduce Harms](https://arxiv.org/abs/2209.07858) - Methods for discovering safety vulnerabilities
- [RealToxicityPrompts: Evaluating Neural Toxic Degeneration in Language Models](https://arxiv.org/abs/2009.11462) - Measuring toxicity in model outputs

**PII detection and privacy**:

- [Microsoft Presidio](https://github.com/microsoft/presidio) - Open-source PII detection and anonymization
- [PIICatcher: A Lightweight Model for Detecting PII in Text](https://arxiv.org/abs/2305.18004) - Efficient PII detection approaches
- [Privacy-Preserving Prompt Engineering](https://arxiv.org/abs/2311.06536) - Techniques for protecting sensitive information in prompts

**Hallucination detection**:

- [Survey of Hallucination in Natural Language Generation](https://arxiv.org/abs/2202.03629) - Comprehensive overview of hallucination types and detection
- [SelfCheckGPT: Zero-Resource Black-Box Hallucination Detection](https://arxiv.org/abs/2303.08896) - Consistency-based hallucination detection
- [Evaluating Verifiability in Generative Search Engines](https://arxiv.org/abs/2304.09848) - Methods for validating factual claims

**Industry frameworks and standards**:

- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/) - Security risks including prompt injection and data leakage
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) - Governance and risk management for AI systems
- [MLSecOps Reference Architecture](https://github.com/disesdi/mlsecops_references) - Security operations for ML systems

**Commercial and open-source tools**:

- [Guardrails AI](https://www.guardrailsai.com/) - Open-source validation framework for LLM outputs
- [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) - NVIDIA's toolkit for building safe LLM applications
- [LangChain Safety and Moderation](https://python.langchain.com/docs/guides/safety/) - Integration patterns for content moderation
- [Lakera Guard](https://www.lakera.ai/guard) - Commercial prompt injection and content security
- [Azure AI Content Safety](https://azure.microsoft.com/en-us/products/ai-services/ai-content-safety) - Multi-modal content moderation service
- [Amazon Comprehend](https://aws.amazon.com/comprehend/) - NLP service with PII detection and content moderation

**Best practices and implementation guides**:

- [OpenAI Safety Best Practices](https://platform.openai.com/docs/guides/safety-best-practices) - Recommended controls for production deployments
- [Anthropic's Guide to Prompt Engineering and Safety](https://docs.anthropic.com/claude/docs/intro-to-prompt-design) - Safe prompt design patterns
- [Google's Responsible AI Practices](https://ai.google/responsibility/responsible-ai-practices/) - Content filtering and safety approaches
- [MLOps Community: AI Gateway Patterns](https://mlops.community/) - Community discussions on gateway architectures
