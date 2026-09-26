# Lazarus

**Recovery of user data from relay history on Nostr**

- **Version:** 0.6.0-draft
- **Status:** DRAFT. Expect changes before 1.0; implementations should track the changelog.
- **Author:** @dmnyc
- **Licensing:** TBD (suggest CC0 for the spec, MIT for the reference library)
- **Home:** https://github.com/dmnyc/lazarus

## Abstract

Nostr clients overwrite replaceable events without reading them first. A client
that touches a user's follow list, mute list, or profile and publishes its own
version destroys every prior version on any relay that honors replacement.
Users lose social graphs, moderation state, and identity data to routine
client bugs.

The history, however, usually survives. Most relays retain superseded versions
of replaceable events, and archival relays keep them deliberately. Lazarus is
a specification for recovering that history: scan the user's relays for every
surviving version of a data kind, rank the candidates, recommend a recovery
target, and republish it with the user's own signer on an explicit click.

Lazarus is a contract plus a conformance suite. It is not a product, a relay,
or a background service.

## Terminology

- **Data kind:** a replaceable event kind holding user state (kind 0, 3,
  10000-19999). Addressable kinds (30000-39999) are out of scope for v1.
- **Candidate:** one distinct event of the target kind, authored by the user,
  found during a scan. Distinctness is by event id.
- **Current:** the candidate with the highest `created_at` in the scan
  results. Note: this is what the scan *saw*, and a relay that already dropped
  history may not show the true current event. Implementations SHOULD treat
  "current" as provisional, and MUST treat it as unconfirmed until at least
  one of the user's write relays has answered (see Relay outcomes).
- **Answered:** a relay answered a request when it sent EOSE for it, with or
  without events. A relay that failed to connect, closed the request or the
  connection first, or timed out did not answer, whatever it sent before.
- **Write relays:** the relays the user's relay list (kind 10002) marks for
  writing, including unmarked relays, which NIP-65 treats as both read and
  write.
- **Empty candidate:** a candidate whose item set is empty. For most kinds an
  empty candidate is a **tombstone** (evidence of a clobber). For kinds
  flagged `meaningful-empty` in the registry, an empty item set is a defined
  state with its own semantics, and the candidate is a **valid option**, not
  a tombstone.
- **Recovery:** publishing a chosen candidate's item set again, as a fresh
  signed event of the same kind, making it the newest version.

## The four invariants

Every Lazarus implementation MUST honor all four. Conformance is void if any
of them breaks.

1. **Recovery is never automatic.** A scan ranks and recommends. It does not
   publish. Not on a timer, not on detection, not "with permission granted
   earlier." Client features that *write* user-state kinds (the clobber
   vectors themselves) are outside Lazarus and out of its control, which is
   precisely why recovery must never be automatic: an auto-repair racing an
   auto-write is two clients fighting over the user's account.
2. **Everything found is shown.** Every candidate, including empty ones and
   candidates older or smaller than current. Hiding candidates to make the
   recommendation look better is a conformance failure. Versions MAY be
   folded into collapsed groups (runs of small edits, and clobber episodes,
   marked as such), and past empty versions MAY be hidden until the user
   asks for them, as long as every version stays one action away and the
   current and recommended versions stay visible. The user may choose any
   version they are shown, except a past empty version (see invariant 3).
3. **Tombstones are never recommended.** An empty version of the list is
   evidence, often the fingerprint of the clobbering client. It MUST NOT be
   the recommended candidate, and a past one (anything but current) SHOULD
   NOT be offered for restore at all, so a clobbered state can't be put
   back by accident. An empty current version is shown as it is: it is
   usually the clobber itself. **Exception:** for kinds flagged
   `meaningful-empty` (see registry), an empty candidate is a valid option
   that MUST be presented with its defined meaning, MUST NOT be
   auto-selected, and MUST NOT be labeled as damage.
4. **Publishing requires an explicit user click** that names what will be
   published, signed by the user's own signer (NIP-07, NIP-46, or
   equivalent). Batch or delegated signing of a recovery is forbidden.

## The delta rule

Before any recovery click, the implementation MUST display the delta between
the chosen candidate and current: how many items would be added, how many
would be removed, and, for kinds where items affect other people, a warning
naming the direction of the harm. Restoring an old follow list re-follows
accounts; that is mostly benign. Restoring an old mute list re-silences
accounts the user may have deliberately unmuted since; that is a moderation
action taken on the user's behalf and MUST be flagged as such. A recovery
that shrinks the list below current MUST require a separate confirmation
from one that grows it.

Items compare by type and value (`tag[0]` and `tag[1]`): a relay hint or
petname a client rewrote is not a change, or an identical follow list would
read as hundreds of follows added and removed. On relay lists the
read/write marker is part of the item, since it changes what the relay is
for. Profiles (kind 0) keep their data in `content`, so their delta is the
list of fields that would change: every field, not a fixed set, since
profile content is extensible and a restore replaces all of it. A fixed set
can report no change while the restore reverts fields it never compared.
Tags that would change count too (NIP-30 custom emoji live in a profile's
tags).

Private items are part of the delta. When the implementation can decrypt a
version's private items, the delta MUST include them: on a mute list they
can be most of the accounts a restore would re-silence. When it can't
(no key available, a decryption that failed or was denied, or a request
too large for a remote signer), the delta MUST state that private items are
uncounted.

For `meaningful-empty` kinds, the delta MUST additionally state the meaning
of both endpoints. Example for kind 10044: "This restores your NIP-4e
encryption keys. Clients will encrypt direct messages to them again." versus
"The current empty state announces that you do not use NIP-4e."

## Algorithm

### Scan

1. Collect relay set: every relay in the user's own relay list (kind
   10002, read and write, not only the first few outbox relays), a
   configurable default set, and a configurable archival set. A user's
   own relays usually keep only the latest version of a replaceable
   event, so a scan limited to them misses most of the history. The relay
   sets are configuration, not protocol; two implementations with
   different archival sets will see different histories and both are
   conformant. The user's relay list is the newest kind 10002 found (by
   `created_at`), in the implementation's own copy or on relays, not the
   first one to arrive; see Relay outcomes for a relay list that is
   missing or could not be fetched.
2. For each relay, request `kinds: [K], authors: [pubkey]` with a per-relay
   timeout (reference: 6000 ms) and a limit of at least 50. Implementations
   MUST NOT treat a partial relay response as a complete history. Record
   each relay's outcome (see Relay outcomes).
3. Relays are untrusted: a relay can return events outside the filter, or
   forged ones, and a restore would sign their content as the user's own.
   Implementations MUST verify each event's signature and discard it
   unless its author is the scanned account and its kind is the requested
   kind, before it can become a candidate, add a relay to a `found_on`
   list, or move a paging cursor. A relay that answered with only invalid
   events answered with nothing. The pre-sign re-read (see Recover)
   applies the same checks.
4. A relay that returns a full page may hold older versions.
   Implementations SHOULD let the user page further back from those
   relays on request, with `until` set to the oldest `created_at` that
   relay returned. `until` is inclusive, so the next page repeats that
   event; a relay whose next page brings nothing older is exhausted.
5. Deduplicate by event id. Preserve `found_on` relay lists per candidate
   and report each relay's outcome, not only the relays that returned
   events.

Informative: `wss://hist.nostr.land` and `wss://relay.ditto.pub` were
observed keeping full replaceable history (hundreds of versions of one
follow list) in 2026-09. Large public relays often still hold versions the
user's own relays already replaced.

### Relay outcomes

A relay that could not be reached and a relay with nothing to give look
the same to naive code: no events. Lazarus asks whether anything is there
in three places (the scan, the relay list lookup, and the re-read before
signing), and in all three it MUST tell the two apart. Every relay request
ends in exactly one outcome:

- **Answered:** the relay sent EOSE, with or without events.
- **Failed:** the connection could not be opened, or the relay closed the
  request (a NIP-01 `CLOSED`, such as NIP-42's `auth-required:`) or the
  connection before EOSE.
- **Timed out:** no EOSE arrived within the per-relay timeout.

Only an answered relay counts as having nothing. A failed or timed-out
relay MUST NOT count toward "nothing found", "no relay list", or
"unchanged since the review": silence is not evidence. Events a relay sent
before it failed or timed out are real signed versions and remain
candidates (invariant 2); the outcome records that the relay's history is
incomplete, not that what arrived is void.

- **Relay list.** If relays answered the relay list lookup and none
  returned a kind 10002, the user has no relay list: implementations MUST
  say so, and MAY use their default set as the write relays, labeled as
  defaults wherever write relays are shown. A relay list that names no
  write relays counts as missing. If no relay answered the
  lookup and the implementation holds no copy of the list, the relay list
  is unknown: implementations MUST NOT substitute their defaults for it.
  The scan still covers the default and archival sets, and current stays
  unconfirmed.
- **Confirmed current.** A scan result MUST set `current_confirmed: true`
  only when at least one of the user's write relays answered. Otherwise
  the newest version found may not be current, since a newer one can sit
  on the write relays the scan never reached. An unconfirmed current
  withholds the recommendation (see Rank) and gates the restore (see
  Recover).
- **Failed scan.** A scan in which no relay answered is a failed scan, not
  a result: implementations MUST present it as an error with a retry,
  never as "no versions found", while still showing any versions that
  arrived before the relays failed (invariant 2).
- **Retry.** Implementations SHOULD offer to retry the relays that failed
  or timed out, without repeating the whole scan.
- **Connection budget.** Implementations SHOULD bound concurrent
  connections, reuse one connection per relay across pages, and release
  connections opened only for the scan once it finishes or is canceled.

Informative: a scan opens many connections at once (the user's full relay
list, the default and archival sets, then paging), often on top of
connections the client already holds for other features. Browsers cap open
WebSocket connections, and a connection refused over the cap reaches the
page as a generic connection error, indistinguishable from a dead relay.
The rules above keep an exhausted pool from producing a wrong answer; they
cannot make it produce a complete one.

### Rank

Ranking is a **per-kind profile** (see the registry). Candidates are
listed newest first by default. For countable list kinds (3, 10000,
10003, 10006), implementations MAY also offer a size order: item count
descending, newer first on ties. Empty candidates are always shown
(invariant 2) and never recommended (invariant 3).

The `count` profile does not recommend a version just because it is
bigger: lists shrink through normal curation. It recommends one only when
the current version looks clobbered:

- A step between two consecutive versions (by `created_at`) is a
  **sudden drop** when the later version is missing at least 20% of the
  earlier version's items and at least 5 items, or is empty while the
  earlier one is not. Curation moves a few items at a time and never
  registers as a drop, however far a list shrinks over time.
- Walking back from the newest version, find the most recent sudden drop
  the current version hasn't recovered from: the current version is still
  missing at least 20%, and at least 5 items, of the version from just
  before the drop.
- Drops back to back, or within 24 hours of each other, form one
  **clobber episode**. Recommend the fullest version from just before any
  drop in that episode, so a list that was clobbered, partly restored, and
  clobbered again points at its fullest state before the damage.
- A clobber the list has since been edited on at least 5 times, over at
  least a week, is **settled**: the current version is the user's choice,
  and nothing is recommended.
- Sizes are compared conservatively: a drop uses the later version's
  maximum and the earlier version's minimum (see Private items), and
  nothing is recommended while the current version's size is unknown.
- Nothing is recommended while current is unconfirmed (see Relay
  outcomes). Drops are measured against current, and a scan that never
  reached the user's write relays may be measuring against a version the
  user already replaced.
- If no drop qualifies, "no recoverable improvement found" is the correct
  answer and MUST be presented as a normal result, not an error, alongside
  the relay outcomes it rests on. A failed scan is not this result (see
  Relay outcomes).

The thresholds (20%, 5 items, 24 hours, 5 edits over a week) are reference
values.
Implementations SHOULD use them, so that recommendations agree across
clients.

`recency` kinds are listed newest first with no recommendation: the user
picks.

For `meaningful-empty` kinds, ranking is FORBIDDEN: there is no "better"
without knowing user intent. All candidates are offered equally, each with
its meaning stated, and the user chooses. The scan result for these kinds
MUST set `recommended: null` and set a `requires_intent_confirmation: true`
flag.

### Private items

Kinds in the NIP-51 family carry private items: an encrypted `content`
alongside public `tags` (NIP-44, with NIP-04 legacy detection via the `iv`
marker). Private-only lists are common in practice, and a private-only
candidate has no public tags at all, so public counts alone read a full
list and an emptied one as the same zero. Implementations therefore
account for private items at one of three certainties:

1. **Exact.** If the implementation holds the user's key material and can
   decrypt a candidate's content, it counts the decrypted tag list and
   ranking MUST use the full count (public plus private).
2. **Estimated.** If content cannot be decrypted, the implementation
   SHOULD still derive an item-count range from the encrypted payload
   size: NIP-44 v2 payloads have 67 bytes of overhead and a
   power-of-two padded length, NIP-04 (AES-CBC) plaintexts are bounded
   to 1-16 bytes of padding per 16-byte block. With the assumed per-item
   JSON shape, the byte range yields a minimum and maximum item count.
   An estimate is enough to tell an emptied private list from a full
   one, which public counts alone cannot.
3. **Flagged.** Only when no key is available AND the payload cannot be
   sized MAY a candidate carry an unbounded private item set. Such a
   candidate MUST be surfaced as partially counted so the UI can warn
   that the recommendation may be wrong, and the delta MUST state that
   private items are uncounted.

Ranking compares total item counts as ranges derived from the three
certainties above, public tags plus private items, exact or estimated.
An implementation that ships only the flagged tier for a kind known to
carry private items is not conformant for that kind.

### Recover

Publishing is: take the chosen candidate's item set verbatim (tags, and
encrypted content if present and decryptable), construct a fresh event of
the same kind, sign with the user's own signer, and publish.

- Immediately before signing, implementations MUST re-read the current
  version, from their local copy and the user's write relays. The list has
  changed only if the re-read finds a version newer than the one the delta
  was computed against: the re-read asks fewer relays than the scan did, so
  an older copy is not a change. If it changed (an edit from another view,
  device or client), the newer version becomes current: recompute the
  delta against it and ask again, since restoring over it would silently
  drop those edits.
- The re-read MUST get an answer from at least one of the user's write
  relays, and SHOULD wait for every write relay up to the timeout rather
  than stopping at the first answer. A relay that answers with no events
  has answered (see Relay outcomes). The local copy cannot confirm current
  on its own: edits from other devices and clients may never reach it. If
  no write relay answers, the implementation MUST NOT sign, except through
  the override below; it says the current version could not be confirmed
  and offers a retry. An absent answer is not an unchanged one.
- After a retry fails, an implementation MAY let the user restore anyway,
  as a separate explicit confirmation stating that the current version
  could not be confirmed and that edits made since the review may be
  lost. That confirmation MUST NOT be pre-selected or remembered between
  restores. The override exists because relay lists naming only dead
  relays are common, and restoring an older relay list is often the fix.
- The recovered event's `created_at` MUST be later than the version it
  replaces: `max(now, current.created_at + 1)`, where current is the
  newest version known at signing (the one the delta was computed
  against), never an older copy the re-read found. Clobbering clients often
  have skewed clocks, and an older timestamp loses to the clobbered version
  on relays and in caches.
- The signing account MUST be the list's author. A client with several
  accounts MUST NOT restore one account's list as another's, including when
  the active account changes during a signer approval.
- Success is judged on the user's write relays: a restore succeeds when at
  least one of them accepts the event (NIP-01 `OK` true), and the result
  SHOULD name which write relays accepted it and which did not. A restore
  that no write relay accepted is reported as failed, whatever other
  relays did. For kind 10002, whose restore replaces the write relays
  themselves, implementations MAY judge success on the write relays the
  restored version names. The recovered version SHOULD also go to every
  other relay that answered the scan, as a best effort that doesn't affect
  the result: those relays hold older copies and keep serving the
  clobbered one otherwise.
- Implementations MUST update their own local copy of the list with the
  recovered version. Otherwise the client's next edit rebuilds from the
  clobbered copy and clobbers the list again.

## Kind registry

Tier 1 kinds are REQUIRED for conformance. Tier 2 kinds are RECOMMENDED.
Tier 3 kinds are OPTIONAL and carry mandatory warnings. Everything not
listed is out of scope until this document is amended.

| Kind | Name | Tier | Ranking profile | Notes |
|---|---|---|---|---|
| 3 | Follow list (NIP-02) | 1 | count: clobber detection | The reference implementation. `p` tags; `content` may hold relay hints, preserve verbatim. |
| 10000 | Mute list (NIP-51) | 1 | count: clobber detection | Private items apply. Delta rule applies with re-mute warning. |
| 0 | Profile metadata (NIP-01) | 2 | recency, user picks | Size ranking is meaningless here; profiles change legitimately and often. Show field-level diffs between candidates and current, covering every field and tag that would change (see the delta rule). Highest rogue-client casualty rate. |
| 10003 | Bookmarks (NIP-51) | 2 | count: clobber detection | Private items apply. `e` and `a` tags. High user pain, zero effect on others. |
| 10044 | Encryption key list (NIP-4e, draft) | 2 | none: intent confirmation required (`meaningful-empty`) | Empty = "I no longer use NIP-4e" is a defined state, not damage (invariant 3 exception). Recovery or re-emptying MUST be preceded by an explicit intent question. Auto-repairing this kind is a conformance violation even for clients that implement NIP-4e. Display: show the `p`-tagged encryption pubkeys per candidate. |
| 10002 | Relay list (NIP-65) | 3 | recency, user picks | Mandatory staleness warning: an old relay list can strand the user on dead relays and silently break event delivery. Implementations SHOULD liveness-check candidate relays before recommending. |
| 10050 | DM relay inbox (NIP-17) | 3 | recency, user picks | Same staleness warning as 10002; a wrong inbox list silently breaks DM delivery. |
| 10006 | Blocked relays (NIP-51) | 3 | count: clobber detection | Low stakes. |

### Why not the rest

- **Addressable kinds (30000-39999):** real clobber risk, but each `d` tag
  has app-specific semantics and no generic ranking exists. Future work,
  and this spec MUST NOT grow it "by accident."
- **Pins (10001), communities (10004), chats (10005), badges (10008),
  groups (10009), interests (10015), emoji (10030), media follows (10020):**
  recoverable by the same machinery, but low pain and low frequency. Each
  would be a standards fight for little return. Left to future amendments.
- **App data (30078):** this is where proactive *backups* live (NIP-78).
  Lazarus recovers history; it does not define backup formats. Different
  concern, different spec, no mixing.
- **Non-replaceable kinds (notes, reactions, zaps):** rogue clients cannot
  erase these by replacement. Hiding them is a client bug; deleting them is
  NIP-09 and explicitly out of scope.

## Client integration

Lazarus is designed to ship **inside** clients, as part of app settings. A
client developer (Jumble, Amethyst, Gossip, a web client fork) should be
able to add a "Data recovery" screen with a library dependency and a
handful of UI components' worth of work.

### Package contract

The reference implementation is published as a framework-agnostic package
(TypeScript, zero UI dependencies; pool and signer are injected, not
bundled). It exports:

- `scan(kind, pubkey, options)` and `rank(candidates, profile)` — pure
  functions, the conformance surface.
- `computeDelta(chosen, current, profile)` — pure.
- `checkCurrent(reviewed, local, answers)` — pure: decides the pre-sign
  re-read from the local copy and each write relay's outcome and events
  (proceed, changed with the newer version, or unconfirmed).
- `buildRecoveryEvent(chosen, current)` — returns an unsigned event
  template, dated after current.
- Registry access: `getProfile(kind)` returns the ranking profile, tier,
  `meaningful-empty` flag, and required warnings, so clients never hardcode
  kind semantics. The core library hardcodes no kind numbers; the registry
  is the single source of truth and is versioned with the spec.
- Adapter interfaces: `Signer` (NIP-07 / NIP-46 / secret-key agnostic),
  `RelayPool` (connect, query with timeout, publish; queries report each
  relay's outcome, never an empty result for a relay that failed or timed
  out), `RelayCatalog` (user / default / archival sets). Anything a client
  already has can be wrapped in a few lines.
- Test vectors as importable JSON, plus a `conformance()` helper that runs
  them: a client's adapters are conformant when `conformance()` passes
  against their `RelayPool` mock and the shared vectors.

### UI contract

A conformant client screen:

- Lives in settings; scans only on user action.
- Renders all candidates with their timestamps, item counts (or
  partially-counted markers), and found-on relays, newest first by
  default, with the recommended candidate highlighted so a long history
  can't bury it. Runs of small edits and clobber episodes MAY be folded
  into expandable groups, with episodes marked, and past empty versions
  hidden until requested (see invariants 2 and 3).
- Offers a way to page further back when a relay filled a page.
- Shows each relay's outcome, at least on request, and says plainly when
  current could not be confirmed or the scan reached no relay; neither is
  shown as "nothing found".
- Shows the computed delta before any publish click, with the
  direction-of-harm warning for kinds that affect other people.
- Renders `meaningful-empty` kinds with the intent question and never
  pre-selects an option.
- Uses the signer for exactly one event per click.
- Offers restore only to accounts that can sign: a view-only account can
  scan its history but not restore it.

Decrypting private items can mean a signer prompt per version, so
implementations SHOULD decrypt up front only the versions shown on their
own rows, and grouped versions when the user expands their group or
reviews one.

Remote signers (NIP-46) receive every request NIP-44 encrypted, which caps
a request at 65,535 bytes: a follow list of roughly 850 accounts, or a mute
list with hundreds of private items, can't be decrypted or signed through
one. On a remote signer, implementations SHOULD mark versions too large to
restore in the list itself, not only at the publish step, and SHOULD
decrypt grouped versions only when the user reviews one, since every
decryption is a signer request.

Minimum viable integration is a list, a delta line (a field list for kind
0), and one button per candidate. Clients MAY go further (relay liveness
checks for kind 10002) per the registry's per-kind notes.

### Suggested rollout for client maintainers

1. Ship the recovery screen in a fork or feature branch; dogfood on your
   own account.
2. Extract and PR the screen upstream with the package dependency.
3. Publish observed clobber incidents (anonymized) to the spec's issue
   tracker; incident reports drive registry changes.

## Conformance

A client or library is **Lazarus-compatible** if and only if:

1. It honors the four invariants and the delta rule.
2. It passes the shared test vectors for every Tier 1 kind it supports:
   - scan fixtures: dedupe, paging, and relays that return foreign or
     forged events;
   - ranking fixtures: tombstone, meaningful-empty, partially-counted,
     private-item-estimate, gradual curation, and clobber-episode cases;
   - delta fixtures, including private items;
   - relay-outcome fixtures: a relay that fails or times out is never
     reported as empty, and a scan no write relay answered recommends
     nothing;
   - pre-sign re-read fixtures: a newer version, an older copy, and no
     write relay answering.
3. It reports its relay configuration and each relay's outcome alongside
   recovery results, so a "nothing found" answer can be judged against the
   relays that actually answered.

Test vectors live with the reference implementation and are versioned with
this spec. Ranking rules change only with a spec version bump; silent
drift between implementations is the failure mode this section exists to
prevent.

## Reference flow

The reference implementation (extracted from Mutable, where the core
survived React, Vue, and Svelte ports, and hardened in production in
the Jumble client, whose port contributed the private-items estimate
tier and clobber detection) performs: scan the relay set with per-relay
timeouts, record each relay's outcome, page back on request, dedupe by
event id, rank by the kind profile, render all candidates with deltas,
recommend per the profile, confirm current on the user's write relays,
publish on explicit click through the user's signer, republish widely.

## Non-goals

- Not a daemon. No scheduled scans, no watching, no background healing.
- Not a relay and not a search service.
- Not a backup format or backup product (see NIP-78; the two are
  complementary: backups are the garage, Lazarus is the seatbelt).
- Not a deletion tool. NIP-09 interactions are out of scope, including
  any attempt to "undo" a deletion request.
- Not a write-proxy. Recovery is signed by the user's own signer, always.
- Not a clobber-prevention mechanism. Lazarus does not police client
  writes (see invariant 1's rationale).

## Future work

- Addressable-kind recovery (requires per-`d`-tag ranking semantics).
- An archival-relay discovery convention (how implementations learn about
  archival sets without hardcoding).
- Accepting client-local version caches (a client that retains historical
  versions, as Jumble has proposed, can feed them to a scan as an
  additional source, clearly labeled as local) as a registered
  `HistorySource`.
- A machine-readable agent surface (MCP or similar). Stretch goal; the
  invariants, especially "never automatic," apply doubly to agents.

## Changelog

- 0.6.0-draft: relay failure semantics, untrusted relays and complete
  deltas, after review of a third implementation found that an
  unreachable relay read as an empty one, which let the re-read before
  signing pass with no relay answering. Every relay request now ends as
  answered, failed or timed out, and only an answered relay counts as
  having nothing; each relay's outcome is reported, and events that
  arrived before a failure stay candidates. The relay list is the newest
  kind 10002 found; a missing one is disclosed before defaults stand in,
  and one that could not be fetched is never substituted. Current is
  confirmed only when a write relay answered, and nothing is recommended
  until it is. A scan no relay answered is a failed scan, not an empty
  result. Relays are untrusted: an event counts only with a valid
  signature, the scanned author and the requested kind. The re-read must
  reach a write relay or the restore stops, with an explicit override
  after a failed retry; only a newer version counts as a change, it
  becomes current for the next review, and the recovery is dated after
  it. Publish success means a write relay accepted the event. Profile
  deltas cover every field and tag that would change, not a fixed set,
  and deltas include private items whenever they can be decrypted. The
  package contract gains `checkCurrent`, `buildRecoveryEvent` takes the
  current version it dates after, and relay queries report outcomes.
  Scans should bound and release their connections.
- 0.5.0-draft: restore safety, after review of a second implementation.
  Re-read the current version before signing and ask again if it changed,
  date the recovered event after the version it replaces, restore only as
  the list's author, judge publish success on the user's write relays with
  the other answering relays as best effort, and update the client's own
  copy after publishing. Deltas compare items by type and value, and
  profiles show the fields that change. The archival set in the README is
  trimmed to relays observed holding history.
- 0.4.0-draft: recommendations rewritten as clobber detection after
  implementation review. A bigger older version is no reason to restore,
  since lists shrink through curation, so the `count` profile now
  recommends only after a sudden drop the current version hasn't
  recovered from, with drops within a day grouped into one episode, and
  not once the list has been edited on for a week since.
  Candidates list newest first, with an optional size order. The scan
  covers the user's read relays too and pages back on request, since
  archival relays keep hundreds of versions. The publish minimum is the
  user's write relays plus every relay that answered the scan. The UI
  contract lets runs of small edits and clobber episodes fold into
  expandable groups, decrypts
  only what is shown, and adds remote-signer guidance: mark versions too
  large for NIP-46 up front, and decrypt grouped versions only on review.
  Past empty versions may be hidden until requested, and are not offered
  for restore.
- 0.3.0-draft: private-items contract rewritten as three certainties
  (exact / estimated / flagged) after implementation review: private-only
  lists are common, public counts alone read a full list and an emptied
  one as the same zero, and encrypted payload sizing (NIP-44 padded
  lengths, NIP-04 block bounds) distinguishes them without decryption.
  Ranking compares total item counts as ranges.
- 0.2.0-draft: client integration section (package contract, UI contract,
  maintainer rollout). Kind 10044 (NIP-4e, draft) added to the registry
  after a live incident in which a client emptied a user's encryption key
  list and broke their inbound DMs; `meaningful-empty` introduced,
  invariant 3 exception added, intent confirmation required.
- 0.1.0-draft: initial draft. Tier 1 registry, four invariants, delta rule,
  per-kind ranking profiles, private-items contract, conformance model.
