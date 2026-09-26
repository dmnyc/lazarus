# Lazarus

**Recovery of user data from relay history on Nostr.**

Nostr clients overwrite replaceable events without reading them first. When a
client touches your follow list, mute list, profile, or pins, whatever it
publishes replaces what was there. Until relays expire it, your history is a
full undo log. Lazarus is a protocol for using it: scan the relay set, rank
the versions found, and restore what a rogue client destroyed — only through
an explicit click, only with the user's own signer.

## The four invariants

1. **Never automatic.** Nothing is restored without the user clicking.
2. **Show everything.** Every version found is displayed: counts, timestamps,
   origin relay, gaps.
3. **Tombstones are never recommended.** A deletion event is a decision, not
   damage. Lazarus surfaces them; it never ranks one as "the good state."
4. **Your keys, your publish.** Restoration uses the client's own signer on
   an explicit click. No key custody, ever.

## Scope

Tier 1 (recoverable now): kind 3 (follows), kind 10000 (mutes). Tier 2:
kind 0 (profile), kind 10003 (bookmarks). Tier 3: 10002 (relay list), 10050
(DMs key relays), 10006 (blocked relays), each with liveness warnings. Kind
10044 carries a defined meaningful-empty: an empty list means opt-out, so it
is flagged and never ranked. Everything else is out of scope until amended.

Private items in NIP-51 lists are handled under a three-certainties
contract: **exact** count after decrypting with the user's key, **estimated**
min/max band from encrypted payload sizing (NIP-44 padded length, NIP-04
block bounds) when it can't be decrypted, **flagged** when neither applies.
Ranking compares total ranges, so a private-only list that was emptied never
reads as a zero.

Recommendations follow clobbers, not size. Lists shrink through normal
curation, so a version is only recommended after a sudden drop (a fifth of
the list at once) that the current version hasn't recovered from, and then
it's the fullest version from before the damage. A clobber the list has
since been edited on for a week is settled, and nothing is recommended.

## Where the history lives

Most relays keep only the latest version of a replaceable event, so a
user's own relays usually hold just the current list: for one account, its
first five outbox relays had a single version of the follow list. The
history survives on relays that keep old versions. Observed in September
2026:

| Relay | What it keeps |
|---|---|
| `wss://relay.ditto.pub` | Every version: hundreds of that follow list, going back 14 months |
| `wss://hist.nostr.land` | Recent history: 20 versions of that follow list from the past few weeks |
| `wss://nos.lol`, `wss://nostr.mom`, `wss://purplepag.es`, `wss://nostr.bitcoiner.social` | Often a version or two that the user's own relays already replaced, including a clobbered, empty follow list |

Scanning every relay in the user's relay list (read and write: that
account's history relay was listed for reading only), the client's default
relays, and these found 51 versions of the follow list on the first page,
and 439 after paging back.

The reference implementations scan these six on top of the user's relays,
kept to relays observed holding history since every relay is another
connection: `relay.ditto.pub`, `hist.nostr.land`, `nos.lol`, `nostr.mom`,
`purplepag.es`, `nostr.bitcoiner.social`. Which relays keep history changes
over time, so treat the list as a starting point: relay sets are
configuration, not protocol (see [SPEC.md](SPEC.md)).

## Implementations

| Client | Spec | Status | Links |
|---|---|---|---|
| Jumble (fork) | 0.6.0 | Shipped — reference implementation | [Live deployment](https://jumble.dmnyc.net) · [dmnyc/jumble-spark](https://github.com/dmnyc/jumble-spark), branch `feat/lazarus-data-recovery-v2` |
| zap.cooking | 0.6.0 | Merged | [zapcooking/frontend#753](https://github.com/zapcooking/frontend/pull/753) |
| Jank (fork) | 0.5.0 | Merged in v26.16.0; the 0.6.0 update is in review | [DocNR/jank#22](https://github.com/DocNR/jank/pull/22) · [DocNR/jank#24](https://github.com/DocNR/jank/pull/24) |

Origins: the core survived React, Vue, and Svelte ports in
[Mutable](https://github.com/dmnyc/mutable); the list-recovery concept first
shipped in [Plebs vs Zombies](https://github.com/dmnyc/plebs-vs-zombies).

## Status

Spec **0.6.0-draft**. Expect changes before 1.0. See [SPEC.md](SPEC.md) for
the full document: per-kind registry, delta rule, meaningful-empty,
recommendation rules, client integration contracts, conformance vectors.
