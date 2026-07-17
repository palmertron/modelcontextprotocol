# SEP-XXXX: Toolset Versioning

- **Status**: Draft
- **Type**: Extensions Track
- **Created**: 2026-07-14
- **Author(s)**: Matt Palmer (@palmertron)
- **Sponsor**: None (seeking sponsor)
- **Extension Identifier**: `io.modelcontextprotocol/toolsets`
- **PR**: [To be filled after PR creation]
- **Related**: [SEP-2133](./2133-extensions.md) (Extensions), [SEP-2549](./2549-TTL-for-list-results.md) (TTL for list results), [SEP-2567](./2567-sessionless-mcp.md) (Sessionless MCP); prior proposals [SEP-1575](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) (Tool Semantic Versioning, dormant), [SEP-1300](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1300) / [SEP-2084](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2084) (tool groups / primitive grouping, rejected)

## Abstract

This SEP proposes an optional MCP extension that introduces **Toolsets**: named, semantically versioned, immutable capability surfaces. A Toolset is a fixed membership of tool names. Clients discover Toolsets via `toolsets/list` and pin a specific `(name, version)` on `tools/list` and `tools/call` requests.

When a Toolset is pinned, the server returns only member tools from `tools/list` and rejects `tools/call` for tools outside that membership. This gives hosts a predictable contract for dynamic discovery without requiring per-tool semantic versioning or session-scoped state.

The design is backward-compatible: clients that omit Toolset parameters continue to see the full flat tool list. The extension follows [SEP-2133](./2133-extensions.md) capability negotiation and is intended to incubate outside the core protocol.

## Motivation

Many MCP hosts discover tools at runtime via `tools/list` and then let the model choose which tools to invoke. That flexibility is valuable, but it creates a production failure mode: when a server adds, removes, or substantially changes tools, agent behavior can shift across many clients with no explicit opt-in from client operators. Call this **uncontrolled tool-surface expansion** (sometimes informally described as unexpected tools appearing in the agent's context and altering selection).

[SEP-1575](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) attempted to address related instability with per-tool Semantic Versioning and client `tool_requirements` constraints. That proposal is **dormant** and never landed in the schema. Reviewer feedback favored versioning at a higher grain than individual tools (server- or bundle-level contracts), noted that multi-version tool registries complicate `tools/list`, and observed that much of the client ecosystem was not ready for constraint-resolution semantics.

Separately, proposals for tool groups and tags ([SEP-1300](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1300), [SEP-2084](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2084)) were **rejected**. Those proposals primarily addressed discovery taxonomy and filter DSLs, not versioned pin contracts for production stability.

What remains missing is a **standardized, pinnable capability surface**: a discoverable, versioned bundle of tools that a client can request consistently. Without it:

1. Hosts that rely on dynamic discovery cannot freeze the tool surface their agents reason over.
2. Operators lack a simple, auditable declaration such as "this agent uses `core-ops@1.2.0`."
3. Server authors invent proprietary "toolset" enablement (common in product MCP servers) that does not interoperate across clients.

This SEP addresses that gap as an optional extension, without reopening per-tool SemVer constraint resolution.

## Specification

### Extension Identifier

This extension is identified as: `io.modelcontextprotocol/toolsets`.

### Capability Negotiation

Clients and servers advertise support via the `extensions` map defined in [SEP-2133](./2133-extensions.md).

```json
{
  "extensions": {
    "io.modelcontextprotocol/toolsets": {}
  }
}
```

No extension-specific settings are required for v1; an empty object indicates support.

A client that wishes to pin Toolsets **MUST** advertise this extension in client capabilities at initialize. A server that exposes `toolsets/list` or honors Toolset parameters on `tools/list` / `tools/call` **MUST** advertise this extension in server capabilities. Advertising declares extension support; the pin itself is a separate per-request `toolset` parameter (see section on Toolset Selection below).

Servers **MUST NOT** require this extension for basic tool use: omitting Toolset parameters **MUST** preserve today's full flat `tools/list` and unrestricted `tools/call` behavior (subject to ordinary authz).

### Toolset Object

```typescript
/**
 * Status of a published Toolset version.
 */
type ToolsetStatus = "stable" | "deprecated" | "experimental";

/**
 * A named, versioned, immutable capability surface.
 */
interface Toolset {
  /**
   * Toolset identifier within the server. Unique together with `version`.
   * SHOULD use reverse-DNS or a stable short name (e.g. "core-ops").
   */
  name: string;

  /**
   * Semantic Version 2.0.0 of this Toolset publication.
   * Once published, the pair (name, version) is immutable.
   */
  version: string;

  /**
   * Optional human-readable display name.
   */
  title?: string;

  /**
   * Optional description of the capability surface.
   */
  description?: string;

  /**
   * Lifecycle status of this Toolset version.
   */
  status: ToolsetStatus;

  /**
   * Fixed membership: tool names included in this surface.
   * Order MAY be significant for display; servers SHOULD keep it stable.
   */
  tools: string[];

  /**
   * Optional ISO-8601 date after which clients SHOULD stop selecting
   * this Toolset version.
   */
  deprecationDate?: string;
}
```

#### Immutability

For a given `(name, version)`:

1. The `tools` membership **MUST NOT** change after publication.
2. Servers **SHOULD NOT** change a member tool's wire contract for the lifetime of that Toolset version. A wire-contract change includes renaming the tool, changing `inputSchema` shape or requiredness in a way that invalidates existing valid arguments, or changing documented success semantics of the tool's result.
3. Changing membership or a member tool's wire contract while retaining the same `(name, version)` **violates this extension's SemVer intent**. Such changes **MUST** be published as a **new** Toolset version: **MINOR** when only adding tools to the surface; **MAJOR** when removing tools or breaking contracts of existing members. Servers **MAY** continue to serve prior versions concurrently.

This extension does **not** require servers to host multiple implementations of the same tool name. Schema and semantics permanence is a **publication discipline** tied to Toolset versions, not a per-tool version registry. Hosts that need a cryptographic freeze of Toolset membership and member tool descriptors should consider a future content `digest` (see Open Questions).

#### SemVer Rules for Toolset Versions

These rules apply to the Toolset package, not to individual tools:

| Change                                                                                                      | Version impact                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Remove a tool from membership, or intentionally break a member tool contract while retiring the old surface | **MAJOR**                                                                                                                                           |
| Add tools in a new Toolset version (prior versions unchanged)                                               | **MINOR**                                                                                                                                           |
| Metadata-only changes (`title`, `description`, `status`, `deprecationDate`) on a **new** publication        | **PATCH** (preferred) or republish metadata carefully; published `(name, version)` fields other than advisory metadata **MUST NOT** mutate in place |

#### Operational Guidance

Servers that publish Toolsets take on a small operational contract beyond today's flat tool list:

1. Servers **MAY** serve multiple versions of the same Toolset name concurrently. Clients select with an exact pin; the server does not resolve ranges in v1.
2. Before removing a version from publication, servers **SHOULD** mark it `deprecated` and **MAY** set `deprecationDate` so hosts can migrate pins.
3. Servers **MAY** later omit a `(name, version)` from `toolsets/list`. Clients still pinned to that pair then receive `unknown_toolset` on pinned `tools/list` / `tools/call`.
4. This SEP does **not** require unbounded retention of every historical Toolset version. Retention and retirement are operator policy. Servers **SHOULD** document which versions they commit to keep available for consumers.
5. Storage and replication of published membership records are implementation details; the normative requirement is only that a published `(name, version)` keep a fixed `tools` membership as long as it appears in the response from `toolsets/list`.

### Methods

#### `toolsets/list`

Lists Toolset versions published by the server.

**Request params** (all optional):

```typescript
interface ListToolsetsRequestParams {
  /**
   * If set, only Toolsets with this name.
   */
  name?: string;

  /**
   * If set, only Toolsets with this status.
   */
  status?: ToolsetStatus;

  /**
   * Cursor for pagination, consistent with other list methods.
   */
  cursor?: string;
}
```

**Result**:

```typescript
interface ListToolsetsResult {
  toolsets: Toolset[];
  nextCursor?: string;
}
```

Servers supporting this extension **MUST** implement `toolsets/list`.

Result order is **unspecified** unless the server documents one. Servers **SHOULD** keep order stable across pages for a given filter set so pagination is deterministic. Servers **MAY** list newest SemVer first for a given `name`.

#### Toolset Selection on `tools/list` and `tools/call`

Selection is **per-request** (not session-scoped), consistent with [SEP-2567](./2567-sessionless-mcp.md). Clients pin by passing the same Toolset reference on discovery and invocation.

```typescript
/**
 * Reference to a published Toolset version.
 */
interface ToolsetRef {
  name: string;
  /**
   * Exact Toolset version (SemVer). Ranges are not supported in v1.
   */
  version: string;
}
```

`tools/list`: when this extension is negotiated, request params **MAY** include:

```json
{
  "toolset": {
    "name": "core-ops",
    "version": "1.2.0"
  }
}
```

If `toolset` is present:

- The server **MUST** verify that `(name, version)` exists.
- The server **MUST** return only tools whose names are in that Toolset's `tools` membership.
- If a membership name has no corresponding registered tool, the server **SHOULD** omit it from the pinned `tools/list` result rather than failing the whole list.
- Membership filtering applies **before** pagination: any `cursor` for `tools/list` pages the filtered membership, consistent with other list methods.
- If the Toolset is unknown, the server **MUST** return an error (see Error Handling).

If `toolset` is absent, behavior is unchanged from core MCP.

`tools/call`: when this extension is negotiated, request params **MAY** include the same `toolset` field.

If `toolset` is present:

- The server **MUST** verify the Toolset exists.
- If `params.name` is not a member of that Toolset, the server **MUST NOT** execute the tool and **MUST** return an error.
- If the Toolset is unknown, the server **MUST** return an error (see Error Handling).

If `toolset` is absent, membership checks from this extension do not apply.

Hosts that pin a Toolset **SHOULD** pass the same `toolset` on both `tools/list` and `tools/call` so discovery and invocation stay aligned.

### Caching

Pinned `tools/list` responses **SHOULD** be cacheable under [SEP-2549](./2549-TTL-for-list-results.md). Clients **MUST** include the Toolset `(name, version)` in the cache key when `toolset` was supplied. Toolset pins **MUST NOT** reuse an unpinned cache entry; distinct pins MUST use distinct cache keys (implementations MAY dedupe identical payloads by value).

### Error Handling

This extension defines the following error conditions. `data` **MUST** identify the extension and a machine-readable `reason`. Exact numeric JSON-RPC `code` values **SHOULD** follow SDK/project conventions for extension errors until cross-SDK coordination exists (see Open Questions). Clients **SHOULD** key behavior on `data.reason` (and `data.extension`), not solely on `code`.

| Condition                         | Meaning                                                                     | Suggested `data.reason` |
| --------------------------------- | --------------------------------------------------------------------------- | ----------------------- |
| Unknown Toolset `(name, version)` | Pin does not match any published Toolset (for `tools/list` or `tools/call`) | `unknown_toolset`       |
| Tool not in Toolset               | `tools/call` name outside pinned membership                                 | `tool_not_in_toolset`   |

Example:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32007,
    "message": "Tool not in Toolset",
    "data": {
      "extension": "io.modelcontextprotocol/toolsets",
      "reason": "tool_not_in_toolset",
      "toolset": { "name": "core-ops", "version": "1.2.0" },
      "tool": "experimental_admin_wipe"
    }
  }
}
```

### Non-Goals (v1)

The following are explicitly out of scope for this SEP:

1. **Per-tool Semantic Versioning** and `tool_requirements` constraint maps (see dormant [SEP-1575](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575)).
2. **Version ranges** on Toolset refs (e.g. `^1.2.0`). Exact pins only in v1.
3. **Groups/tags filter DSLs** for general discovery taxonomy (see rejected [SEP-1300](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1300) / [SEP-2084](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2084)).
4. Versioning of resources or prompts.
5. Replacing `Implementation.version` or registry server-package versioning.

### Examples

#### Publish multiple Toolset versions

```json
{
  "toolsets": [
    {
      "name": "core-ops",
      "version": "1.2.0",
      "title": "Core operations",
      "description": "Stable CRM operations for production agents",
      "status": "stable",
      "tools": ["search_contacts", "create_deal", "update_deal_stage"]
    },
    {
      "name": "core-ops",
      "version": "1.3.0",
      "title": "Core operations",
      "description": "Adds analyze_report; prior 1.2.0 unchanged",
      "status": "stable",
      "tools": [
        "search_contacts",
        "create_deal",
        "update_deal_stage",
        "analyze_report"
      ]
    }
  ]
}
```

A client pinned to `core-ops@1.2.0` never sees `analyze_report`, even after `1.3.0` is published.

#### Pin on list and call

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {
    "toolset": { "name": "core-ops", "version": "1.2.0" }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "search_contacts",
    "arguments": { "query": "acme" },
    "toolset": { "name": "core-ops", "version": "1.2.0" }
  }
}
```

## Rationale

### Why an extension, not a core Standards Track change?

Toolsets are optional production-governance machinery. [SEP-2133](./2133-extensions.md) exists to incubate such capabilities without forcing every implementation to adopt them. SDKs can ship the extension disabled-by-default and evolve it independently of core protocol releases. Graduation to core remains possible after demonstrated adoption.

### Why membership of tool names, not SemVer ranges per tool?

Per-tool SemVer with caret/tilde resolution (SEP-1575) did not land and faced substantial review pushback: coupled tools, multi-version list ambiguity, and weak client-side metadata support. Toolsets version the **surface** (the set an agent is allowed to see and call), which matches the reviewers' preference for higher-level contracts and directly addresses uncontrolled expansion without a package-manager dependency solver.

Stability of member tool wire contracts is a publication discipline on the Toolset version (**SHOULD NOT** break in place; breaking changes **MUST** mint a new **MAJOR** Toolset version), not a constraint engine hosting multiple SemVer'd implementations of the same tool name.

### Why per-request pins instead of session-active Toolsets?

[SEP-2567](./2567-sessionless-mcp.md) removes protocol sessions and requires list endpoints not to vary by implicit session state. Carrying `toolset` on each request:

- keeps `tools/list` cacheable with an explicit cache key ([SEP-2549](./2549-TTL-for-list-results.md));
- works for hosts that create a connection per call;
- makes the pin auditable in logs without reconstructing session state.

Hosts remain free to configure a preferred Toolset once in application config and attach it automatically.

### Why exact versions only in v1?

Ranges invite resolution rules (latest matching, pre-release policy, conflict behavior) that complicated SEP-1575. Exact pins are trivial to implement, easy to audit, and sufficient for production freezes. Ranges can be considered later if demand is clear.

### Differentiation from rejected grouping SEPs

SEP-1300 / SEP-2084 proposed general-purpose groups/tags and filter syntax for organizing primitives. This SEP does not introduce a taxonomy DSL. It introduces a **versioned pin contract** whose primary purpose is stability and governance of the tool surface exposed to agents.

### Prior Art

Product MCP servers already expose informal "toolsets" as enablement bundles. Standardizing discovery (`toolsets/list`) and pin semantics makes those patterns interoperable across clients and servers.

## Backward Compatibility

Fully backward-compatible:

- Clients and servers that ignore this extension behave exactly as today.
- Presence of Toolset fields on requests is optional and gated by extension negotiation.
- No changes to the core meaning of unversioned `tools/list` / `tools/call`.
- Existing tools require no schema changes to be included in a Toolset.

## Security Implications

1. **Reduced unintended capability exposure:** Pinning a Toolset limits which tools enter model context and which calls succeed, reducing risk from newly published powerful tools appearing unexpectedly in `tools/list`.
2. **Server-side enforcement required:** Clients **MUST NOT** be the sole enforcer. Servers **MUST** validate `toolset` membership on `tools/call`.
3. **Authz orthogonality:** Toolsets are not a replacement for authentication or authorization. A Toolset pin narrows the advertised/callable surface; ordinary authz **MUST** still apply.
4. **Deprecation:** Deprecated Toolsets remain callable until removed; servers **SHOULD** communicate `status` / `deprecationDate` so hosts can plan migrations.
5. **No new trust boundary:** Toolsets do not introduce a new identity or auth mechanism.

## Reference Implementation

A reference implementation is required before this SEP can advance to Final, per SEP guidelines and [SEP-2484](./2484-conformance-tests-required-for-final-seps.md) expectations for protocol changes. That reference implementation is provided at the links below.

- **Python SDK** (`Toolsets` extension): [palmertron/python-sdk@feature/toolset-versioning](https://github.com/palmertron/python-sdk/tree/feature/toolset-versioning) — advertises `io.modelcontextprotocol/toolsets`, serves `toolsets/list`, filters pinned `tools/list` / `tools/call`, with coverage in `tests/server/test_toolsets.py`.
- **E2E demo** (Streamable HTTP server + pinning clients): [palmertron/mcp-toolset-example](https://github.com/palmertron/mcp-toolset-example) — publishes concurrent `core-ops` versions (`1.0.0` / `1.1.0` / `2.0.0`); `client/verify.py` asserts pin membership and `tool_not_in_toolset` without an LLM; a CLI agent pins `core-ops@1.1.0` for interactive demos.

### SDK Impact

Official SDKs typically ship both MCP server and MCP client libraries. Expected impact for this extension:

- **Server libraries:** opt-in enablement (disabled by default per [SEP-2133](./2133-extensions.md)); include the extension in server capabilities when enabled; implement `toolsets/list`; enforce optional `toolset` on `tools/list` / `tools/call`.
- **Client libraries:** include the extension in client capabilities at initialize when the host will pin; pass `toolset` on `tools/list` / `tools/call`; when caching pinned `tools/list` results, include `(name, version)` in the cache key (see Caching).

## Performance Implications

- `toolsets/list` is a small additional list endpoint; servers with few Toolset versions should remain negligible in cost.
- Pinning can **reduce** `tools/list` payload sizes and token usage for agents by excluding non-member tools from model context.
- Per-request `toolset` adds small parameter overhead and makes membership checks O(membership) per call (trivial for typical sizes).

## Testing Plan

Interoperable implementations **SHOULD** cover:

1. Advertise/negotiate `io.modelcontextprotocol/toolsets`.
2. `toolsets/list` returns published Toolsets; filters by `name` / `status`.
3. `tools/list` without `toolset` returns the full tool list.
4. `tools/list` with a valid pin returns exactly the target toolset version membership.
5. `tools/list` / `tools/call` with unknown `(name, version)` errors.
6. `tools/call` for a non-member under a pin errors; member succeeds.
7. Immutability: republishing the same `(name, version)` with different membership is rejected or treated as a server bug in conformance tests.
8. Concurrent Toolset versions: pinning `1.2.0` does not observe tools only added in `1.3.0`.
9. Pinned `tools/list` omits membership names that have no registered tool (rather than failing the list).

## Alternatives Considered

1. **Revive SEP-1575 and compose Toolsets as SemVer BOMs.** Rejected for v1: depends on dormant machinery; reopens constraint-resolution complexity; does not match reviewer guidance favoring higher-level contracts.
2. **Session-scoped "active Toolset".** Rejected: conflicts with sessionless MCP and list cacheability.
3. **Server-only** `Implementation.version` **bumps.** Helpful but too coarse when one server publishes multiple capability products or must keep an old surface alive while shipping a new one.
4. **Client-side allowlists only.** Useful locally, but not discoverable, not enforceable server-side, and not interoperable.
5. **Informational-only Toolset metadata without call enforcement.** Insufficient against hosts that still call whatever the model selects from a broader ambient list.

## Open Questions

1. Should `tools/list_changed` (or subscription filters) be Toolset-scoped when a pin is active, or always describe the full server catalog?
2. Should v1 allow an optional content `digest` on `Toolset` covering membership and member tool descriptors (for supply-chain pinning of a Toolset snapshot)?
3. Prefer extending `tools/list` params vs introducing `toolsets/select` (stateless handle returned)? Current draft prefers param-on-list/call for simplicity and sessionlessness.
4. How should extension-specific JSON-RPC error `code` integers be coordinated across official SDKs, given that `data.reason` is already the stable cross-implementation signal?

## Acknowledgments

This proposal benefits from the discussion on [SEP-1575](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) (especially arguments for server-/bundle-level versioning over per-tool SemVer) and from product patterns that already group MCP tools into named enablement sets.