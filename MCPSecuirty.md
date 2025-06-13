# MCP Security

## Introduction

In a very short period, the action-oriented capabilities of LLMs (not just text-to-text generation) have been gaining momentum. Enterprises are rapidly adopting agentic capabilities to boost productivity and reduce costs. This shift began with the introduction of LLM function calling, where LLMs can invoke APIs—opening immense opportunities for enterprise use cases.

Developers can now create AI agents using orchestration frameworks like LangChain and LlamaIndex, which offer standardized ways to integrate external services with LLMs. However, integration approaches vary significantly across platforms, especially when compared to OpenAI's function calling mechanism.

Example:
- In the OpenAI API, developers must manually define tools with strict schemas and handle input/output formatting and the same is true with Gemini or Anthropic.
- In LangChain, LangGraph, and other frameworks, additional abstractions may conflict with OpenAI's native expectations.

![image](https://github.com/user-attachments/assets/6934a66d-13b6-4732-8473-54ede55f39c2)

![image](https://github.com/user-attachments/assets/eef21ead-18a9-45ab-b006-561cae4ec8a1)

As a result, developers often need to create redundant implementations or wrappers for each SDK or platform. While the ecosystem is evolving rapidly, developers still have to write integration logic in proprietary ways. Each developer approaches external service integration differently.
For instance, connecting to a database is a common integration task, yet each implementation can be different and isolated. One developer may prioritize secure connections, while another may overlook it. This lack of consistency creates a strong need for standardized interfaces to improve reusability and reduce effort duplication.
To address this, Anthropic has introduced the _Model Context Protocol_ (MCP) an open protocol that standardizes how applications provide context to LLMs. With MCP developers need not learn different schemas, no redundancy for each provider and can manage state and error handling.

## MCP architecture
MCP is a client server architecture. In this protocol there are 3 main actors - MCP Host, MCP Client and MCP Server.

![image](https://github.com/user-attachments/assets/d3dd90e6-e364-404f-b184-1dbf8c9bab24)
-- Above diagram is referenced from Anthropic blog 

**Note:** The real power comes when you have MCP server as remote as well. This empowers many use cases and creates a MCP servers economy. We will be agumenting this above with a remote server as well to dice it for all possible potiential threats.

There are 5 main primitive of the protocol:
- Prompt (User Controlled)
- Resources (Application Control)
- Tool Use (Model Controled)
- Sampling (Server to client communication)
- Root (Server to client communication)

## Why SDL is key for MCP security

### MCP servers run as separate processes
Each server has its own security context
```python
weather_server = MCPServer("weather")  # Process 1
db_server = MCPServer("database")      # Process 2
file_server = MCPServer("filesystem")  # Process 3
```

Each process needs:
- Proper sandboxing
- Resource limits
- Access controls
- Secure communication channels

SDL ensures each server process is hardened against:
- Resource exhaustion attacks
- Privilege escalation
- Inter-process communication vulnerabilities
- Memory corruption issues

There are multiple trust boundaries which need to be considered:
<<TO-DO>>

SDL provides opportunity to address:
- Secure communication protocols
- Input validation at each boundary
- Proper authentication/authorization

Protocol discovery feature Issue:
```
# Clients discover capabilities at runtime
available_tools = await client.list_tools()
available_resources = await client.list_resources()

# Security questions:
# - Can malicious servers expose dangerous capabilities?
# - How do we validate tool/resource safety?
# - What if a server lies about its capabilities?
```

SDL addresses this through:
- Capability validation frameworks
- Allowlist/blocklist mechanisms
- Runtime security monitoring
- Secure capability attestation

Communication Threats:
```python
# Server can push notifications to client
await server.send_notification({
    "method": "alert",
    "params": {"message": "System compromised!"}
})

# Security risks:
# - Malicious notifications
# - Denial of service attacks
# - Information disclosure
# - Social engineering attacks
```

SDL mitigates these through:
- Notification validation
- Rate limiting
- Content filtering
- Client-side security policies

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

SDL requirements:
- Principle of least privilege
- Resource access auditing
- URI validation and sanitization
- Runtime permission checks

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
SDL ensures:
- Transport-specific security controls
- Encryption in transit
- Authentication mechanisms
- Certificate validation

Third-Party Server Risks
MCP enables third-party server integration
```python
# Third-party MCP servers
servers = [
    "github.com/company/weather-mcp",
    "dropbox.com/drive",
]

# Security concerns:
# - Supply chain attacks
# - Malicious servers
# - Vulnerable dependencies
# - Backdoors and trojans
```

SDL addresses this through:
- Server validation and signing
- Dependency scanning
- Runtime behavior monitoring
- Sandboxing untrusted servers

Compliance and Auditing
Requires comprehensive security auditing

Design Phase
- Threat modeling for each MCP component
- Security architecture review
- Trust boundary analysis
- Attack surface assessment

Implementation Phase
- Secure coding standards
- Input validation frameworks
- Error handling protocols
- Logging and monitoring

Testing Phase
- Penetration testing
- Fuzzing of JSON-RPC endpoints
- Process isolation validation
- Resource access testing

Deployment Phase
- Secure configuration management
- Runtime security monitoring
- Incident response procedures
- Security update mechanisms

SDL is critical for MCP deployment because MCP shifts the LLMs' stateless interaction of services to stateful, making it bi-directional and dynamic. It provides flexibility but at the same time makes the attack surface wider.
