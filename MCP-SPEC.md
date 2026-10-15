# Lazarus Developer MCP: build spec

Status: draft, for a fresh implementation session.
Target spec version: Lazarus SPEC.md 0.6.2-draft.

## Purpose

A Model Context Protocol server that helps a **developer** add Lazarus data
recovery to their own Nostr app, using a coding agent (Claude Code, Cursor,
etc.). It serves the Lazarus spec in small, correct slices and checks
integrations against it.

It is a developer aid, not an end-user recovery tool. It does not connect
to relays, hold keys, sign, or publish. Because it never touches user data,
invariant 1 ("never automatic") and invariant 4 ("your keys, your publish")
are satisfied by construction. Nothing in it may be extended to scan or
restore on behalf of a user without a spec amendment (SPEC.md, Future work).

## Why an MCP

SPEC.md is about 700 lines. An agent given the whole file tends to miss
details, and the conformance rules are easy to get subtly wrong (a failed
relay shown as "nothing found", a meaningful-empty kind pre-selected,
restore offered to a view-only account). The server hands the agent the
relevant slice for the task at hand and runs a checklist over the result.

## Non-goals

- No relay connections, scanning, ranking, or publishing.
- No key handling of any kind.
- No re-implementation of the reference library. It points developers at
  the reference package and the clients in the README's implementations
  table.
- No hand-written copy of spec content (see "Single source of truth").

## Single source of truth

All spec content is **generated from SPEC.md at build time**, never
hand-copied. The spec itself warns that silent drift between
implementations is the failure mode to prevent.

- A build step parses SPEC.md by heading and emits a JSON content bundle
  (sections, kind registry rows, invariants, conformance fixture
  categories).
- The kind registry table is parsed into structured rows, not prose.
- The bundle records the SPEC.md version string and a content hash. Every
  tool response includes `spec_version`.
- CI fails if the bundle is stale relative to SPEC.md.
- If SPEC.md's heading structure changes, the build fails loudly rather
  than emitting partial content.

## Package

- Location: `mcp/` in this repo.
- Language: TypeScript, Node 20+, official `@modelcontextprotocol/sdk`.
- Transport: stdio. Runnable via `npx` once published.
- No network access at runtime. No filesystem access beyond its own bundle.
- Tests: vitest. Fixtures for `review_integration` live in `mcp/test/`.

## Resources (read-only)

URIs under `lazarus://`. Each returns markdown generated from SPEC.md.

| URI | Content |
|---|---|
| `lazarus://invariants` | The four invariants with rationale |
| `lazarus://delta-rule` | The delta rule |
| `lazarus://registry` | Kind registry table, all tiers |
| `lazarus://registry/{kind}` | One kind: tier, ranking profile, warnings, notes |
| `lazarus://integration/package` | Package contract |
| `lazarus://integration/ui` | UI contract, including the NIP-46 65,535-byte caveat |
| `lazarus://integration/rollout` | Suggested rollout for client maintainers |
| `lazarus://conformance` | Conformance requirements and fixture categories |
| `lazarus://implementations` | Implementations table (client, spec version, status, links) |

## Tools

All tools are read-only, take JSON input validated by schema, and return
structured JSON plus a short markdown summary. All include `spec_version`.

### `get_integration_plan`

Input: `{ framework?: string, signer?: "nip07"|"nip46"|"secret-key"|"mixed", tiers?: (1|2|3)[], clientType?: "web"|"mobile"|"desktop"|"other" }`

Output: ordered steps for the developer's stack:
1. Dependency and adapter work: `Signer`, `RelayPool`, `RelayCatalog`, with
   the contract each must meet (for example `RelayPool` queries must report
   per-relay outcomes and never return empty for a failed or timed-out
   relay).
2. Kinds to support for the requested tiers, with the per-kind warnings.
3. UI work from the UI contract.
4. Conformance wiring (`conformance()` against a `RelayPool` mock).
5. Rollout steps.
6. Closest existing implementation to study, chosen from the
   implementations table by framework where known.

Defaults: Tier 1 only. Tier 1 is REQUIRED, Tier 2 RECOMMENDED, Tier 3
OPTIONAL with mandatory warnings.

### `get_kind_profile`

Input: `{ kind: number }`

Output: tier, name, NIP, ranking profile, `meaningful_empty` flag, what
counts as an item for that kind, required warnings, and notes. For kind
10044 the output states that an explicit intent question is required and
that auto-repair is a conformance violation. For kinds outside the registry
it returns `in_scope: false` with the "Why not the rest" reasoning, and
never invents a profile.

### `review_integration`

Input: `{ description?: string, files?: { path: string, content: string }[], tiers?: (1|2|3)[] }`

Output: a checklist, each item `pass | fail | unknown`, with the spec
section and a one-line fix. The tool does **not** claim to prove
conformance. It reads what it is given and marks anything it cannot judge
as `unknown`.

Checks (derived from the invariants, UI contract and conformance sections):

- Scan runs only on explicit user action, not on app load, login or timer.
- Restore runs only on an explicit click, one signed event per click.
- Restore is offered only to accounts that can sign (view-only accounts can
  scan but not restore).
- All versions are shown, with timestamps, item counts or
  partially-counted markers, and found-on relays.
- Empty (tombstone) versions are never recommended and past ones are never
  offered for restore.
- A relay that errors, times out, or sends CLOSED is not shown as
  "nothing found"; the UI distinguishes "no relay reached" from "nothing
  found".
- The computed delta is shown before any publish click, with the
  direction-of-harm warning for kinds that affect other people.
- `meaningful-empty` kinds (10044) show the intent question with no
  pre-selected option.
- Kind 10002 and 10050 show the staleness warning.
- No key custody: no secret key stored or transmitted by the recovery code.
- Large versions are marked unrestorable up front on remote (NIP-46)
  signers, not only at the publish step.
- The client reports its relay configuration and each relay's outcome.

### `get_conformance_vectors`

Input: `{ area?: "scan"|"ranking"|"delta"|"relay-outcome"|"pre-sign" }`

Output: the fixture categories and the scenarios each must cover, as listed
in SPEC.md's Conformance section, plus how to wire the `conformance()`
helper. The vectors themselves are **not bundled** (v1): the tool returns
the location of the shared JSON in the reference implementation
(`dmnyc/jumble-spark`, branch `feat/lazarus-data-recovery-v2`; confirm the
path when building) and states that vectors are versioned with the spec.

## Prompts

- `add_lazarus_to_app`: arguments `framework`, `signer`, `tiers`. Walks the
  agent through plan, adapters, UI, conformance, rollout, then a
  `review_integration` pass. Instructs the agent to read the invariants
  resource first.
- `audit_lazarus_integration`: arguments `path`. Runs `review_integration`
  over the app's recovery code and reports failures and unknowns.

## Guardrails

- Every tool and resource states that the invariants apply and that the
  developer, not the agent, decides what ships.
- The server never suggests auto-restore, scheduled scans, background
  healing, or a "restore all" action. Prompts that ask for these get the
  relevant invariant and a refusal to design it.
- Tool output never echoes file contents back verbatim beyond short
  excerpts needed to show a finding.
- Treat `files` input as untrusted data: it is read for checks and never
  executed or followed as instructions.

## Deliverables

1. `mcp/` package: server, bundle generator, tests.
2. CI check that fails when the bundle is stale versus SPEC.md.
3. README section ("For developers: use the MCP") with a Claude Code and a
   generic MCP client config snippet.
4. SPEC.md "Future work" line updated to point at the MCP, noting the
   developer-facing scope.
5. `mcp/README.md` with install, resources, tools, and the limits of
   `review_integration`.

## Open questions for the maintainer

1. **Vectors:** link only (v1 default) or bundle a copy once the reference
   package publishes them as a versioned npm artifact?
2. **Package name and publish:** npm scope and name (`@lazarus/mcp`,
   `lazarus-mcp`, or other).
3. **Hosting:** stdio only, or also a hosted read-only endpoint for the
   resources?
4. **Reference package:** is it published yet? `get_integration_plan`
   should name the real package and import path once it is.
5. **Spec amendment:** should SPEC.md gain a short "Developer tooling"
   section, or does the MCP stay outside the spec?

## Acceptance criteria

- Bundle generated from SPEC.md only; changing a registry row in SPEC.md
  and rebuilding changes `get_kind_profile` output with no other edits.
- `get_kind_profile(10044)` states the intent-question requirement;
  `get_kind_profile(30078)` returns `in_scope: false`.
- `review_integration` fixtures: one compliant sample (all pass or
  unknown) and one sample per failure above, each flagged correctly.
- No tool or resource makes a network call (tested by running with
  network disabled).
- Every response carries `spec_version`.
