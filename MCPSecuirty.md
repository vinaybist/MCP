
# MCP Security

## Introduction

In a very short period, the action-oriented capabilities of LLMs (not just text-to-text generation) have been gaining momentum. Enterprises are rapidly adopting agentic capabilities to boost productivity and reduce costs. This shift began with the introduction of LLM function calling, where LLMs can be actionable or invoke APIs, opening immense opportunities for enterprise use cases.

Developers can now create AI agents using orchestration frameworks like LangChain and LlamaIndex, which offer standardized ways to integrate external services with LLMs. However, integration approaches vary significantly across platforms, especially when compared to OpenAI's function calling mechanism.

Example:
- In the OpenAI API, developers must manually define tools with strict schemas and handle input/output formatting, and the same is true with Gemini or Anthropic.
- In LangChain, LangGraph, and other frameworks, additional abstractions may conflict with OpenAI's native expectations.

![image](https://github.com/user-attachments/assets/6934a66d-13b6-4732-8473-54ede55f39c2)

![image](https://github.com/user-attachments/assets/eef21ead-18a9-45ab-b006-561cae4ec8a1)

As a result, developers often need to create redundant implementations or wrappers for each SDK or platform. While the ecosystem is evolving rapidly, developers still have to write integration logic in proprietary ways. Each developer approaches external service integration differently.

For instance, connecting to a database is a common integration task, yet each implementation can be different and isolated. One developer may prioritize secure connections, while another may overlook it. This lack of consistency creates a strong need for standardized interfaces to improve reusability and reduce effort duplication.

To address this, Anthropic has introduced the _Model Context Protocol_ (MCP), an open protocol that standardizes how applications provide context to LLMs. With MCP, developers need not learn different schemas, have no redundancy for each provider, and can manage state and error handling.

## MCP architecture

MCP is a client-server architecture. In this protocol, there are 3 main actors – MCP Host, MCP Client, and MCP Server.

- **MCP Host** – System which holds the MCP clients or home for MCP Clients or where Client code runs. For example, Desktop Computer or Mobile Phone.
- **MCP Client** – The piece of software or application which invokes Tools, inserts prompts (user-provided), and can query for Resources. For example, Claude Desktop, Cursor, or Windsurf.
- **MCP Server** – The piece of software or API which abstracts Tools logic. It exposes Resources and Prompts (code-provided).

![image](https://github.com/user-attachments/assets/d3dd90e6-e364-404f-b184-1dbf8c9bab24)  
-- Above diagram is referenced from Anthropic blog

**Note:** The real power comes when you have a remote MCP server. This empowers many use cases and creates an MCP server economy. We will be augmenting this above with a remote server (let's say MCP Server D) as well to slice it for all possible potential threats.

There are 5 main primitives of the protocol:
- Prompt (User Controlled)
- Resources (Application Controlled)
- Tool Use (Model Controlled)
- Sampling (Server-to-client communication)
- Root (Server-to-client communication)
- Notification (Server's push notification capability/notify client)

## Why SDL is key for MCP security

Our approach is to go with the primitives and try to find the potential threats and see how SDL is useful to mitigate those.

## Prompt

The source of the prompts can be the User and Server itself. In other words, Prompts can be provided and controlled by the user, but for a great experience, they can also be written on the server itself.

### Example:

These are prompts that users input directly into the MCP client (like Claude Desktop):

```... (truncated for brevity)
