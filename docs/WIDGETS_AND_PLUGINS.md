# Widgets and Plugins

Status: Draft RFC

This document proposes a first-class widget and plugin system for Paseo.

## Summary

Paseo does not currently have a general runtime plugin or widget platform. It already has several extension points:

- agent providers
- MCP server integration via `mcpServers`
- daemon-side MCP for agent management
- provider modes and feature toggles surfaced in the UI

Those are useful building blocks, but they do not yet form a user-installable, cross-platform plugin system.

This RFC proposes:

- a **daemon-side plugin runtime** for privileged local logic
- a **JSON-render-based widget protocol** for safe, cross-platform UI
- **Paseo-owned component and action catalogs** for consistency and security
- optional **MCP Apps** export for interoperability with MCP-capable hosts

## Decision

Paseo widgets should use a **JSON-render-style declarative UI protocol** as the canonical internal format.

More specifically:

- **Use JSON-render internally** for widgets rendered in Paseo mobile, web, and desktop clients.
- **Use daemon-side TypeScript plugins** for execution, data access, local integrations, and actions.
- **Use MCP** for agent-facing tools and integrations.
- **Use MCP Apps optionally** as an external interoperability layer, not as the primary internal widget runtime.

## Why this approach

Paseo needs all of the following at once:

- native-feeling mobile UI
- desktop and web support
- local-first security
- no arbitrary third-party client code execution
- backward-compatible protocol evolution
- a way for plugins to expose interactive UI

A JSON-render-style protocol fits those constraints better than:

- arbitrary webviews/iframes as the primary widget system
- arbitrary third-party React bundles
- a very limited hand-written widget schema that becomes too rigid over time

## Prior art considered

### JSON-render

Vercel Labs' `json-render` is a strong fit for the widget layer because it provides:

- typed catalogs via Zod
- declarative UI trees
- actions
- state bindings
- conditional visibility
- streaming/spec updates
- React and React Native renderers
- MCP Apps integration via `@json-render/mcp`

This makes it a strong foundation for Paseo's internal widget protocol.

### MCP Apps

MCP Apps are a useful direction for interactive UI inside MCP-capable chat clients.

We should treat MCP Apps as:

- useful for interoperability
- useful for exporting selected widgets outside Paseo
- not the canonical internal UI format for Paseo clients

Reason: Paseo's primary surfaces are its own app clients, especially mobile, not external chat hosts.

### `mcp-ui`

`mcp-ui` is promising and worth tracking, but it is better treated as an experimental interoperability ecosystem than the primary foundation for Paseo's internal widget runtime.

Reason:

- it is still described as experimental/community-driven
- it is centered on HTML/iframe/remote-dom style rendering
- that is a weaker fit for native mobile rendering than JSON-render

## Goals

- Allow third-party and first-party plugins to add useful dashboard widgets.
- Keep all privileged logic on the daemon.
- Make widgets render safely across mobile, web, and desktop.
- Preserve Paseo's backward-compatibility guarantees.
- Allow plugins to expose actions without client-side code execution.
- Allow some widgets and plugin capabilities to be exported to MCP-capable clients.

## Non-goals

- Running arbitrary third-party JavaScript or React bundles inside Paseo clients.
- Letting plugins define arbitrary navigation routes in clients in v1.
- Letting plugins fully control client layout or theming.
- Replacing MCP for agent tool integration.
- Designing a fully generic cross-host UI standard for the entire MCP ecosystem.

## Architecture

## 1. Daemon-side plugin runtime

Plugins run locally on the Paseo daemon machine.

They may:

- inspect projects, worktrees, agents, and daemon state
- call external APIs
- maintain plugin state
- expose widget data
- handle widget actions
- optionally register MCP tools/resources for agents

They do **not** run inside the mobile/web/desktop client.

### Why daemon-side

This matches Paseo's local-first model:

- your code and local state stay on your machine
- privileged integrations stay on your machine
- client trust remains simple
- permission boundaries are easier to reason about

## 2. Widget protocol

Widgets are described using a **JSON-render-based declarative spec**.

A plugin returns:

- widget metadata
- a UI spec
- optional state
- action identifiers

Clients render the spec using a Paseo-owned registry.

## 3. Paseo-owned catalog

Plugins do not ship arbitrary component implementations.

Instead, Paseo defines the allowed component and action catalog. Plugins can only emit specs that use those approved components and actions.

This ensures:

- security
- visual consistency
- mobile compatibility
- predictable upgrades
- easier testing

## 4. Action execution model

Widget actions are invoked by the client but executed by the daemon/plugin.

Flow:

1. user taps a button or changes a field
2. client emits a widget action request to the daemon
3. daemon routes the request to the owning plugin
4. plugin performs work
5. plugin returns updated widget state/spec and optional messages
6. daemon pushes widget updates to subscribed clients

This avoids arbitrary client-side code execution while keeping widgets interactive.

## 5. Optional MCP interoperability

A plugin may optionally also expose:

- MCP tools for agents
- MCP App UIs for external MCP hosts

This is additive.

The canonical internal widget representation remains Paseo JSON-render specs.

## Canonical model

### Plugin manifest

```ts
export interface PaseoPluginManifest {
  id: string;
  name: string;
  version: string;
  capabilities: Array<"widgets" | "actions" | "mcp" | "settings">;
}
```

### Plugin interface

```ts
export interface PaseoPlugin {
  manifest: PaseoPluginManifest;

  activate(ctx: PluginContext): Promise<void>;

  getWidgets?(input: WidgetQuery): Promise<PaseoWidget[]>;
  invokeAction?(input: InvokeWidgetActionInput): Promise<WidgetActionResult>;

  registerMcp?(ctx: PluginMcpContext): Promise<void>;
}
```

### Widget locations

```ts
type WidgetLocation =
  | "home"
  | "project"
  | "agent"
  | "settings";
```

### Widget query

```ts
interface WidgetQuery {
  location: WidgetLocation;
  projectId?: string;
  agentId?: string;
}
```

### Widget shape

```ts
interface PaseoWidget {
  id: string;
  pluginId: string;
  title: string;
  location: WidgetLocation;
  order?: number;
  refreshPolicy?: {
    kind: "manual" | "interval" | "subscription";
    intervalMs?: number;
  };
  spec: JsonRenderSpec;
  state?: Record<string, unknown>;
}
```

### Widget action request

```ts
interface InvokeWidgetActionInput {
  pluginId: string;
  widgetId: string;
  actionId: string;
  params?: unknown;
  context?: {
    projectId?: string;
    agentId?: string;
  };
}
```

### Widget action result

```ts
interface WidgetActionResult {
  ok: boolean;
  message?: string;
  patch?: {
    spec?: JsonRenderSpec;
    state?: Record<string, unknown>;
  };
}
```

## Proposed built-in widget catalog

The exact catalog should remain small in v1.

Suggested initial components:

- `Card`
- `Stack`
- `Row`
- `Text`
- `Badge`
- `StatusDot`
- `Button`
- `Divider`
- `Stat`
- `List`
- `Table`
- `Log`
- `Tabs`
- `Toggle`
- `Select`
- `Input`
- `AgentLink`
- `ProjectLink`

Suggested built-in action kinds:

- `plugin.invoke`
- `open.agent`
- `open.project`
- `open.url`
- `refresh.widget`

Important constraint:

- Paseo owns the implementation of these components and actions.
- Plugins may emit specs using them.
- Plugins do not register executable component code in clients in v1.

## State and bindings

Widgets may include state.

That state is intended for:

- simple view state
- input values
- loading status
- plugin-calculated values

The daemon remains the source of truth for durable or privileged state.

Client-local ephemeral state is acceptable only where it does not affect correctness or security.

## API and transport changes

Paseo should add widget support in a backward-compatible way.

### Principle

All new fields and messages must be optional or ignorable by older clients/daemons.

### New message types

Suggested protocol additions:

- `list_widgets`
- `widgets_snapshot`
- `widget_updated`
- `invoke_widget_action`
- `invoke_widget_action_result`

### Example shapes

```ts
interface WidgetsSnapshotMessage {
  type: "widgets_snapshot";
  location: WidgetLocation;
  widgets: PaseoWidget[];
}
```

```ts
interface WidgetUpdatedMessage {
  type: "widget_updated";
  widget: PaseoWidget;
}
```

```ts
interface InvokeWidgetActionMessage {
  type: "invoke_widget_action";
  pluginId: string;
  widgetId: string;
  actionId: string;
  params?: unknown;
}
```

Older clients should safely ignore these messages.

## Security model

This system must preserve Paseo's local-first security posture.

### Allowed

- daemon-side plugin code
- declarative widget specs
- daemon-mediated actions
- plugin-owned local/network integrations with explicit permissions

### Not allowed by default

- arbitrary third-party JS execution inside Paseo clients
- arbitrary third-party React bundles in clients
- arbitrary HTML/iframe widgets as the primary rendering model
- direct client access to privileged plugin capabilities

### Plugin capability permissions

Plugins should declare required privileges, for example:

- `readProjects`
- `readAgents`
- `createAgents`
- `sendAgentMessage`
- `networkAccess`
- `filesystemRead`
- `filesystemWrite`
- `registerMcp`

These permissions can later be surfaced in plugin install/enable flows.

## Rendering strategy

### Internal Paseo clients

Primary rendering path:

- mobile: JSON-render React Native renderer
- web/desktop: JSON-render React renderer or shared RN/web abstraction

### External MCP clients

Optional export path:

- selected widgets may be exposed as MCP Apps
- use `@json-render/mcp` where appropriate

This should be treated as an adapter/export layer, not the canonical internal format.

## Relationship to MCP

MCP remains the correct tool for agent-facing integrations.

### Use MCP for

- agent tools
- resources/prompts
- external integration points for coding agents
- daemon-side control surfaces for agent workflows

### Do not use MCP UI as the primary internal widget runtime

Reason:

- Paseo's primary UI surfaces are its own clients
- Paseo needs native mobile rendering
- MCP UI standards and libraries are still evolving

### Use MCP Apps as an adapter

When useful, a widget or plugin may also expose an MCP App for:

- Claude
- ChatGPT
- Cursor
- VS Code
- other MCP-capable clients

## UX placement

Suggested initial widget locations:

### Home

- running agents
- pending approvals
- daemon health
- recent project activity

### Project

- worktrees
- CI state
- PR status
- branch info
- plugin-specific project integrations

### Agent

- plugin-provided controls or diagnostics related to the current agent
- task context
- integration state

### Settings

- plugin management
- plugin configuration
- integration health

## Versioning and compatibility

Paseo's compatibility rules are strict and should remain so.

### Compatibility requirements

- New widget protocol fields must be optional.
- Existing fields must not become required.
- Widget specs should support graceful degradation when clients are older than the daemon.
- Unknown component types or action types should fail gracefully.

### Client behavior for unknown content

Clients should:

- ignore unknown message types when safe
- render a fallback placeholder for unknown widget components
- show a non-fatal unsupported-version message where appropriate

## Example plugin: GitHub / CI

### Responsibilities

- read repo metadata
- fetch PR and CI status
- expose actions like `rerun-checks` or `open-pr`
- optionally provide MCP tools for agents

### Example widget use cases

- show current PR title and status
- show failing checks
- button: create an agent to investigate failures
- button: open PR in browser
- button: refresh CI state

## Example widget flow

1. User opens a project screen.
2. Client requests widgets for `location = "project"`.
3. Daemon queries enabled plugins.
4. GitHub plugin returns a widget spec and state.
5. Client renders the widget using Paseo's built-in registry.
6. User taps `Rerun Checks`.
7. Client sends `invoke_widget_action`.
8. Plugin performs the rerun via API.
9. Plugin returns an updated state/spec.
10. Client updates the widget.

## Implementation phases

## Phase 1

- daemon plugin loader
- plugin manifest and capability declaration
- widget query API
- widget action API
- small built-in widget catalog
- home/project/agent/settings widget locations
- basic settings UI for enable/disable

## Phase 1.1

- subscriptions for live widget updates
- plugin configuration forms
- widget pinning and ordering
- better loading/error states

## Phase 2

- richer widget components
- more advanced tables/logs/timelines/charts
- plugin action auditing
- MCP App export for selected widgets
- plugin marketplace or install flow if desired

## Open questions

- Should widget state be fully daemon-owned, or partially client-owned for ephemeral controls?
- How much of JSON-render should be exposed in v1 versus a smaller constrained subset?
- Should plugin installation be npm-package-based, local-folder-based, or both?
- How should plugin permissions be approved and reviewed by the user?
- Should widget actions be serialized through a generic command bus or call plugin handlers directly?
- Which widgets, if any, should be exposable as MCP Apps by default?

## Recommended next steps

1. Define the initial Paseo widget catalog.
2. Add protocol types for widget snapshot and action invocation.
3. Build a daemon-side plugin manager.
4. Implement one internal reference plugin, ideally GitHub/CI or worktrees.
5. Render widgets in one location first, likely the project screen.
6. Add MCP Apps export later as an interoperability layer.

## Final recommendation

Paseo should adopt the following architecture:

- **daemon-side plugins** for privileged execution and integrations
- **JSON-render-based widgets** as the canonical internal UI protocol
- **Paseo-owned component/action catalogs** for security and consistency
- **MCP** for agent-facing tools and integrations
- **MCP Apps** as an optional export/interoperability layer

This gives Paseo a plugin system that is practical, safe, native-friendly, and aligned with the existing architecture.
