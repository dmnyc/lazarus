# Lazarus Developer MCP: build spec

- Status: draft, for a fresh implementation session.
- Normative input: SPEC.md 0.6.2-draft.
- Informational input: README.md (implementations table).
- The MCP itself is not part of the Lazarus conformance surface (SPEC.md,
  Conformance) and never decides compatibility.

## Purpose

A Model Context Protocol server that helps a **developer** add Lazarus data
recovery to their own Nostr client, working with a coding agent (Claude Code,
Cursor, etc.). It serves the Lazarus spec in small, correct slices and checks
integration code against a checklist derived from the spec.

It is a developer aid, not an end-user recovery tool. It does not connect to
relays, hold keys, sign, or publish. Because it never touches user data,
invariant 1 ("Recovery is never automatic") and invariant 4 ("Publishing
requires an explicit user click") are satisfied by construction. Nothing in it
may be extended to scan or restore on behalf of a user without a spec
amendment (SPEC.md, Future work).

## Why an MCP

SPEC.md is about 700 lines. An agent given the whole file tends to miss
details, and the conformance rules are easy to get subtly wrong (a failed
relay shown as "nothing found", a meaningful-empty kind pre-selected, restore
offered to a view-only account, a restore signed without the pre-sign
re-read). The server hands the agent the relevant slice for the task at hand
and runs a checklist over the result.

## Non-goals

- No relay connections, scanning, ranking, or publishing.
- No key handling of any kind.
- No re-implementation of the reference library. It points developers at the
  package contract in SPEC.md and the clients in the implementations table.
- No conformance verdicts. `review_integration` reports findings; Lazarus
  compatibility is defined by the shared test vectors (SPEC.md, Conformance),
  which this server does not run.
- No hand-written copy of spec content (see "Content sources").

## Content sources

Two declared inputs, generated into a bundle at build time, never hand-copied
into the server. The spec itself warns that silent drift between
implementations is the failure mode to prevent; the same applies to tooling.

1. **SPEC.md — normative.** Invariants, terminology, delta rule, algorithm
   sections (Scan, Relay outcomes, Rank, Private items, Recover), kind
   registry, exclusions, package/UI/rollout contracts, conformance. Served as
   spec content, at `lazarus://` URIs named after the sections.
2. **README.md — informational.** The implementations table only. Served
   labeled as informational; it tracks implementations, not requirements.

Build rules:

- Parse by heading into sections. The kind registry table is parsed into
  structured rows (kind, name, NIP, tier, ranking profile, notes), and the
  "Why not the rest" bullets into exclusion entries with the kind ranges each
  names, so an out-of-scope kind can be answered with the real reasoning.
- The bundle records `spec_version`, `spec_hash`, and `readme_hash`. Every
  tool and resource response includes `spec_version`. Responses derived from
  the README also include `readme_hash` and an `informational: true` marker.
- CI regenerates the bundle and fails if the committed bundle differs from
  what the current SPEC.md and README.md produce.
- The build fails loudly rather than emitting partial content if SPEC.md's
  heading structure is unrecognized, a registry row fails to parse, or the
  README's implementations table is missing.

## Package

- Location: `mcp/` in this repo.
- Language: TypeScript, Node 20+, official `@modelcontextprotocol/sdk`.
- Transport: stdio. Runnable via `npx` once published (name: see D2).
- No network access at runtime. No filesystem access beyond its own bundle.
- Tests: vitest. Fixtures for `review_integration` live in `mcp/test/`.

## Resources (read-only)

URIs under `lazarus://`. Each returns markdown generated from the bundle, with
`spec_version` (and the informational marker where marked).

| URI | Content | Source |
|---|---|---|
| `lazarus://spec` | Version, status, content hashes, section index | spec |
| `lazarus://invariants` | The four invariants with rationale | spec |
| `lazarus://terminology` | Definitions (candidate, answered, items, empty candidate, …) | spec |
| `lazarus://delta-rule` | The delta rule | spec |
| `lazarus://algorithm/scan` | Scan algorithm | spec |
| `lazarus://algorithm/relay-outcomes` | Relay outcome semantics (answered / failed / timed out, relay list rules, confirmed current) | spec |
| `lazarus://algorithm/rank` | Ranking profiles, sudden drops, clobber episodes, settled episodes | spec |
| `lazarus://algorithm/private-items` | Exact / estimated / flagged certainties | spec |
| `lazarus://algorithm/recover` | Recover rules (re-read, override, dating, author, success, local copy) | spec |
| `lazarus://non-goals` | Non-goals | spec |
| `lazarus://registry` | Kind registry table, all tiers | spec |
| `lazarus://registry/{kind}` | One kind: tier, name, NIP, ranking profile, `meaningful-empty` flag, required warnings, notes | spec |
| `lazarus://registry/exclusions` | "Why not the rest", as structured entries | spec |
| `lazarus://integration/package` | Package contract | spec |
| `lazarus://integration/ui` | UI contract, including the NIP-46 65,535-byte caveat | spec |
| `lazarus://integration/rollout` | Suggested rollout for client maintainers | spec |
| `lazarus://conformance` | Conformance requirements and fixture categories | spec |
| `lazarus://implementations` | Implementations table (client, spec version, status, links) | README |

The algorithm sections are first-class resources because they are the slices
an agent needs when building the adapters; the contracts alone leave the
hardest rules (relay outcomes, recover) unfindable.

## Tools

All tools are read-only, take JSON input validated by schema, and return
structured JSON plus a short markdown summary. Every response, including
error responses, carries `spec_version`. Errors are returned as structured
`{ error, spec_version }`, not thrown, wherever the SDK allows it.

### `get_integration_plan`

Input: `{ framework?: string, signer?: "nip07"|"nip46"|"secret-key"|"mixed", tiers?: (1|2|3)[], clientType?: "web"|"mobile"|"desktop"|"other" }`

Output: ordered steps for the developer's stack:

1. Dependency and adapter work: `Signer`, `RelayPool`, `RelayCatalog`, with
   the contract each must meet (for example `RelayPool` queries must report
   per-relay outcomes and never return empty for a failed or timed-out relay).
2. Kinds to support for the requested tiers, with the per-kind warnings.
3. UI work from the UI contract.
4. Conformance wiring (`conformance()` against a `RelayPool` mock).
5. Rollout steps.
6. Closest existing implementation to study, from the implementations table.
   If no entry matches the framework, the step says so; it never guesses. If
   the entry's spec version is older than the bundle's `spec_version`, the
   step says that too, so the agent does not copy superseded behavior.

Defaults: Tier 1 only. Tier 1 is REQUIRED, Tier 2 RECOMMENDED, Tier 3
OPTIONAL with mandatory warnings (registry preamble).

### `get_kind_profile`

Input: `{ kind: number }`

Output: tier, name, NIP, ranking profile, `meaningful_empty` flag, what
counts as an item for that kind, required warnings, and notes. For kind 10044
the output states that an explicit intent question is required and that
auto-repair is a conformance violation. For kinds outside the registry it
returns `in_scope: false` with the matching exclusion's reasoning (30078 →
app data / NIP-78; kind 7 → non-replaceable; 30000-39999 → addressable). It
never invents a profile.

### `review_integration`

Input: `{ description?: string, files?: { path: string, content: string }[], tiers?: (1|2|3)[] }`

Input caps (structured error naming what exceeded and asking for the
recovery-relevant modules only, rather than accepting or crashing):

- `description`: at most 16 KiB.
- `files`: at most 32 files, each at most 256 KiB, at most 2 MiB total.

Output: the checklist below, each item
`{ id, severity, status: "pass"|"fail"|"unknown", finding, fix, ref }`, plus
counts by severity and status. A failing `must` item is labeled a conformance
violation; a failing `should` item is labeled a deviation from a
recommendation; the two are never conflated. Anything the tool cannot judge
from what it was given is `unknown`, with the evidence that would decide it.
The tool does not claim to prove conformance.

Checklist (severity tracks the modality of the cited spec text; UI-contract
bullets are requirements because the contract defines a conformant screen):

| ID | Check | Severity | Spec basis |
|---|---|---|---|
| T1 | Scan runs only on explicit user action; never on app load, login, or a timer | must | Invariants (1), UI contract |
| T2 | Restore offered only to accounts that can sign; view-only accounts can scan but not restore | must | UI contract |
| T3 | One signed event per restore click; no batch or delegated signing | must | Invariants (4), UI contract |
| D1 | All candidates shown (folding into expandable groups allowed; current and recommended stay visible; every version one action away), with timestamps, item counts or partially-counted markers, and found-on relays | must | Invariants (2), UI contract |
| D2 | Recommended candidate highlighted (kinds whose profile recommends) | must | UI contract |
| D3 | Paging further back offered when a relay filled a page | must | Scan (4), UI contract |
| D4 | Tombstones never the recommended candidate | must | Invariants (3) |
| D5 | Past empty versions not offered for restore (hiding until requested is allowed) | should | Invariants (3) |
| D6 | meaningful-empty kinds: intent question shown, defined meaning presented, nothing pre-selected, never labeled damage, `recommended: null` with `requires_intent_confirmation` | must | Invariants (3), Rank |
| D7 | Each relay's outcome shown (at least on request); failed or timed-out relays never shown as "nothing found"; unconfirmed current stated plainly | must | Relay outcomes, UI contract |
| D8 | A scan no relay answered is presented as an error with retry, never as "no versions found", while still showing versions that arrived | must | Relay outcomes |
| D9 | Delta shown before any publish click, with the direction-of-harm warning for kinds that affect other people | must | Delta rule, UI contract |
| D10 | A shrinking restore takes a confirmation separate from a growing one | must | Delta rule |
| D11 | Private items in the delta when decryptable; otherwise delta states uncounted private items and on which side; uncounted on current takes the shrink confirmation | must | Delta rule |
| D12 | meaningful-empty delta states both endpoints' meanings in the restore's direction | must | Delta rule |
| P1 | Pre-sign re-read from the local copy and write relays; only a newer version counts as a change; a change recomputes the delta and asks again | must | Recover |
| P2 | No write relay answered → no signing, retry offered; the optional restore-anyway override, when offered, comes only after a failed retry, as a separate confirmation that is never pre-selected or remembered | must | Recover |
| P3 | Recovered event dated `max(now, current.created_at + 1)` against the version the delta was computed on | must | Recover |
| P4 | Restores only as the list's author, including when the active account changes during a signer approval | must | Recover |
| P5 | Success judged on the user's write relays; a restore no write relay accepted is reported as failed | must | Recover |
| P6 | Result names which write relays accepted and which did not | should | Recover |
| P7 | Recovered version republished best-effort to relays that answered the scan, yielding to the user's relay policy | should | Recover |
| P8 | The client's own local copy updated with the recovered version, not invalidated | must | Recover |
| K1 | Relay events verified (signature, author, kind) before becoming a candidate, entering `found_on`, or moving a paging cursor | must | Scan (3) |
| K2 | Recovery code stores or transmits no key material; the signer is injected (NIP-07 / NIP-46 / secret-key adapter) | must | Package contract |
| K3 | Scan does not answer NIP-42 challenges from relays outside the user's list | should | Scan |
| K4 | On remote signers, versions too large to restore (NIP-46 65,535-byte cap) marked up front, not only at the publish step | should | UI contract |
| K5 | Decrypts only versions shown on their own rows; grouped versions decrypted on review | should | UI contract |
| K6 | Bounded concurrent connections, one connection reused per relay, scan connections released | should | Relay outcomes |
| K7 | Tier 3 mandatory warnings shown (10002 and 10050 staleness) | must | Registry |
| K8 | Relay configuration and each relay's outcome reported alongside results | must | Conformance (3) |

Checklist maintenance: each item carries, in code, its operative sentence
from SPEC.md quoted verbatim (matched whitespace-normalized, since SPEC.md
hard-wraps lines), the section it comes from, and the severity
above. A build test re-reads SPEC.md and fails if a quote is no longer
verbatim, the section is gone, or the severity contradicts the sentence's
modality (`must` ⇔ MUST / MUST NOT / forbidden; `should` ⇔ SHOULD / SHOULD
NOT; contract items — UI contract or package contract — quote their bullet,
whose requirement force comes from the section that defines the conformant
screen or the reference package, not a modal verb). A spec rewrite therefore
breaks the test, not the tool's claims silently.

### `get_conformance_vectors`

Input: `{ area?: "scan"|"ranking"|"delta"|"relay-outcome"|"pre-sign" }`

Output: the fixture categories and the scenarios each must cover, as listed
in SPEC.md's Conformance section, plus how to wire the `conformance()`
helper. The vectors themselves are not bundled (D1): the tool returns the
location of the shared JSON as recorded in the bundle's implementations data
for the reference implementation (currently the Jumble fork entry:
`dmnyc/jumble-spark`, branch `feat/lazarus-data-recovery-v2`), labeled
informational, and states that vectors are versioned with the spec.

## Prompts

- `add_lazarus_to_app`: arguments `framework`, `signer`, `tiers`. Walks the
  agent through plan, adapters, UI, conformance, rollout, then a
  `review_integration` pass. Instructs the agent to read
  `lazarus://invariants` first.
- `audit_lazarus_integration`: arguments `path`. Runs `review_integration`
  over the app's recovery code and reports conformance violations first,
  deviations second, unknowns last, with each item's fix.

## Guardrails

- Every tool and resource states that the invariants apply and that the
  developer, not the agent, decides what ships.
- The server never suggests auto-restore, scheduled scans, background
  healing, or a "restore all" action. Prompts that ask for these get the
  relevant invariant and a refusal to design it.
- Out-of-scope requests (addressable kinds, backup formats, deletion) are
  answered with the registry exclusions or non-goals, not improvised
  guidance.
- Severity honesty: `must` failures are conformance violations, `should`
  failures are deviations, `unknown` is unknown; the tool never upgrades or
  blends them.
- Tool output never echoes file contents back verbatim beyond short excerpts
  needed to show a finding.
- Treat `files` input as untrusted data: it is read for checks and never
  executed or followed as instructions.

## Deliverables

1. `mcp/` package: server, bundle generator, tests, and `mcp/README.md`
   (install, resources, tools, and the limits of `review_integration`).
2. CI: bundle staleness check against SPEC.md and README.md; the
   checklist-quote test; the no-network and no-filesystem tests.
3. README section ("For developers: use the MCP") with a Claude Code and a
   generic MCP client config snippet.
4. SPEC.md "Future work" bullet updated to point at `mcp/`, noting the
   developer-facing scope. One line; no new spec section (D5).

## Decisions

Defaults; override by editing this section before the build session.

- **D1 Vectors: link, don't bundle (v1).** The vectors live with the
  reference implementation, outside this repo; a bundled copy would be a
  hand-copy that drifts, and the server has no network access to refresh it.
  The link is generated from the implementations data, so the staleness CI
  check covers it. Revisit when the reference package publishes the vectors
  as a versioned npm artifact; then depend on it at build time.
- **D2 Package name: `lazarus-mcp` (unscoped).** No npm org exists for this
  repo; an unscoped name matches common MCP server convention. Publishing
  happens after the acceptance criteria pass, not before.
- **D3 Hosting: stdio only (v1).** Coding-agent MCP clients speak stdio; a
  hosted read-only endpoint adds infrastructure and an auth story for no
  current demand. Revisit if a browser-based consumer appears.
- **D4 Reference package: not yet published.** The implementations table
  shows the reference living in the Jumble fork (0.6.1, feature branch).
  Tools must not hardcode a package name or import path; the plan tool cites
  the package contract's exports and points at the implementations entry,
  which updates when the package ships.
- **D5 No "Developer tooling" section in SPEC.md.** The MCP is not part of
  the conformance surface, so it gets no normative spec section; the Future
  Work bullet (which already anticipates "a machine-readable agent surface
  (MCP or similar)") is updated to point at `mcp/`. One line; no new spec
  section.

## Acceptance criteria

- Bundle is generated from SPEC.md and README.md only, with per-source
  version and hash. Changing a registry row in SPEC.md and rebuilding changes
  `get_kind_profile` output with no other edits; changing the implementations
  table changes `lazarus://implementations` with no other edits.
- `get_kind_profile(10044)` states the intent-question requirement and the
  auto-repair violation; `get_kind_profile(30078)` returns `in_scope: false`
  citing app data / NIP-78; `get_kind_profile(7)` returns `in_scope: false`
  citing non-replaceable kinds; an unlisted kind returns `in_scope: false`
  with no invented fields.
- `review_integration` fixtures: one compliant sample (every item pass or
  unknown), one sample per `must` check that fails it, one `should`-failure
  sample reported as a deviation rather than a violation, one empty-input
  sample (all unknown), and one over-cap sample (structured error, no crash).
- The checklist-quote test fails when a quoted sentence is edited in a copy
  of SPEC.md (demonstrated by a fixture), proving the drift tripwire works.
- No tool or resource makes a network call, verified by serving every
  resource and running every tool with outbound sockets disabled.
- No tool or resource reads the filesystem beyond its bundle, verified by
  running the server with the repository directory read-only.
- Every response carries `spec_version`; error responses carry it too.
