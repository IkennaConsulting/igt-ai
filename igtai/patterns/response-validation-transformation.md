
# Response validation and transformation

## The problem

AI models generate natural language responses that are inherently unstructured, non-deterministic, and variable in format. Applications, however, typically require structured data in predictable formats—JSON objects conforming to specific schemas, validated data types, or transformed content ready for downstream processing. This mismatch creates a fundamental integration challenge: developers must write extensive parsing logic, error handling, and retry mechanisms in every application that consumes AI outputs.

The non-deterministic nature of AI models means that even when explicitly instructed to return JSON, models may occasionally produce malformed output, include conversational text before or after the structured data, use incorrect field names, omit required fields, or return data types that don't match expectations. For example, a model asked to return a JSON object with a numeric `confidence` field might sometimes return the string "high" instead of a number, or wrap the JSON in markdown code fences, or add an explanatory sentence before the JSON payload.

Without centralized validation and transformation capabilities, each development team implements its own parsing logic, leading to inconsistent error handling, scattered retry strategies, duplicated code across services, and applications that break unpredictably when model outputs vary. As applications scale to consume multiple models across different use cases, this integration complexity compounds rapidly.

## Forces

Several competing concerns make AI response handling particularly challenging:

- **Schema conformance**: Applications need responses to match predefined data structures, but AI models generate free-form text that may not strictly adhere to schemas even when instructed to do so. Expecting perfect compliance from non-deterministic systems is unrealistic.

- **Format variability**: Models may return valid information in unexpected formats—JSON wrapped in markdown, XML instead of JSON, prose explanations mixed with structured data, or variations in field naming conventions that require normalization.

- **Type safety**: Strongly-typed languages and downstream systems require specific data types (integers, booleans, dates), but model outputs are fundamentally strings that may not parse correctly into the expected types.

- **Error recovery**: When responses don't meet requirements, applications must decide whether to retry with modified prompts, apply heuristic fixes, return errors to users, or fall back to default behaviors. This logic is complex and use-case dependent.

- **Latency vs. reliability trade-off**: Validating and transforming responses adds processing time, while retrying on validation failures increases latency significantly. Organizations must balance response speed against correctness guarantees.

- **Model-specific quirks**: Different models have different output tendencies—some consistently wrap JSON in code blocks, others add conversational preambles, and others handle edge cases differently. Handling these variations requires model-aware transformation logic.

- **Evolution and drift**: As models update, their output patterns may change subtly. Validation logic must be robust enough to handle variations without breaking while still catching genuine errors.

- **Transparency vs. abstraction**: Applications benefit from receiving clean, transformed data, but they may also need access to raw model outputs for debugging, auditing, or specialized processing requirements.

## Risk the pattern mitigates

This pattern primarily mitigates **[lack of resilience](../risks.md#lack-of-resilience)** by providing structured error handling and automatic retry mechanisms when AI models return malformed or schema-violating outputs. Without validation and transformation capabilities, applications crash or produce incorrect results when models behave unpredictably, leading to poor user experiences and reduced system reliability.

The pattern also addresses **[poor observability](../risks.md#poor-observability)** by centralizing response validation logging in the gateway. When validation failures occur, the gateway captures detailed information about what was expected versus what was received, which model produced the output, and how the error was resolved. This visibility is essential for diagnosing quality issues, identifying model regressions, and understanding when prompt engineering improvements are needed. Without centralized validation, response parsing failures scatter across applications, making patterns impossible to detect and root causes difficult to diagnose.

Additionally, the pattern provides early detection of hallucinations or nonsensical outputs by validating that responses contain required fields and meet basic structural requirements, offering a first line of defense against serving completely invalid model outputs to end users.

## How it works

Response validation and transformation operates as a post-processing layer within the AI gateway that sits between the model's raw output and the application's consumption of that data. The gateway defines expected response schemas, applies validation rules, transforms outputs into application-ready formats, and implements intelligent error recovery strategies when validation fails.

### Schema definition and enforcement

Applications or template definitions specify the expected structure of model responses through schema definitions. These schemas describe required fields, data types, format constraints, and validation rules. Common schema formats include:

**JSON Schema**: Standard JSON Schema specifications define object structures, required properties, type constraints, and validation rules. For example:

```json
{
  "type": "object",
  "required": ["sentiment", "confidence", "key_issues"],
  "properties": {
    "sentiment": {"type": "string", "enum": ["positive", "neutral", "negative"]},
    "confidence": {"type": "number", "minimum": 0, "maximum": 1},
    "key_issues": {"type": "array", "items": {"type": "string"}},
    "priority": {"type": "string", "enum": ["high", "medium", "low"]}
  }
}
```

**Structured output formats**: Some models support native structured output modes where the model is constrained to generate only schema-compliant JSON. The gateway validates that the model was configured correctly and that the response matches expectations.

**Type definitions**: For simpler cases, applications specify expected return types such as "boolean", "integer", "email", or "ISO date" without full schema definitions. The gateway validates and coerces responses to these types.

When a model response arrives, the gateway parses the output and validates it against the defined schema. Validation checks include:

- **Presence of required fields**: All mandatory properties are present in the response
- **Type correctness**: Fields have the expected data types (string, number, boolean, array, object)
- **Format compliance**: Strings match expected patterns (email, URL, date format, UUID)
- **Enumeration validity**: String values are within allowed sets
- **Range constraints**: Numbers fall within specified minimum and maximum values
- **Array and object structure**: Nested structures conform to schema definitions

If validation succeeds, the response proceeds to transformation. If validation fails, the gateway triggers error recovery logic.

### Response parsing and extraction

AI models often return structured data embedded within conversational text or formatted in ways that require preprocessing before validation. The gateway applies intelligent parsing strategies:

**JSON extraction**: Models frequently wrap JSON responses in markdown code fences, add explanatory text before or after the JSON, or include multiple JSON objects. The gateway uses pattern matching to extract JSON content:

```
Sure, here's the analysis you requested:

```json
{"sentiment": "positive", "confidence": 0.85}
```

Let me know if you need anything else!
```

The gateway extracts the JSON payload and discards surrounding text.

**Format conversion**: When models return data in formats other than the expected one, the gateway applies conversions. For example, if an application expects JSON but the model returns XML, the gateway parses the XML and converts it to the requested JSON structure.

**Multi-format detection**: The gateway attempts parsing in multiple formats sequentially—first as raw JSON, then as markdown-wrapped JSON, then as key-value pairs, then as structured prose—selecting the first successful parse that meets schema requirements.

**Entity extraction**: For responses that contain structured information in prose form, the gateway can apply named entity recognition or pattern matching to extract required data points. For example, parsing "The sentiment is positive with 85% confidence" into `{"sentiment": "positive", "confidence": 0.85}`.

### Response transformation

Once parsed and validated, responses undergo transformation to match application-specific format requirements:

**Format conversion**: The gateway transforms model outputs into the required content type:

- **Markdown to HTML**: Convert markdown-formatted text into HTML for web applications
- **Plain text to structured markup**: Apply formatting tags, links, and structure to plain text responses
- **JSON to XML or YAML**: Restructure data into alternative serialization formats
- **Flattening or nesting**: Reshape JSON structures to match existing application data models

**Field mapping and renaming**: Models may use different field names than applications expect. The gateway applies mapping rules:

```
Model output: {"analysis_result": "positive", "certainty": 0.85}
Transformed to: {"sentiment": "positive", "confidence": 0.85}
```

**Data normalization**: Standardize data representations:

- **Date/time formatting**: Convert various date formats to ISO 8601 or application-preferred formats
- **Case normalization**: Transform "Positive", "POSITIVE", "positive" to consistent casing
- **Unit conversions**: Apply metric to imperial conversions or currency exchanges if needed
- **Text sanitization**: Remove or escape special characters, normalize whitespace, or apply length limits

**Augmentation with metadata**: Append additional fields to responses:

```json
{
  "sentiment": "positive",
  "confidence": 0.85,
  "_metadata": {
    "model": "claude-3-5-sonnet-20250219",
    "timestamp": "2025-10-09T14:32:11Z",
    "request_id": "req_abc123",
    "tokens_used": 145
  }
}
```

This provides applications with context about response provenance, costs, and traceability without requiring modifications to model prompts.

### Error recovery and retry strategies

When validation fails, the gateway implements intelligent recovery mechanisms rather than immediately returning errors:

**Automatic retry with clarification**: The gateway resubmits the request with an augmented prompt that includes the validation error and specific instructions for correction. For example:

```
[Original prompt content]

Important: Your previous response did not include the required "confidence" field.
Please provide a JSON response with all required fields: sentiment, confidence, key_issues.
```

This allows the model to self-correct without application involvement.

**Heuristic fixes**: For common, predictable errors, the gateway applies automated repairs:

- **Type coercion**: Convert string numbers to numeric types ("0.85" → 0.85)
- **Boolean inference**: Map common phrases to boolean values ("yes"/"true" → true, "no"/"false" → false)
- **Field inference**: If a required field is missing but its value can be inferred from other fields or defaults, populate it automatically
- **Wrapper removal**: Strip markdown formatting, code fences, or conversational text automatically

**Partial validation**: When a response contains most required fields but is missing some optional ones, the gateway can return a partial success with a warning rather than failing completely. Applications receive the available data and handle missing fields according to their own logic.

**Fallback responses**: For critical use cases, the gateway can be configured with fallback responses when validation fails after retry attempts. For example, returning `{"sentiment": "neutral", "confidence": 0.5}` rather than an error when sentiment analysis fails.

**Error escalation**: If automated recovery fails, the gateway returns structured error responses to applications that include:

- The validation error details (which fields failed, why)
- The raw model output for application-level debugging
- The number of retry attempts made
- Suggestions for prompt improvements to avoid future failures

### Transparent operation modes

The gateway supports multiple operation modes to balance reliability against flexibility:

**Strict mode**: Responses must pass validation or the request fails after retry attempts. Applications receive either valid data or structured errors. This mode is appropriate for critical paths where data quality is mandatory.

**Lenient mode**: Validation warnings are logged but responses are returned even if they don't fully conform to schemas. Applications handle validation edge cases themselves. This mode supports experimentation and use cases where partial data is acceptable.

**Pass-through mode**: Raw model outputs are returned without validation or transformation, but validation results are included as metadata. Applications receive both the original response and validation diagnostics, allowing flexible handling while maintaining observability.

## Solution checklist links

Implementation considerations for response validation and transformation include:

- **[AI Gateway Definition: Output validation and schema enforcement](../ai-gateway-definition.md#output-validation-and-schema-enforcement)**: Technical details on validation mechanisms and schema enforcement approaches in gateway architectures.

- **[AI Gateway Definition: Response formatting and transformation](../ai-gateway-definition.md#response-formatting-and-transformation)**: Transformation capabilities that bridge natural language outputs and application data requirements.

- **Schema design best practices**: Define schemas that are strict enough to catch errors but flexible enough to accommodate reasonable model variations. Include optional fields for model-specific extensions.

- **Retry budget configuration**: Set maximum retry attempts per request to prevent excessive latency. Common values are 1-2 retries before failing.

- **Validation observability**: Implement metrics tracking validation success rates, retry frequencies, specific field failures, and model-specific patterns to identify prompt engineering opportunities.

- **Schema versioning**: Maintain multiple versions of response schemas to allow gradual migration as application requirements evolve without breaking existing consumers.

- **Transformation pipeline configurability**: Allow applications to configure transformation rules through gateway configuration rather than requiring code changes.

## Contraindications

Response validation and transformation is not appropriate in several scenarios:

**Raw natural language output use cases**: When applications are designed to work directly with free-form text responses—such as chatbots displaying conversational replies, content generation systems producing articles, or creative writing tools—imposing structured validation provides no value and may discard useful natural language content. In these cases, applications handle raw text appropriately without transformation.

**Streaming token-by-token responses**: Applications that stream AI responses token by token for low-latency user experiences cannot buffer entire responses for validation. While some lightweight validation (profanity filtering, PII detection) can occur during streaming, comprehensive schema validation and transformation require complete responses and would eliminate streaming benefits.

**Highly variable or exploratory outputs**: Research environments, experimentation frameworks, or agent systems where response structures are intentionally open-ended and variable would be hindered by strict validation. Validation would create false positives and unnecessary failures when outputs legitimately vary.

**Performance-critical low-latency paths**: Validation and transformation add processing overhead—typically 10-50ms for schema validation and 50-200ms for complex transformations or retries. Applications where every millisecond matters may prefer to handle raw responses and implement minimal custom validation for performance.

**Native structured output support**: Some models support native structured output modes that guarantee schema-compliant responses through model-level constraints rather than post-processing validation. When using these capabilities, gateway-level validation becomes redundant, though lightweight verification of the structured output configuration may still provide value.

When validation and transformation are not suitable, organizations should implement alternative strategies:

- **Client-side validation**: Move validation logic into application code for cases requiring flexible or performance-optimized handling
- **Schema documentation**: Provide clear specifications of expected response formats so applications can implement appropriate parsing logic
- **Error reporting**: Even without validation, capture and log response parsing failures from applications to identify prompt engineering opportunities
- **Selective validation**: Apply validation only to critical fields or subsets of responses where correctness is essential

## References

- [AI Gateway Definition: Output validation and schema enforcement](../ai-gateway-definition.md#output-validation-and-schema-enforcement) - Technical details on validation mechanisms
- [AI Gateway Definition: Response formatting and transformation](../ai-gateway-definition.md#response-formatting-and-transformation) - Transformation capabilities and approaches
- [IGT-AI Risks: Lack of resilience](../risks.md#lack-of-resilience) - Operational risks mitigated by validation
- [IGT-AI Risks: Poor observability](../risks.md#poor-observability) - Visibility improvements through centralized validation
- [JSON Schema Specification](https://json-schema.org/) - Standard for defining JSON data structures
- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) - Model-level structured output capabilities
- [Anthropic Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) - Structured responses through function calling
