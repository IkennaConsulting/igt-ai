
# Prompt templates

Prompt templates are a defensive mechanism that structures prompts in a way that clearly separates system instructions from user-provided content. By placing user inputs into predefined variable slots rather than allowing them to influence the prompt structure, prompt templates reduce the risk of [prompt injection](../prompt-injection.md). This separation is conceptually similar to parameterized SQL queries, which isolate query logic (code) from user-supplied values (data) to mitigate SQL injection attacks. While prompt templates significantly limit an attacker’s ability to manipulate system prompts, they should be combined with additional safeguards for comprehensive protection.
