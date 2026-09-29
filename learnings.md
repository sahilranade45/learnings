# My Learning Log

## Python
*2026-09-27 15:01*

Python is a high-level, general-purpose programming language known for its simplicity, readability, and versatility. Created by Guido van Rossum and first released in 1991, it has grown to become one of the most popular programming languages in the world

## C
*2026-09-27 15:05*

c is a low level language

## MCP (Model Context Protocol) — theory
*2026-09-27 16:30*

MCP is a standardized protocol that lets AI applications connect to external tools and data sources without needing a custom integration for every model-tool pairing, cutting an M-models × N-tools problem down to roughly M+N. It's often compared to USB-C for AI apps: one universal connector instead of a different cable for every device.

Architecture is client-host-server: the host is the AI application itself (e.g. Claude Desktop, an IDE) and manages the session and security boundaries; the client lives inside the host and keeps a dedicated 1:1 connection to a single server; the server is an external program that exposes capabilities to the client. A host can connect to many servers, but each client-server pair is isolated.

Servers expose up to four kinds of primitives: Tools (functions the model can invoke to take an action, model-controlled based on name/description), Resources (read-only data the client can pull into context, like a GET endpoint), Prompts (reusable parameterized templates usually chosen explicitly by the user, e.g. a slash command), and Sampling (a less common primitive where the server can ask the client's LLM to generate a completion, borrowing the model's intelligence mid-task).

Under the hood, messages use JSON-RPC 2.0, transported either via stdio (server runs as a local subprocess, communicates over standard input/output — simple and fast for local tools) or HTTP with SSE / Streamable HTTP (for remote servers that stream responses back).

The connection lifecycle has four stages: Initialization (client and server handshake protocol versions and capabilities), Discovery (client asks what tools/resources/prompts the server offers and gets schemas back), Invocation (client calls a specific tool with arguments and gets a result), and Notifications (server can push updates, like a change in its tool list, without being asked).

This design matters because it decouples tool authors from model providers, keeps a security boundary where the host mediates what the model can see/invoke (so a compromised server can't silently take over — humans stay in the loop for consent), and makes tool use composable across many servers.

## MCP transports — stdio vs HTTP
*2026-09-29 19:09*

MCP servers can communicate with clients over two main transports: stdio and HTTP (with SSE / Streamable HTTP).

stdio (Standard Input/Output): the client launches the MCP server as a local child process, and JSON-RPC messages are exchanged by writing to the process's stdin and reading from its stdout (newline-delimited JSON). It's local only — the server runs on the same machine as the client. The server's lifecycle is tied 1:1 to the client: started when the client connects, killed when it disconnects. No network stack is needed (no ports, TLS, or auth headers), so it's simple and fast with minimal overhead. Since stdout is reserved for protocol messages, servers can freely use stderr for debug logging. Good for tools operating on local resources — filesystem access, local git commands, a local SQLite file, or wrapping a local CLI tool.

HTTP (with SSE or the newer Streamable HTTP transport): the server runs as an independent, possibly remote, long-lived process — essentially a normal web server. The client sends requests over HTTP, and the server responds either as a single JSON response or streams events back via SSE/Streamable HTTP. This is remote-capable — the server can live anywhere (own infra, cloud, a SaaS company's servers) and isn't spawned by the client, so it can serve multiple clients/sessions concurrently. It requires the usual web concerns: authentication (API keys, OAuth), TLS, rate limiting, and handling reconnects. Streaming support (SSE) lets the server push incremental updates for long-running tool calls or notifications. Good for tools wrapping external/cloud services, like a company exposing their product's API as an MCP server (e.g. Google Drive, Asana connectors).

Practical tradeoff — mental model: stdio is for "my machine, my tool" (local dev tools, filesystem, CLI wrappers, low setup complexity, no multi-client support, security inherited from OS process permissions); HTTP is for "someone else's service, exposed to many clients" (SaaS integrations, shared/hosted services, higher setup complexity due to auth/networking, supports multiple clients, needs explicit auth like tokens or OAuth).
