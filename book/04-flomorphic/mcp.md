# Driving FloMorphic over MCP

> **Status: outlined.**

The FloMorphic API is **itself an MCP server**, mounted at `/mcp` and on by default.

Every entity the canvas edits — workflows, contexts, prompts, memory stores, triggers,
human tasks, runs — is an MCP tool over **the same call path the web app uses**. So an AI
client can **design a workflow, save it, run it, and read back what it produced**, including
the same AI-authoring brain the canvas's *AI Build* dialog uses.

```bash
claude mcp add --transport http flomorphic http://localhost:8026/mcp
```

Claude Desktop connects by URL (Settings → Connectors → Add custom connector), or via the
`mcp-remote` bridge on classic builds.

## Sections planned

**1. Why this is architecturally interesting, not just convenient.** The product's own UI
and an external agent are peers on one API. Nothing is UI-only, which is a constraint the
product had to hold rather than a feature it added.

**2. The `flo_*` tool catalog.** What each tool covers, grouped by entity.

**3. The two directions, kept distinct.** FloMorphic *as* an MCP server (an agent drives
FloMorphic) versus FloMorphic's *MCP node* as a client (a flow drives an MCP server). Easy
to conflate; entirely different.

**4. A first design → run → inspect session.** Walked end to end.

**5. Per-client setup.** Claude Desktop, Claude Code, Cursor.

**6. Auth**, and what to consider before exposing `/mcp` beyond localhost.

## Source material

`FloMorphic/getting-started/docs/mcp.md`, `flomorphic-api/mcpserver/`, and the blog post
*FloMorphic Is Now an MCP Server — Drive It From Claude or Any Agent*.
