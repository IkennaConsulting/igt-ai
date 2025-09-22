
# IGT-AI

A risk-based framework for evaluating runtime AI-API governance and integration efficiency. The risks mitigated by the IGT-AI framework are detailed in the
[IGT-AI risks document](risks.md).

## What runtime AI-API governance means
Runtime AI-API governance is the set of policies, controls, and monitoring mechanisms
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


## Example

Imagine an enterprise application that uses an LLM for customer support.
 At runtime, the AI–API governance layer could:

- Block prompts containing confidential customer account data.
- Ensure answers don’t include unverified financial advice.
- Add a mandatory disclaimer automatically.
- Log the interaction securely for compliance review.
