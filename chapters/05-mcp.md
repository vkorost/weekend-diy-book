# Chapter 5: MCP -- Connecting Claude Code to Everything

## What You'll Learn

Model Context Protocol is how Claude Code reaches beyond the filesystem. It is the structured interface between an AI agent and everything else: databases, APIs, cloud services, market data feeds, internal tools, third-party platforms. Without MCP, Claude Code is capable but isolated -- a very smart process that can read files and run bash commands. With MCP, it becomes a node in your infrastructure.

But MCP is not plug-and-play. Every MCP server you connect costs context tokens at session start. Connections fail silently mid-session. Tools disappear without warning. Background subagents cannot use MCP at all. The difference between an MCP integration that works and one that quietly degrades your session is configuration discipline, architectural awareness, and an honest understanding of what the protocol costs you.

What follows is the mechanics of MCP in Claude Code: how tool loading works, what it costs, how to configure it for teams, where it breaks, and how practitioners have built production-scale integrations with dozens of tools. You will learn how to reference MCP resources with the `@server:resource` syntax, how managed settings control which servers your organization permits, and how hook patterns like `mcp__server__tool` enable fine-grained logging, validation, and transformation. You will see a sovereign wealth fund running MCP integrations across thousands of portfolio managers, a production trading platform with 41 MCP tools and sub-10ms GPU-accelerated inference, and a marketing team that built an MCP server for campaign analytics in days. When MCP is the right choice over raw bash, and what a real-world connector architecture looks like at scale.

---

## How MCP Tool Loading Works

When a Claude Code session starts, every configured MCP server connects and exposes its tool definitions. These definitions -- names, descriptions, parameter schemas -- load into the context window immediately. This is the cost of MCP: you pay context tokens for every tool definition before you use any of them.

The tool search system mitigates this. Enabled by default, it loads tool definitions for up to roughly 10% of context capacity directly. The rest are deferred -- their descriptions are indexed but their full schemas are not held in context until Claude determines it needs them. When Claude encounters a task that might require a deferred tool, it searches the index, loads the relevant schema, and proceeds.

This deferred loading is the reason MCP can scale to dozens of servers without immediately consuming your entire context window. But it introduces a subtle cost: the search step itself uses context and adds latency. For tools you know you will use every session, eager loading is better. For a catalog of 40 tools where you use 5 per session, deferred loading is essential.

You can control this with the `ENABLE_TOOL_SEARCH` environment variable. Disabling it forces all tool definitions to load eagerly, which is fine if you have a small number of MCP servers and want zero search latency. It is catastrophic if you have many servers with extensive tool catalogs.

### Measuring Your MCP Context Cost

The `/mcp` slash command shows per-server token costs. Run it. If your MCP servers are consuming 15% of your context window before you type a single prompt, you have a configuration problem. Either reduce the number of connected servers, or ensure tool search is deferring most of them.

The context cost is not just the tool definitions. Each tool's JSON schema -- parameter types, descriptions, enums, nested objects -- adds up. A well-documented MCP tool with rich parameter descriptions costs more context than a terse one. If you are building MCP servers for your team, keep schemas precise but concise. Every word in a parameter description is a token Claude pays for on every request.

## MCP Resources

MCP servers can expose not just tools but also resources -- structured data that you reference using `@` mentions, the same way you reference files. The syntax is `@server:protocol://resource/path`:

```
> Can you analyze @github:issue://123 and suggest a fix?

> Please review the API documentation at @docs:file://api/authentication

> Compare @postgres:schema://users with @docs:file://database/user-model
```

Type `@` in your prompt to see available resources from all connected MCP servers. Resources appear alongside files in the autocomplete menu. When you reference a resource, it is fetched automatically and included as an attachment in the conversation.

This matters because it puts external data on the same footing as local files. Instead of asking Claude to "use the database MCP tool to fetch the users table schema," you reference it directly as `@postgres:schema://users` and it appears in context. Multiple resources can be referenced in a single prompt. The model sees them as structured inputs, not as tool call outputs, which often produces better responses because the data is present before reasoning begins rather than being fetched mid-conversation.

## Centralized Configuration

MCP servers are configured in `.mcp.json` at the project root. This file should be committed to version control. One person figures out the MCP configuration; the whole team benefits.

```json
{
  "mcpServers": {
    "analytics": {
      "command": "node",
      "args": ["./mcp-servers/analytics/index.js"],
      "env": {
        "DB_CONNECTION": "${ANALYTICS_DB_URL}"
      }
    }
  }
}
```

Environment variable interpolation lets the configuration reference credentials without embedding them. The `.mcp.json` file contains the structure; actual secrets live in each developer's environment.

### Managed MCP: Allowlists, Denylists, and Auto-Approval

For enterprises, managed settings can restrict which MCP servers are permitted. An allowlist (`allowedMcpServers` in managed settings) specifies the only servers that may be configured. A denylist (`deniedMcpServers`) blocks specific servers. This prevents developers from connecting arbitrary external services to their Claude Code sessions -- a meaningful control when the organization has data classification policies.

The allowlist/denylist operates at the configuration level. If a server is not on the allowlist, Claude Code will not connect to it, regardless of what appears in the project's `.mcp.json`. This is the same override pattern as the permission scope hierarchy described in Chapter 2: managed settings win.

Three user-level settings control the approval flow for project-defined MCP servers:

**`enableAllProjectMcpServers`** automatically approves all MCP servers in project `.mcp.json` files. Set this to `true` when you trust the repositories you work in and want zero friction. Set it to `false` (the default) when you want to approve each server individually.

**`enabledMcpjsonServers`** is a list of specific server names to auto-approve: `["memory", "github"]`. Servers on this list connect without prompting. Servers not on this list still prompt for approval.

**`disabledMcpjsonServers`** is a list of specific servers to reject: `["filesystem"]`. Servers on this list never connect, even if the project `.mcp.json` defines them.

These settings matter for organizational deployment. A team can commit `.mcp.json` to version control with all the servers the project needs, then configure managed settings to allowlist those servers and denylist everything else. New team members clone the repo, get the MCP configuration automatically, and the managed settings ensure no rogue servers connect.

## Where MCP Breaks

MCP connections are not as reliable as local tool calls. Understanding the failure modes prevents confusion when things go silent.

### Silent Disconnection

An MCP server can disconnect mid-session. When it does, the tools it provided simply disappear. Claude Code does not throw an error. It does not notify you. The tools are just gone. If Claude was relying on a tool from that server, it will either fail to find it or attempt a fallback strategy that may not be what you wanted.

The symptom is subtle: Claude stops using a capability it was using moments ago, or it starts doing something manually that it was doing through a tool. If Claude suddenly begins writing raw HTTP requests instead of using your API connector, check your MCP connections.

The `/mcp` command shows current connection status. Make it a reflex when behavior changes unexpectedly.

### Server Startup Failures

MCP servers can fail to start when a session begins. A missing dependency, a bad environment variable, a port conflict -- any of these can prevent a server from initializing. Claude Code will start the session without those tools, and you may not notice their absence until you need them.

The diagnostic path: run `/mcp` at session start. Confirm all expected servers are connected and their tool counts match expectations. This takes five seconds and prevents thirty minutes of confusion.

### Tools with Identical Names

If two MCP servers expose tools with the same name, behavior is undefined. One will shadow the other. Which one wins may depend on server initialization order, which you do not control. Name your tools with server-specific prefixes to avoid collisions.

## MCP in Subagents

MCP tools are available in foreground subagents but not background subagents. This is a hard architectural constraint, not a configuration option.

Background subagents run concurrently and cannot interact with the user. MCP servers may require interactive authentication, may have rate limits that conflict with parallel access, and may produce output that needs human review. Rather than introducing partial support with unpredictable behavior, the constraint is binary: foreground gets MCP, background does not.

The practical impact: if you are designing a workflow where subagents need to query external services, those subagents must run in the foreground. As covered in Chapter 4, this affects your parallelism strategy -- foreground subagents block the main conversation.

The workaround for background subagents that need external data: have the main agent fetch the data via MCP first, then pass the results to the background subagent as part of its task description. This trades real-time data access for parallelism.

## MCP in Plugins

Plugins can bundle MCP server configurations that auto-start when the plugin is enabled. From the user's perspective, the plugin's MCP tools appear as standard tools -- indistinguishable from built-in capabilities.

This is the distribution mechanism for MCP. Rather than asking every team member to configure MCP servers manually, a plugin packages the server binary, its configuration, and the auto-start logic into something that activates with a single enablement step.

The auto-start behavior means plugin MCP servers consume context from the moment the plugin is active. Disabling an unused plugin recovers that context. If you are running tight on context budget, audit which plugins are active and whether their MCP tools are actually being used.

## MCP Tool Hooks

Hooks interact with MCP tools through a naming convention: `mcp__servername__toolname`. This pattern allows PreToolUse and PostToolUse hooks to match specific MCP tools with the same regex-based matching used for any other tool.

A hook matching `mcp__analytics__.*` intercepts every tool from your analytics MCP server. A hook matching `mcp__.*__write.*` intercepts any write operation across all MCP servers. A hook matching `mcp__memory__.*` catches all operations on a memory server specifically. The pattern matching is flexible enough to express server-level, tool-level, or cross-server policies.

The double-underscore convention (`mcp__server__tool`) is not arbitrary -- it is the exact format Claude Code uses internally to namespace MCP tools. When Claude calls a tool from your analytics server named `get_price_history`, the internal tool name is `mcp__analytics__get_price_history`. Your hook matcher targets this full namespaced name, which means you can write hooks that distinguish between tools of the same name on different servers.

### Logging

The most immediately valuable MCP hook is a logging hook. Every MCP tool invocation -- server, tool, parameters, timestamp -- written to a file or sent to a monitoring service. This creates an audit trail for external service access that is otherwise invisible.

For regulated industries, this is not optional. If your MCP servers connect to financial data sources or customer databases, an auditable log of every query Claude made is a compliance requirement. A PreToolUse hook on `mcp__.*` with a logging command handles this in a few lines of configuration.

### Transformation

PostToolUse hooks can transform MCP tool output before Claude processes it. This is useful when an MCP tool returns more data than Claude needs. A hook that filters a large API response to the relevant fields reduces context consumption and improves Claude's ability to focus on what matters.

The opposite transformation is also valuable: enriching sparse MCP output with additional context. A hook that appends metadata or formatting to a tool's output can improve Claude's interpretation without modifying the MCP server itself.

### Validation

PreToolUse hooks on MCP tools can enforce input validation before the request reaches the external service. A hook that checks whether a database query MCP tool is receiving a SELECT statement (allowed) versus a DROP TABLE statement (denied) prevents catastrophic mistakes without modifying the MCP server's code.

This is defense in depth. The MCP server should have its own input validation. The hook is a second layer that catches what the server might not.

## Domain-Specific Connector Patterns

The most instructive MCP deployments are in domains with rich external data requirements. Financial data analysis provides a useful case study because it involves multiple data sources, real-time feeds, structured and unstructured data, and strict security requirements.

### The Connector Ecosystem

A production analytics platform in financial services might integrate over a dozen data connector categories via MCP, each a separate server wrapping a specific data provider:

- **Fundamental equity data**: company financials, rankings, screening tools
- **Alternative data**: non-traditional datasets aggregated from diverse sources
- **Fund research**: ratings, analysis, fund comparison data
- **Private markets data**: venture, private equity, M&A intelligence
- **Earnings transcripts**: real-time call transcripts and investor event coverage
- **Expert intelligence**: interview transcripts, company research, industry analysis
- **Document governance**: secure data room access with permission controls
- **Live market data**: real-time pricing for fixed income, FX, equities, macro indicators
- **Credit ratings**: entity ratings and research across millions of entities
- **News feeds**: global multi-asset news in real time
- **Regulatory filings**: SEC filings, international regulatory submissions

Each connector is an MCP server that exposes tools for querying its specific data source. The pattern is consistent across all of them:

1. The MCP server wraps an API client for the data source.
2. Tools expose domain-specific queries (get price history, search filings, fetch fundamentals).
3. Authentication is handled at the server level, not passed through Claude.
4. Response schemas are structured for Claude to interpret without additional parsing.

This architecture keeps credentials out of Claude's context (they live in the MCP server's environment), provides structured access patterns instead of raw API calls, and makes every data access auditable through hooks.

### Pre-Built Agent Skills for Domain Workflows

Beyond raw connectors, some domains have developed pre-built agent skills that combine MCP data access with standardized analysis workflows. In financial services, for example, skills exist for comparable company analysis (valuation multiples and operating metrics producing benchmark datasets), discounted cash flow modeling (projections, discount rate calculations, sensitivity tables), due diligence data processing (extracting structured data from document rooms), earnings analysis (pulling metrics, guidance changes, and management commentary from transcripts), and coverage initiation reports (industry analysis with valuation frameworks).

These skills are not MCP servers themselves -- they are packaged workflows that consume data from MCP connectors and produce structured output. The pattern generalizes: any domain with recurring analytical tasks can build skills that standardize the workflow while pulling data from MCP connectors. The MCP server handles data access; the skill handles the analytical process.

### Sovereign Wealth Fund: MCP at Institutional Scale

The most striking MCP deployment in the public record involves a sovereign wealth fund managing over a trillion dollars in assets. The fund built custom MCP integrations connecting Claude to internal portfolio company data systems. Approximately 9,000 portfolio managers query these integrations daily, receiving AI-generated insights on portfolio company performance, metrics, and trends.

The scale is instructive. This is not a developer tool or a prototype. It is infrastructure serving thousands of users in a regulated financial context. The MCP architecture makes it work because credentials and data access policies are server-side, every query is auditable, and the integration points are structured -- portfolio managers interact with Claude, Claude interacts with MCP tools, MCP tools query the fund's internal systems. No credentials traverse the conversation context. No unstructured API calls risk malformed queries against production databases.

### Production Neural Trading Platform: 41 MCP Tools

The ceiling of what is possible with MCP integration is visible in a production trading platform that combines neural forecasting with real-time market analysis. The system integrates 41 MCP tools across multiple domains:

- **Real-time analytics**: market data feeds, price history, volume analysis
- **Neural forecasting**: interface to GPU-accelerated prediction models
- **Backtesting**: Monte Carlo simulation and historical strategy evaluation
- **Risk management**: position sizing, drawdown analysis, correlation monitoring
- **News and sentiment**: real-time news analysis and sentiment scoring

The neural forecasting engine runs state-of-the-art time series models with sub-10ms prediction latency, GPU-accelerated with a 6,250x speedup via CUDA optimization. Claude Code generated the forecasting engine code and the MCP integration layer. The MCP tools provide the structured interface between Claude's reasoning and the GPU-accelerated inference -- Claude decides what to predict and how to act on predictions, while the heavy numerical computation runs server-side where GPU acceleration matters.

At this scale, every configuration discipline described earlier in this chapter becomes mandatory. Server-per-domain grouping isolates failures. Deferred loading prevents context exhaustion. Naming conventions keep 41 tools navigable. Health monitoring catches disconnections before they cascade.

### The Prompt Router Pattern

A recurring architectural pattern in production MCP deployments is the prompt router: a classification layer that sits between user input and MCP tool dispatch. When a user sends a message, the router classifies intent before dispatching to the appropriate processing pipeline.

The pattern looks like this in practice: an inbound request arrives. A router determines whether the user wants to create something, analyze something, query data, or perform an action. Based on the classification, the system invokes specific MCP tools in a structured sequence. A "create strategy" intent triggers a pipeline that creates a portfolio object, loops through strategy creation, breaks down conditions, builds structured objects, and sends the result. An "analyze" intent triggers a different pipeline that fetches data via MCP, runs analysis, and returns findings.

This is not specific to any domain. Any system that exposes many MCP tools to Claude benefits from a router that narrows the tool set before Claude starts reasoning. Instead of Claude choosing from 41 tools on every turn, the router reduces the candidate set to the 5-8 tools relevant to the classified intent. This reduces both token cost (fewer tool descriptions in context) and error rate (fewer irrelevant tools to mistakenly invoke).

### The Three-Layer Architecture

A production deployment pattern that combines MCP with user-facing applications uses three layers:

**Layer 1: AI-powered creation.** Users describe what they want in natural language. Claude processes the intent, invokes MCP tools to access data and computation, and produces structured output. This layer is token-expensive but handles the hard part: translating human intent into structured data.

**Layer 2: Visual display.** The structured output from Layer 1 renders in a standard UI. Users see results, configurations, and data in a readable format. This layer costs nothing in AI tokens -- it is pure frontend rendering.

**Layer 3: No-code refinement.** Users manually adjust the AI-generated output through a visual interface -- toggling parameters, adjusting thresholds, fine-tuning configurations. This layer also costs zero AI tokens. Changes feed back into the same data model that Layer 1 produced.

The economic rationale is clear: AI handles ideation (expensive but high-value), the UI handles display (free), and no-code editing handles refinement (also free). MCP sits in Layer 1, providing the data access and computation that Claude needs to generate structured output. The three-layer split means you pay for AI tokens only during creation, not during the entire user workflow.

### Why Not Just Curl?

A developer could accomplish the same thing by giving Claude bash access and letting it run curl commands against data APIs. This works and is worse in every measurable way.

With curl through bash, credentials appear in command strings (visible in context, logs, and potentially in API calls to the model provider). Access patterns are unstructured (parsing curl output is fragile). Audit logging requires parsing bash command strings. Rate limiting must be handled by the model's judgment rather than server-side logic.

With MCP, credentials are server-side. Access is structured. Logging is automatic with hooks. Rate limiting is enforced by the server. The initial setup cost is higher. The operational security is categorically better.

## Architecture Patterns

### The Go + JavaScript Pattern

A recurring architecture in production MCP deployments uses Go for the heavy computation and JavaScript for the MCP server interface. The rationale is practical and practitioner-motivated: Go handles concurrent data processing, complex algorithms, and performance-sensitive operations. JavaScript is what Claude Code writes most fluently when generating MCP server code.

One practitioner described the reasoning bluntly: Go for heavy lifting because of existing infrastructure expertise, JavaScript for the MCP server because it is the easiest language for Claude to write. The architecture works like this: the Go backend exposes a local API (HTTP or gRPC). The JavaScript MCP server translates between MCP protocol and the Go API. Claude interacts with the MCP server; the MCP server delegates to Go.

The token-saving angle is significant. A codebase analysis tool built with this architecture reported 80-90% token savings compared to having Claude analyze code directly. The Go backend does the heavy parsing and analysis, returning structured results through the MCP server. Claude receives pre-digested information rather than raw source files, consuming a fraction of the context.

This is not the only valid architecture. Python MCP servers work. Rust MCP servers work. But the Go + JavaScript split exploits a specific strength: when you ask Claude Code to build or modify your MCP server, it produces cleaner JavaScript than it does Go. If your MCP server is a thin translation layer, quality of AI-generated code matters more for the interface than for the computation.

### MCP Beyond Engineering: Marketing Campaign Analytics

MCP is not limited to developer tools and data APIs. A growth marketing team built an MCP server integrated with a major social media advertising platform's API to query campaign performance, spending data, and ad effectiveness directly within Claude. Instead of switching between the ad platform's dashboard and Claude for analysis, they query everything in one place.

The MCP server wraps the advertising API, exposes tools for fetching campaign metrics, and returns structured data that Claude can reason about. The team uses it to answer questions like "which campaigns had the highest cost per acquisition this week" or "compare performance of headline variant A versus variant B across all active campaigns." Every query is auditable through hooks, and the API credentials never enter Claude's context.

This is the MCP pattern generalized beyond engineering: any workflow that involves querying an external service, analyzing the response, and making decisions based on the analysis is a candidate for an MCP integration. The effort to build the server is a one-time cost. The ongoing benefit is structured, secure, auditable access to external data from within Claude's reasoning loop.

### Scaling to 41+ Tools

Production deployments exist with over 40 MCP tools across multiple servers. At this scale, configuration discipline becomes non-negotiable.

The patterns that make large integrations work:

- **Server-per-domain grouping**: One MCP server per data domain or service. Analytics tools on one server, portfolio tools on another, backtesting tools on a third. This makes failures isolated -- a crashed analytics server does not take down portfolio management.
- **Deferred loading by default**: With 40+ tools, eager loading would consume a prohibitive fraction of context. Tool search must be enabled.
- **Naming conventions**: Every tool prefixed with its domain: `analytics_price_history`, `portfolio_current_holdings`, `backtest_run_strategy`. Claude can reason about tool selection better when names are self-documenting.
- **Minimal schemas**: Each tool's parameter schema describes exactly what is needed and nothing more. Rich descriptions go in a skill or CLAUDE.md reference, not in the schema that loads into context every session.
- **Health monitoring**: A startup hook or script that verifies all MCP servers are responding before beginning work. Thirty seconds of verification prevents an hour of debugging why tools are missing.

## MCP Over CLI for Data Security

The security argument for MCP over raw CLI access deserves emphasis because it contradicts the instinct to "just use bash."

Bash is universal. Any API with a CLI client or a curl endpoint is accessible through bash. MCP requires building and maintaining a server. The effort difference is real.

But the security properties are not equivalent. When Claude runs a bash command, the full command -- including any embedded credentials, API keys, or tokens -- exists in the conversation context. That context is sent to the model provider's API. Your credentials transit through a third-party service.

When Claude uses an MCP tool, the tool name and parameters exist in context. The credentials exist in the MCP server's environment, which never leaves your machine. The MCP server makes the authenticated request locally. Only the response data enters Claude's context.

For organizations with data classification policies, this distinction matters. It is the difference between "our API keys appear in third-party API calls" and "our API keys stay on our infrastructure." If your security team has opinions about where credentials can exist -- and they should -- MCP is the architecture that satisfies those opinions.

This is not about trust in any particular model provider. It is about architectural hygiene. Credentials belong in server environments, not in conversation contexts, regardless of who is on the other end.

A data infrastructure team put it concisely: use MCP servers rather than the CLI for sensitive data to maintain better security control, especially for data with logging and privacy concerns. The CLI approach works for internal, non-sensitive operations. For anything with compliance requirements -- database queries against production, API calls to financial data providers, access to customer data systems -- MCP is the architecture that satisfies auditors.

### Enterprise Data Privacy Patterns

In regulated environments, MCP is one component of a broader data handling strategy:

**Local filesystem for sensitive data.** Financial data, customer records, and proprietary datasets stay on the local machine. Claude Code accesses them through standard file operations. The data never transits through external APIs beyond what the model provider's API requires for the conversation context.

**MCP for restricted system integrations.** When Claude needs to query a database, access an internal API, or interact with a governed data platform, MCP servers mediate the connection. Credentials stay server-side. Access patterns are structured and auditable. The MCP server can enforce rate limits, query restrictions, and data filtering before results enter Claude's context.

**Context window for non-sensitive research.** Large context windows (100,000+ tokens) are useful for processing non-sensitive documents -- research papers, public filings, open-source documentation. These can be loaded directly without MCP overhead.

The pattern is: classify your data first, then choose the access architecture based on sensitivity. Not everything needs MCP. Not everything should bypass it. The classification determines the architecture, not the other way around.

---

## Key Takeaways

- Every MCP tool definition costs context tokens at session start; use `/mcp` to measure and tool search to defer what you do not need immediately.
- MCP resources (`@server:resource` syntax) put external data on the same footing as local files, enabling direct references in prompts.
- MCP connections fail silently -- run `/mcp` at session start and whenever Claude's behavior changes unexpectedly.
- Background subagents cannot use MCP tools; design workflows accordingly or pre-fetch data in the main agent.
- Hook patterns like `mcp__server__tool` enable logging, validation, and transformation of MCP tool invocations; the double-underscore naming is the exact internal format Claude Code uses.
- MCP servers keep credentials out of Claude's context, making them categorically more secure than equivalent bash commands for sensitive data access.
- Managed settings (`enableAllProjectMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers`) control server approval at the user level; `allowedMcpServers` and `deniedMcpServers` enforce policy at the organizational level.
- Commit `.mcp.json` to version control so one person's configuration effort benefits the entire team.
- The three-layer architecture (AI creation, visual display, no-code refinement) minimizes token costs by restricting AI to the ideation phase, with MCP providing structured data access in that layer.
- At scale (40+ tools), server-per-domain grouping, deferred loading, and strict naming conventions prevent context bloat and tool collisions.
- For regulated environments, classify data first then choose the access architecture: local filesystem for sensitive data, MCP for restricted system integrations, direct context for non-sensitive research.
