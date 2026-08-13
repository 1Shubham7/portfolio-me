---
title: "MCP Explained: What It Is and Why It Exists"
description: "A breakdown of the Model Context Protocol - the problem it solves, its architecture, and how a tool call actually works under the hood."
dateString: July 2026
draft: false
tags: ["MCP", "AI", "DevOps"]
weight: 3
cover:
    image: "/blog/mcp/mcp.png"
---

MCP (Model Context Protocol) is an open standard for connecting AI apps to external tools and data. Here's what it actually is, why it exists, and how it works under the hood.

## The problem it solves

Before MCP, every AI app that wanted to talk to an external tool needed its own custom integration for that tool. Claude needed a GitHub integration. Cursor needed its own GitHub integration. ChatGPT needed another one. Same tool, rebuilt separately for every app.

MCP standardizes the interface both sides talk to. Build a GitHub integration once, as an MCP server, and every MCP-compliant app can use it without extra work. Someone still has to build the server - but once, not once per app.

MCP was introduced by Anthropic in November 2024, created by David Soria Parra and Justin Spahr-Summers. It's inspired by LSP (Language Server Protocol) - the thing that lets `gopls` work identically across VS Code, Neovim, and JetBrains without each editor writing Go-specific code. MCP does the same thing for AI apps and tools. In December 2025, Anthropic donated MCP to the Agentic AI Foundation under the Linux Foundation, so it's now a vendor-neutral standard, not an Anthropic-owned one.

## Host, Client, Server

MCP has three roles:

- **Host** - the AI application you're using. Claude Desktop, Cursor, ChatGPT.
- **Client** - lives inside the host, one instance per connected server, handles the actual message passing.
- **Server** - the program exposing tools and data. It doesn't know or care which host it's plugged into, it just responds to messages.

A host spawns a separate client for every server it connects to.

## JSON-RPC and stdio

MCP messages are JSON-RPC 2.0 - just a spec for what a request and response look like:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": { "name": "list_pods", "arguments": { "namespace": "kube-system" } }
}
```

That's the message format. `stdio` is how the bytes actually travel for a local server. The host spawns the server as a subprocess and pipes messages through its stdin/stdout - the same stdin/stdout any CLI program uses. No networking, no ports. It's the same mechanism as `cat file.txt | grep something` - the shell wires one process's stdout to another's stdin using a kernel pipe. The host does this programmatically instead of a shell doing it.

The server's code doesn't change because of this. It's just: read a line from stdin, do the work, write a line to stdout, loop.

## The three primitives

MCP servers expose three kinds of things:

**Tools** - actions the model calls with parameters, on its own, mid-conversation. Something like `list_pods` or `get_pod_logs`. The model decides when to call them.

**Resources** - static-ish content the user attaches, like uploading a file. A server could expose something like `k8s://cluster/pods-snapshot` as a Resource. No parameters, no autonomous calling - the user picks it from a list, same as attaching a PDF to a chat.

**Prompts** - reusable instruction templates the user triggers, usually as a slash-command. A server could expose `/diagnose-crashloop`, which expands into a full, well-written diagnostic prompt so the user doesn't have to write it themselves.

Quick way to remember it: **Tools are verbs, Resources are nouns, Prompts are pre-packaged sentences.**

## The handshake

Before any tool can be called, client and server negotiate:

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: initialize (protocol version, capabilities)
    S->>C: initialize result (server capabilities)
    C->>S: initialized (notification)
    C->>S: tools/list
    S->>C: [list_pods, describe_pod, ...]
    Note over C,S: Now the model can call tools
    C->>S: tools/call (list_pods)
    S->>C: result: pod names
```

This whole exchange happens before you type a single message - it's why Claude Desktop shows a brief "connecting" state when it spawns a server.

## From Claude to the MCP server: a full tool call, end to end

The handshake explains how the client and server talk. What it doesn't explain is where the model fits in, and this is the part most MCP diagrams skip: **the model never talks to the MCP server. It never even knows MCP exists.**

The model is an HTTP API. The host sends it a request containing the conversation plus a list of tool definitions, and gets back text and, sometimes, a request to call a tool. That's the entire interface. MCP lives on the other side of the host. The host is the bridge between two completely separate protocols: the LLM API on one side, JSON-RPC-over-stdio on the other.

Here's what actually happens when you type "how many pods are running in kube-system?" into Claude Desktop with a Kubernetes MCP server connected:

**1. The host translates MCP tools into API tool definitions.** At connection time the client already did `tools/list` and got back every tool with its name, description, and JSON Schema. The host converts these into the `tools` array of the LLM API request. From the model's point of view, an MCP tool is indistinguishable from any other tool - it's just a name, a description, and a schema.

**2. The host sends your message plus the tool list to the model.** One HTTP request to the LLM API: the conversation so far, plus the tool definitions.

**3. The model decides to call a tool.** It can't run anything itself. What it does is stop generating and return a response whose `stop_reason` is `tool_use`, containing a `tool_use` block:

```json
{
  "stop_reason": "tool_use",
  "content": [
    { "type": "text", "text": "Let me check the pods in kube-system." },
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "list_pods",
      "input": { "namespace": "kube-system" }
    }
  ]
}
```

This is where the tool description from your server earns its keep. The model picked `list_pods` and filled in `{"namespace": "kube-system"}` purely from the name, description, and schema. It has never seen the server's code.

**4. The host routes the call to the right client.** The host may have five servers connected, each with its own client. It looks up which server owns `list_pods` and hands the call to that client. This is also where the "allow this tool to run?" popup happens - it's the host pausing before forwarding, not anything in the protocol.

**5. The client sends `tools/call` over stdio.** Now, and only now, does MCP traffic happen: the JSON-RPC request from earlier in this post goes down the server's stdin, the server runs its handler (the actual `clientset.CoreV1().Pods(...)` call against your cluster), and writes the result to stdout.

**6. The host feeds the result back to the model.** The tool result goes into the conversation as a new message containing a `tool_result` block whose `tool_use_id` matches the `id` from step 3, and the host makes a second HTTP request to the LLM API:

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "{\"pods\": [{\"name\": \"coredns-5d78c9869d-xk2vp\", \"status\": \"Running\", ...}]}"
    }
  ]
}
```

**7. The model writes the answer.** With the result now in its context, it generates "There are 12 pods running in kube-system..." and stops with `stop_reason: end_turn`. If it instead decides it needs more information - say, logs from one of those pods - it returns another `tool_use` block and the loop goes back to step 4. The host keeps looping until the model stops asking for tools.

```mermaid
sequenceDiagram
    participant U as You
    participant H as Host (Claude Desktop)
    participant M as Model (LLM API)
    participant S as MCP Server
    U->>H: "pods in kube-system?"
    H->>M: conversation + tool definitions
    M->>H: tool_use: list_pods
    H->>S: tools/call (JSON-RPC over stdio)
    S->>H: result: pod list
    H->>M: conversation + tool_result
    M->>H: "There are 12 pods..."
    H->>U: answer
```

Two things fall out of this that are worth internalizing:

- **Every tool call costs two model requests minimum**, and the entire conversation (including every previous tool result) is re-sent each time, because the LLM API is stateless. This is why returning trimmed-down results from your server matters so much - a bloated result gets paid for again on every subsequent request in the conversation.
- **The security boundary is the host, full stop.** The model produces a request to call a tool; whether that call actually reaches your server is entirely the host's decision. The MCP server can't tell whether the call was approved by a human or fired automatically, and the model can't tell whether the call was executed or blocked - it only sees whatever comes back as the `tool_result`.

## Building one in Go

The [official Go SDK](https://github.com/modelcontextprotocol/go-sdk) handles the protocol part - framing, the handshake, generating schemas - so a working server is less code than you'd expect. Here's the shape of it, using the Kubernetes server I built as the example.

You create a server, register some tools on it, and run it over a transport. For a local server that transport is stdio:

```go
func main() {
    server := mcp.NewServer(&mcp.Implementation{
        Name:    "kubeaid-mcp",
        Version: "0.1.0",
    }, nil)

    registerTools(server)

    // Run blocks, reading requests from stdin and writing replies to stdout.
    if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
        log.Fatal(err)
    }
}
```

One thing to get right before anything else: your logs go to stderr, not stdout. stdout is the protocol's channel, and a single stray line there breaks the JSON framing. It's the first way a new server usually falls over.

A tool is a Go struct for its input plus a function. You don't write JSON Schema by hand - you tag the struct fields, and the SDK reflects them into the schema the model reads:

```go
type listPodsInput struct {
    Namespace string `json:"namespace,omitempty" jsonschema:"namespace to list pods in; omit for all namespaces"`
}

type listPodsOutput struct {
    Pods []podSummary `json:"pods"`
}
```

The handler gets the typed input and returns the typed output:

```go
func listPods(ctx context.Context, req *mcp.CallToolRequest, in listPodsInput) (*mcp.CallToolResult, listPodsOutput, error) {
    pods, err := clientset.CoreV1().Pods(in.Namespace).List(ctx, metav1.ListOptions{})
    if err != nil {
        return nil, listPodsOutput{}, err
    }

    var out listPodsOutput
    for _, p := range pods.Items {
        out.Pods = append(out.Pods, podSummary{
            Name:     p.Name,
            Status:   podStatus(&p),      // derived, not just p.Status.Phase
            Restarts: totalRestarts(&p),
        })
    }
    return nil, out, nil
}
```

Note what it returns - not the raw pod objects. A single pod is hundreds of lines of YAML, and handing all of it back buries the few fields the model actually needs under a wall of noise it has to pay for in tokens. Deciding what to leave out is most of the work in writing a good tool.

Registering it wires the input and output types to the handler:

```go
mcp.AddTool(server, &mcp.Tool{
    Name:        "list_pods",
    Description: "List pods in a namespace, with status, restart count and age.",
}, listPods)
```

That description does more than it looks like. The model picks which tool to call from the name and description alone - it never sees your code - so a vague one gets skipped or misfired.

Because it's all just JSON over stdin/stdout, you can test the whole thing from a terminal, no AI app involved:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"cli","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_pods","arguments":{"namespace":"kube-system"}}}' \
  | ./kubeaid-mcp
```

Watching the raw request and response go by once takes most of the mystery out of MCP. (One gotcha: piping everything in at once closes stdin immediately, and the server can exit on EOF before it replies. A real client holds the connection open - a small script that writes a message, reads the reply, then writes the next behaves the same way.)

That's the entire loop. Prompts and resources hang off the same server with `AddPrompt` and `AddResource`, but tools are where most of the value is.

## Security - what MCP does not do

MCP is a message format, not a security system. The spec says hosts must get user consent before invoking a tool, but it can't enforce that - it's guidance, not a runtime guarantee. That "allow this tool to run?" popup you see in Claude Desktop is the host's own UI layer, not part of the protocol itself.

This matters more the closer a tool gets to mutating state. A read-only tool going wrong is annoying; a tool that can delete or modify things going wrong is a production incident. A malicious or poorly-described tool from some other server in the same session can also try to manipulate the model into calling tools it shouldn't (prompt injection via tool descriptions) - worth keeping in mind when pulling in third-party MCP servers.

The ecosystem is actively hardening this. Enterprise-Managed Authorization is now stable and being adopted by Anthropic, Microsoft, and Okta, and the next spec revision moves authorization closer to standard OAuth 2.0/OIDC instead of every host inventing its own flow.
