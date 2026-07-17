# SEP-XXXX: Toolset Versioning

- **Status**: Draft
- **Type**: Extensions Track
- **Created**: 2026-07-14
- **Author(s)**: Matt Palmer (@palmertron)
- **Sponsor**: None (seeking sponsor)
- **Extension Identifier**: `io.modelcontextprotocol/toolsets`
- **PR**: [To be filled after PR creation]
- **Related**: [SEP-2133](./2133-extensions.md) (Extensions), [SEP-2575](./2575-stateless-mcp.md) (Stateless MCP), [SEP-2549](./2549-TTL-for-list-results.md) (TTL for list results), [SEP-2567](./2567-sessionless-mcp.md) (Sessionless MCP); prior proposals [SEP-1575](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) (Tool Semantic Versioning, dormant), [SEP-1300](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1300) / [SEP-2084](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2084) (tool groups / primitive grouping, rejected)

## Abstract

This SEP proposes an optional MCP extension that introduces **Toolsets**: named, semantically versioned, immutable capability surfaces. A Toolset is a fixed membership of tool names. Clients discover Toolsets via `toolsets/list` and pin a specific `(name, version)` on `tools/list` and `tools/call` requests.

When a Toolset is pinned, the server returns only member tools from `tools/list` and rejects `tools/call` for tools outside that membership. This gives hosts a predictable contract for dynamic discovery without requiring per-tool semantic versioning or session-scoped state.

The design is backward-compatible within MCP protocol revision `2026-07-28` and later: clients that omit Toolset parameters continue to see the full flat tool list. The extension uses the extension framework introduced by [SEP-2133](./2133-extensions.md) with the stateless capability advertisement model introduced by [SEP-2575](./2575-stateless-mcp.md), and is intended to incubate outside the core protocol.

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

This extension targets MCP protocol revision `2026-07-28` and later. It does not define support for earlier protocol revisions.

[SEP-2133](./2133-extensions.md) introduced the `extensions` capability map and originally described exchanging it during `initialize`. [SEP-2575](./2575-stateless-mcp.md) removes `initialize` for `2026-07-28` and later, replacing that handshake with per-request client capabilities and server capability discovery through `server/discover`. This SEP is specified in preparation for that protocol revision and intentionally supports only the SEP-2575 model, not SEP-2133's initialization-based negotiation model.

Clients advertise support on each extension-dependent request:

```json
{
  "_meta": {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientInfo": {
      "name": "example-client",
      "version": "1.0.0"
    },
    "io.modelcontextprotocol/clientCapabilities": {
      "extensions": {
        "io.modelcontextprotocol/toolsets": {}
      }
    }
  }
}
```

Servers advertise support in the capabilities returned by `server/discover`:

```json
{
  "capabilities": {
    "extensions": {
      "io.modelcontextprotocol/toolsets": {}
    }
  }
}
```

No extension-specific settings are required for v1; an empty object indicates support.

A client **MUST** include this extension in `params._meta["io.modelcontextprotocol/clientCapabilities"].extensions` on every `toolsets/list` request and every `tools/list` or `tools/call` request carrying a Toolset pin. A server that exposes `toolsets/list` or honors Toolset parameters on `tools/list` / `tools/call` **MUST** advertise this extension in the server capabilities returned by `server/discover`.

Before invoking `toolsets/list` or sending a Toolset pin, a client **MUST** confirm that the server advertised this extension. A server supporting this extension that receives an extension-dependent request without the corresponding per-request client capability **MUST** return `MissingRequiredClientCapabilityError` (`-32021`). Advertising declares extension support; the pin itself is a separate per-request `toolset` parameter (see section on Toolset Selection below).

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

This extension distinguishes mechanically enforced selection from publisher conformance. The protocol mechanically resolves an exact `(name, version)`, filters `tools/list` by that Toolset's membership, and rejects `tools/call` for names outside that membership. Compatibility between Toolset versions is a guarantee made by the publisher. This extension does not snapshot tool descriptors, require version-specific implementations, or prevent a server from publishing a non-conformant change. A publisher that violates the requirements below is non-conformant with this extension.

For a given `(name, version)`:

1. The `tools` membership **MUST NOT** change after publication.
2. While the Toolset version remains published, the server **MUST NOT** introduce a breaking change to a member tool's wire contract. Breaking changes include renaming the tool; changing `inputSchema` in a way that rejects previously valid arguments; adding runtime validation that rejects previously valid arguments even when the schema is unchanged; changing `outputSchema` incompatibly; returning content that no longer conforms to the prior output contract; or changing documented result or content semantics in a way that invalidates existing consumers.
3. Changing membership or a member tool's wire contract while retaining the same `(name, version)` **violates this extension's SemVer intent**. Such changes **MUST** be published as a **new** Toolset version: **MINOR** when only adding tools to the surface; **MAJOR** when removing tools or breaking contracts of existing members. Servers **MAY** continue to serve prior versions concurrently.

Schema and semantics compatibility is a **publication discipline** tied to Toolset versions, not a per-tool version registry. Hosts that need a cryptographically verifiable snapshot of Toolset membership and member tool descriptors should consider a future content `digest` (see Open Questions).

#### SemVer Rules for Toolset Versions

These rules apply to the Toolset package, not to individual tools:

| Change                                                                                                                                           | Version impact                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| Remove a tool from membership, or intentionally break a member tool contract while retiring the old surface                                      | **MAJOR**                                                      |
| Add tools while retaining every member and preserving the contracts of carried-forward tools from earlier versions in the same major family      | **MINOR**                                                      |
| Change `title` or `description` without changing membership or breaking member contracts                                                         | **PATCH**; servers **SHOULD NOT** mutate these fields in place |
| Change `status` or `deprecationDate`                                                                                                              | None; lifecycle metadata **MAY** mutate in place               |

A Toolset version with greater SemVer precedence within the same major family **MUST** retain every tool from lower versions in that family and **MUST NOT** introduce breaking contract changes to carried-forward tools. These are publisher conformance requirements; clients are not required to compare versions or validate them.

#### Operational Guidance

Servers that publish Toolsets take on a small operational contract beyond today's flat tool list:

1. Servers **MAY** serve multiple versions of the same Toolset name concurrently. Clients select with an exact pin; the server does not resolve ranges in v1.
2. If a server continues publishing an older Toolset version after introducing a breaking change in a new **MAJOR** version, it **MUST** continue honoring the older version's membership and member contracts. This extension does not prescribe whether the server uses separate implementations, compatibility adapters, version-aware routing, or another internal mechanism. A server unable to preserve the older contract must retire that version.
3. Before removing a version from publication, servers **SHOULD** mark it `deprecated` and **MAY** set `deprecationDate` so hosts can migrate pins.
4. Servers **MAY** later omit a `(name, version)` from `toolsets/list`. Clients still pinned to that pair then receive `unknown_toolset` on pinned `tools/list` / `tools/call`.
5. This SEP does **not** require unbounded retention of every historical Toolset version. Retention and retirement are operator policy. Servers **SHOULD** document which versions they commit to keep available for consumers.
6. Storage and replication of published membership records are implementation details; the normative requirement is only that a published `(name, version)` keep a fixed `tools` membership as long as it appears in the response from `toolsets/list`.

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

##### Request Parameter

After confirming server support and advertising client support on the request, request params for both `tools/list` and `tools/call` **MAY** include a `toolset` field containing a `ToolsetRef`:

```json
{
  "toolset": {
    "name": "core-ops",
    "version": "1.2.0"
  }
}
```

##### `tools/list` Behavior

If `toolset` is present:

- The server **MUST** verify that `(name, version)` exists.
- The server **MUST** return only tools whose names are in that Toolset's `tools` membership.
- If a membership name has no corresponding registered tool, the server **SHOULD** omit it from the pinned `tools/list` result rather than failing the whole list.
- Membership filtering applies **before** pagination: any `cursor` for `tools/list` pages the filtered membership, consistent with other list methods.
- If the Toolset is unknown, the server **MUST** return an error (see Error Handling).

If `toolset` is absent, behavior is unchanged from core MCP.

##### `tools/call` Behavior

If `toolset` is present:

- The server **MUST** verify the Toolset exists.
- If `params.name` is not a member of that Toolset, the server **MUST NOT** execute the tool and **MUST** return an error.
- If the Toolset is unknown, the server **MUST** return an error (see Error Handling).

If `toolset` is absent, membership checks from this extension do not apply.

Hosts that pin a Toolset **SHOULD** pass the same `toolset` on both `tools/list` and `tools/call` so discovery and invocation stay aligned.

### Caching

[SEP-2549](./2549-TTL-for-list-results.md) caching semantics apply to pinned `tools/list` responses. As specified by SEP-2549, each page of a paginated response is independently cacheable and may have its own freshness metadata. Clients **MUST** treat responses that differ by Toolset selection, cursor, or any other response-varying request or context dimension as distinct cache variants. In particular, an unpinned request and requests for different Toolset `(name, version)` pairs **MUST** remain distinct. Implementations **MAY** deduplicate identical payload storage, provided that freshness and invalidation semantics remain correct for each variant.

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

The protocol mechanically enforces Toolset membership, while stability of member tool wire contracts is a publisher guarantee on the Toolset version. Breaking contracts **MUST NOT** be published in place and **MUST** mint a new **MAJOR** Toolset version. The extension does not prescribe a constraint engine or how a server internally preserves contracts for concurrently published versions.

### Why per-request pins instead of session-active Toolsets?

[SEP-2567](./2567-sessionless-mcp.md) removes protocol sessions and requires list endpoints not to vary by implicit session state. Carrying `toolset` on each request:

- keeps `tools/list` cacheable with an explicit cache key ([SEP-2549](./2549-TTL-for-list-results.md));
- works for hosts that create a connection per call;
- makes the pin auditable in logs without reconstructing session state.

Per-request selection does not prevent a host from configuring an exact Toolset pin per server as an application-level default and automatically including it on both `tools/list` and `tools/call` after confirming server support.

### Why exact versions only in v1?

Ranges invite resolution rules (latest matching, pre-release policy, conflict behavior) that complicated SEP-1575. Exact pins are trivial to implement, easy to audit, and sufficient for production freezes. Ranges can be considered later if demand is clear.

### Differentiation from rejected grouping SEPs

SEP-1300 / SEP-2084 proposed general-purpose groups/tags and filter syntax for organizing primitives. This SEP does not introduce a taxonomy DSL. It introduces a **versioned pin contract** whose primary purpose is stability and governance of the tool surface exposed to agents.

### Prior Art

Product MCP servers already expose informal "toolsets" as enablement bundles. Standardizing discovery (`toolsets/list`) and pin semantics makes those patterns interoperable across clients and servers.

## Backward Compatibility

Backward-compatible within MCP protocol revision `2026-07-28` and later:

- Clients and servers that ignore this extension behave exactly as today.
- Presence of Toolset fields on requests is optional and gated by per-request capability advertisement.
- No changes to the core meaning of unversioned `tools/list` / `tools/call`.
- Existing tools require no schema changes to be included in a Toolset.

This extension does not define Toolset support for `2025-11-25` or earlier protocol revisions, which use the initialization handshake superseded by SEP-2575.

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

## SDK Impact

Official SDKs typically provide both MCP client and server libraries. Expected impact:

- **Server libraries:** support opt-in enablement (disabled by default per [SEP-2133](./2133-extensions.md)); advertise the extension in server capabilities when enabled; implement `toolsets/list`; filter `tools/list` and enforce membership on `tools/call` when a `toolset` is supplied.
- **Client libraries:** discover server support through `server/discover`; advertise the extension in per-request client capabilities when listing Toolsets or using a pin; pass `toolset` on `tools/list` and `tools/call`; include `(name, version)` in the cache key for pinned `tools/list` results (see Caching).

## Performance Implications

- `toolsets/list` is a small additional list endpoint; servers with few Toolset versions should remain negligible in cost.
- Pinning can **reduce** `tools/list` payload sizes and token usage for agents by excluding non-member tools from model context.
- Per-request `toolset` adds small parameter overhead and makes membership checks O(membership) per call (trivial for typical sizes).

## Testing Plan

Interoperable implementations **SHOULD** cover:

1. Advertise `io.modelcontextprotocol/toolsets` through `server/discover` and per-request client capabilities; reject extension-dependent requests lacking client advertisement with `-32021`.
2. `toolsets/list` returns published Toolsets; filters by `name` / `status`.
3. `tools/list` without `toolset` returns the full tool list.
4. `tools/list` with a valid pin returns only registered tools from the target Toolset version's membership.
5. `tools/list` / `tools/call` with unknown `(name, version)` errors.
6. `tools/call` for a non-member under a pin errors; member succeeds.
7. Immutability: republishing the same `(name, version)` with different membership is rejected or treated as a server bug in conformance tests.
8. Concurrent Toolset versions: pinning `1.2.0` does not observe tools only added in `1.3.0`.
9. Pinned `tools/list` omits membership names that have no registered tool (rather than failing the list).

These cases test the mechanically enforced protocol behavior. Publisher test suites **SHOULD** additionally verify that versions within a major family retain prior membership and preserve the contracts of carried-forward tools. Generic protocol conformance tests cannot determine whether arbitrary implementation behavior or content semantics remain compatible.

## Alternatives Considered

1. **Revive SEP-1575 and compose Toolsets as SemVer BOMs.** Rejected for v1: depends on dormant machinery; reopens constraint-resolution complexity; does not match reviewer guidance favoring higher-level contracts.
2. **Session-scoped "active Toolset".** Rejected: conflicts with sessionless MCP and list cacheability.
3. **Server-only** `Implementation.version` **bumps.** Helpful but too coarse when one server publishes multiple capability products or must keep an old surface alive while shipping a new one.
4. **Client-side allowlists only.** Useful locally, but not discoverable, not enforceable server-side, and not interoperable.
5. **Informational-only Toolset metadata without call enforcement.** Insufficient against hosts that still call whatever the model selects from a broader ambient list.

## Open Questions

1. Should `tools/list_changed` (or subscription filters) be Toolset-scoped when a pin is active, or always describe the full server catalog?
2. Should v1 allow an optional content `digest` on `Toolset` covering membership and member tool descriptors (for supply-chain pinning of a Toolset snapshot)?
3. Prefer passing `ToolsetRef` directly on `tools/list` and `tools/call` versus introducing `toolsets/select`, which returns a handle to pass on subsequent requests? Both approaches are compatible with sessionless MCP. The current draft prefers direct parameters because exact Toolset references are already compact identifiers and avoid an additional round trip and handle-lifecycle semantics.
4. How should extension-specific JSON-RPC error `code` integers be coordinated across official SDKs, given that `data.reason` is already the stable cross-implementation signal?

## Acknowledgments

This proposal benefits from the discussion on [SEP-1575](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1575) (especially arguments for server-/bundle-level versioning over per-tool SemVer) and from product patterns that already group MCP tools into named enablement sets.