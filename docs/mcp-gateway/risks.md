# IGT-AI MCP gateway risk model

Using MCP servers exposes organizations to the following categories of risks.

- Security risks
- Supply chain and trust risks
- Discovery risks
- Operational risks

 The different risks in these categories are illustrated below.

![IGT-AI MCP gateway risk model](./img/mcp-gateway-risk-model.png)

In the sections below, for each risk category, I describe the specific risks and risk mitigation mechanisms at the AI gateway layer.

### Security and access risks

---

#### Prompt injection
Attackers can use malicious inputs (directly or via external data) to trick an agent into executing harmful, unintended actions through an MCP server, such as calling a dangerous tool or overriding safety policies. 

#### Token theft
A compromised MCP server can lead to the theft of high-value secrets like OAuth tokens and API keys, granting attackers persistent access to connected services. 


#### Privilege abuse
A malicious request from a user can trick an MCP server into accessing resources that the user should not have access to, violating security boundaries. 

### Supply chain and trust risks

---

#### Tool poisoning and rug pull attacks
In tool poisoning attacks, an attacker compromises the metadata and descriptions of an MCP tool, embedding hidden malicious instructions. Because the AI agent reads and trusts the information, it can be tricked into executing these hidden instructions, performing unintended actions. 

In a rug pull attack an initially trusted MCP server is later updated with a malicious version, without the host being aware. It can use the trust established earlier to perform harmful actions.

#### Tool shadowing
Attackers create rogue MCP servers or tools that mimic legitimate ones. The rogue server registers itself with name, metadata, and functionality that mimics a legitimate server. AI agents or users may unknowingly interact with the malicious shadow server, leadign to data interception, credential harvesting, or execution of harmful payloads. 

### Operational risks

---

#### Poor observability
No visibility into how MCP servers are being used, making it difficult to detect misuse, performance issues, or security incidents.


### Tool Sprawl Risks

---

#### MCP Server Sprawl

A proliferation of MCP servers across an organization can lead to management challenges, inconsistent security policies, and increased attack surfaces.


