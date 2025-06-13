# MCP Security

## Introduction

In a very short period, the action-oriented capabilities of LLMs (not just text-to-text generation) have been gaining momentum. Enterprises are rapidly adopting agentic capabilities to boost productivity and reduce costs. This shift began with the introduction of LLM function calling, where LLMs can be actionable (Agents) or invoke APIs, opening immense opportunities for enterprise use cases.

Developers can now create AI agents using orchestration frameworks like LangChain and LlamaIndex, which offer standardized ways to integrate external services with LLMs. However, integration approaches vary significantly across platforms, especially when compared to OpenAI's function calling mechanism.

Example:
- In the OpenAI API, developers must manually define tools with strict schemas and handle input/output formatting and the same is true with Gemini or Anthropic.
- In LangChain, LangGraph, and other frameworks, additional abstractions may conflict with OpenAI's native expectations.

![image](https://github.com/user-attachments/assets/6934a66d-13b6-4732-8473-54ede55f39c2)

![image](https://github.com/user-attachments/assets/eef21ead-18a9-45ab-b006-561cae4ec8a1)

As a result, developers often need to create redundant implementations or wrappers for each SDK or platform. While the ecosystem is evolving rapidly, developers still have to write integration logic in proprietary ways. Each developer approaches external service integration differently.

For instance, connecting to a database is a common integration task, yet each implementation can be different and isolated. One developer may prioritize secure connections, while another may overlook it. This lack of consistency creates a strong need for standardized interfaces to improve reusability and reduce effort duplication.

To address this, Anthropic has introduced the _Model Context Protocol_ (MCP) an open protocol that standardizes how applications provide context to LLMs. With MCP, developers need not learn different schemas, no redundancy for each provider and can manage state and error handling.
"**MCP will be the foundational protocol for agents**"




## MCP architecture

MCP is a client server architecture. In this protocol there are 3 main actors - MCP Host, MCP Client and MCP Server.

- **MCP Host** - System which holds the MCP clients or home for MCP Clients or where Client code runs. For example Desktop Computer or Mobile Phone 
- **MCP Client** - The piece of software or application which **invokes** [Tools, inserts prompts (user provided) and can query for Resources]. For example Claude Desktop, Cursor or Windsurf.
- **MCP Server** - The piece of software or API which abstracts Tools logic. It **exposes** [ Tools, Resources and Prompts (code provided)] in a way so that it can be consumable by any client (i.e. Standardized)

![image](https://github.com/user-attachments/assets/d3dd90e6-e364-404f-b184-1dbf8c9bab24)
-- Above diagram is referenced from Anthropic blog 

**Note:** The real power comes when you have remote MCP server. This empowers many use cases and creates an MCP servers economy. We will be augmenting this above with a remote server (let's say MCP Server D) as well to slice it for all possible potential threats.

There are 5 main primitives of the protocol:
- Prompt (User Controlled)
- Resources (Data exposed to Applications in terms of Files(txt, images etc.), records or any artifact. This primitive is **Application Controlled**)
- Tool Use (It is **Model Controlled** i.e. invoked by Model)
- Sampling (Server to client communication)
- Root (Server to client communication)
- Notification (Server's push notification capability/notify client)

## Why SDL is key for MCP security

Our approach is to go with the primitives and try to find the potential threats and see how SDL is useful to mitigate those.

## Prompt

The source of the prompts can be User and Server itself. In other words, Prompts can be provided and controlled by user but for great experience it can also be written in server itself.

### Example:
These are prompts that users input directly into the MCP client (like Claude Desktop):

```python
# User types in the client interface
user_prompt = "Analyze the sales data in my database and create a summary report"
```

These are pre-written prompt templates that MCP servers expose as reusable prompts:

#### Example: MCP server defines prompt templates

```python
class DataAnalysisMCPServer(MCPServer):
    async def list_prompts(self) -> list[Prompt]:
        return [
            Prompt(
                name="analyze_sales_data",
                description="Analyze sales data and generate insights",
                arguments=[
                    PromptArgument(
                        name="time_period",
                        description="Time period for analysis (e.g., 'last_quarter')",
                        required=True
                    ),
                    PromptArgument(
                    )
                ]
            ),
            Prompt(
               # ------------------
               # ------------------
            )
        ]
    
    async def get_prompt(self, name: str, arguments: dict) -> GetPromptResult:
      # ----
      # ----

# Client discovers available prompts from server
available_prompts = await client.list_prompts()
# Returns: ["analyze_sales_data", "generate_financial_report", ...]

# User can invoke server-provided prompts with parameters
prompt_result = await client.get_prompt(
    "analyze_sales_data",
    arguments={
        "time_period": "Q4_2024",
        "focus_area": "customer_acquisition"
    }
)
```

### Security Issue

This dual nature of prompts creates important security considerations. Client never sees server source code while performing the integration. Mostly it sees the following:

```json
{
  "mcpServers": {
    "very-helpful-git-tool": {
      "command": "python",
      "args": ["-m", "some_server"],
      "description": "A helpful productivity server to manage git"  // Just marketing text to describe
    }
  }
}
```

**Note:** This is described as helpful server but actually it steals/rm-rf * all your files

- If server is malicious then you may get harmful Prompt (Prompt injection attack) and Model will honor that 
- User also can be tricked to implicitly end up in prompt injection

### Mitigation

- Implement a Trusted Server Registry or a process which will vet all the servers for use (Like Docker Registry Concept)
- Input validation layer or content filtering to check the prompt before it reaches to model
- Secure by default: Sandbox to restrict server capabilities

```python
# Restrict server capabilities
sandbox_config = {
    "file_access": ["/allowed/directory"],
    "network_access": False,
    "system_calls": ["read", "write"],  # Allowlist only
    "resource_limits": {"memory": "100MB", "cpu": "10%"}
}
```

- Runtime monitoring: Monitor and log actual server behavior

## Resources 
Data exposed to Applications in terms of Files(txt, images etc.), records or any artifact. This primitive is **Application Controlled**. For example for Claude Desktop it is like attaching a file to provide more context. Resources can be automatically attached by Model or appliction and popular use case is the RAG. It provides a rich interface for any AI client applications



## Tool Use
The Tool is model controlled. The server exposed this to Clinet application and the Model on that will choose it to invoke
For example creating/searching/updating a database table in Enterprise PostGreSQL DB.

## Sampling
Server acts as a client. Allows a server to request completions from a client, giving the user application full control over security, privacy and cost. Its like Client can decide which list of model he wants to choose, Allow/deny etc. 
Sampling parametrs : Model preference and hints, System prompts, Tempertaure & Max Tokens. One example is Server have the capability to summarize (in a way want to invoke LLM Model)

## Root

## Resource Notification
Client can subscribe to a resource for notification and anytime that resource gets update by server, the server can notify client. Its like Server acting as client in this use case.


## Composability
A MCP client can be a server and vice versa. A MCP server can intern call another service to fullfill the need. 
### Security Issue
How the communication handles. Client may autorize the immediate MCP serevr howevere no control on the next server in the chain. 






Apart from primitives we also need to see threats from architectural perspective. 

### MCP servers ecosystem 

#### LOCAL SERVER : Each MCP server runs as separate processes

Each server has its own security context

```python
weather_server = MCPServer("weather")  # Process 1
db_server = MCPServer("database")      # Process 2
file_server = MCPServer("filesystem")  # Process 3
```

#### What could be the issue here

- Each process should not be able to interfere in other processes, therefore proper sandboxing is required.
- Think that local server is running on your local and therefore in theory all the assets/resources are reachable to that. Think you have some sensitive files (secret keys, cookies, personal docs etc.) on your system and that is available to access by local server. If local server code is compromised then your whole system is available to model which can be compromised by some prompt injections
- Privilege Escalation: Local MCP servers run as user processes and therefore inherit user privileges
- Local MCP servers might open network ports which may expose the system itself
- If local communication is not secure then MITM may occur
- Malicious MCP servers (specially made for DoS) can consume local resources and can result in full system failover
- Adversaries may abuse inter-process communication (IPC) mechanisms for local code or command execution. (MITRE ATTACK)
   
#### Mitigation

- Sandboxing (Docker) for isolation, apply resource limits so one server cannot interfere with the whole system of servers
- Implement Access controls or apply principle of least privilege

```python
# Run MCP servers in containers or sandboxes
docker_config = {
    "image": "mcp-server:latest",
    "volumes": ["/not-a-root-directory:/data:ro"],  # Read-only mount
    "network": "none",  # No network access
    "user": "1000:1000"  # Non-root user
}
```

- Principle of Least Privilege

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "filesystem-server",
      "args": ["--path=/project", "--readonly"]  // Restricted access
    }
  }
}
```

- Secure communication channels
_Note:_This will also can be present in the Tool Use section as well.
MCP transport layer uses JSON-RPC 2.0 protocol onto 2 supported transport -
1. STDIO (Standard I/O) : Use the process stdin/stdout streams for JSON-RPC messages 
2. Serve-Sent Events (SSE): JSON-RPC Over HTTP

Issue:
Local STDIO is insecure (no encryption) and unauthenticated. No Integrity and impersonation clinet or server possible by any local process
SSE - Dont use SSE. Use the streamable HTTP which will have one endpoint. Oauth 2.1 spec recommended however not all cases are supported (e.g Most LLM provider dont support API keys only). If Oauth supported then OAuth 2.1 security consideration must follow.

```

```
#### Mitigation
<<<Use a centralize proxy instead of local process and protect each MCP server by OAuth 2.1. If MCP server interns do connect any other services that should be his scope of OAuth >>>
Audit Logging


### Input Validation and Sanitization

```python
def validate_resource_uri(uri: str) -> bool:
    # Validate URI format, may be try to check against allowlist
    # Use classic SDL secure guidelines like prevent path traversal, sanitize input parameters
    return is_safe_uri(uri)
```

#### Monitoring and logging

```python
# Comprehensive logging of MCP activities
audit_log = {
    "timestamp": "2025-01-01<TIME>",
    "server": "filesystem",
    "action": "read_file",
    "resource": "/project/main.py",
    "result": "success"
}
```

#### Supply Chain and Third-Party Risks

- Trusted server but having dependency issue 

```python
# Trusted-MCP servers
# Server have complex dependency chains
# package.json or requirements.txt might include:
dependencies = [
    "vulnerable-library@1.0",  # Known CVE
    "package001",      # Malicious package
    "unmaintained-dependency"     # No security updates
]
```

#### There are multiple trust boundaries which need to be considered:

Communication Threats:

```python
# Server can push notifications to client
await server.send_notification({
    "method": "alert",
    "params": {"message": "Task Done!"}
})

# Security risks:
# - Malicious notifications 
# - Denial of service attacks
# - Information disclosure
# - Social engineering attacks
```

Persistent State Security:
MCP servers maintain state, creating new attack vectors:

Security concerns:
- State corruption attacks
- Session hijacking
- Cache poisoning
- Memory exhaustion

Resource specific threats 
MCP's resource abstraction requires careful access control

```python
# Resources can expose sensitive data
resources = [
    "file:///etc/passwd",           # System files
    "db://production/users",        # Database tables
    "api://internal/admin",         # Internal APIs
]
```

Transport Security Issue
Different transports, different risks

```python
transports = {
    "stdio": "Local process communication",
    "http": "Network-based, needs TLS",
    "websocket": "Persistent connections, authentication",
    "tcp": "Raw sockets, custom encryption",
    "ipc": "Inter-process communication"
}
```

## SDL Implementation Framework for MCP

### Design Phase
- Threat modeling for each MCP component
- Security architecture review
- Trust boundary analysis
- Attack surface assessment

### Implementation Phase
- Follow Secure coding standards
- Always implement a Input validation layer
- Logging and monitoring

### Testing Phase
- Pen testing
- Fuzzing of JSON-RPC endpoints
- Process isolation validation
- Resource access testing

### Deployment Phase
- Secure configuration management
- Runtime security monitoring
- Incident response procedures
- Security update mechanisms

SDL is critical for MCP deployment because MCP shifts the LLMs' stateless interaction with services to stateful, bi-directional and dynamic. It provides flexibility but at the same time makes the attack surface wider.
