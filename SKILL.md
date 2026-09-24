---
name: sip-01
description: "Build, validate, publish, and consume SIP-01 Search Index Protocol events (Nostr kind 39697, Web Index Observations) — the decentralized web-search index by NostrDanish (github.com/NostrDanish/SIP-01). Use whenever the task involves SIP-01, kind 39697, widx: d-tags, URL normalization per SIP-01 §7, x content hashes, decentralized search index events, the UNCAGED index relay / Crawlstr / indexstr / 0xSearchstr / 0xPresearchstr ecosystem, NIP-50 search operators for web documents (site:, domain:, lang:, filetype:), crawler heartbeats (kind 16919), or building a crawler/search-engine/index-relay on this protocol. Triggers on 'SIP-01', 'SIP01', 'widx:', 'kind 39697', 'web index observation', 'UNCAGED', 'Crawlstr', 'indexstr', 'Searchstr'."
---

# SIP-01 — Search Index Protocol (kind 39697)

One addressable event per `(indexer pubkey, normalized URL)`: *"Indexer `pubkey` observed this web document at this time, and here is its lightweight metadata."* Crawlers publish, relays store/validate/index, engines group-and-rank. No single owner.

## The event (memorize)

```json
{
  "kind": 39697,
  "content": "{\"title\":\"Example Page\",\"description\":\"...\"}",
  "tags": [
    ["d", "widx:<sha256(normalized_url)[0:32]>"],
    ["u", "<canonical normalized URL>"],
    ["x", "<sha256(title + \"\\n\" + description)>"],
    ["v", "1"],
    ["alt", "Web index observation: Example Page"]
  ]
}
```

- **Required** (exactly one each): `d`, `u`, `v` ("1"), `alt`; content JSON has `title` (1–300 trimmed).
- **Optional**: `description` ≤1000, `image` (https only), `t` topics ×≤8 (`^[a-z0-9][a-z0-9-]{0,99}$`), `l` (ISO 639-1, bare two-letter), `published` (unix seconds — NOT `published_at`), `source` (indexer software id, informational).
- **Extension tags** (spec §9.2): `type`, `platform`, `category`, `network`, `country` (uppercase alpha-2), `mime`. Experiments use `x-` prefix. Unknown tags MUST be ignored, unknown `v` MAY be rejected.
- Three identities: `d` = URL identity (same across all indexers → group-by-`d`, count distinct pubkeys = independent-observation count), `u` = canonical URL, `x` = content-agreement hash.

## Task routing

| Task | Do this |
|---|---|
| Normalize a URL, compute `d`/`x`, build an event, validate an event, run test vectors | Run `scripts/sip01.py` (zero-dependency; `selftest` runs spec §13 vectors). Byte-compatible with the TS reference — verified against Node's WHATWG URL |
| Sign/publish the built event, generate an indexer key | Use the `nostr` skill's `nostr_key.py` (sign) and `nostr_query.py` (publish/query) |
| Exact field rules, normalization steps, deviations, NIP dependency table | Read [references/spec.md](references/spec.md) — the canonical spec (v1.2), verbatim |
| Publisher/consumer/relay integration walkthroughs, checklists | Read [references/implementation-guide.md](references/implementation-guide.md) — verbatim upstream guide |
| Ecosystem repos, NIP-50 operator semantics, NIP-11 `uncaged_index` block, kind 16919 heartbeats | Read [references/ecosystem.md](references/ecosystem.md) |

## Non-negotiable rules

1. **Byte-compatibility is rule zero.** A one-character drift in URL normalization breaks dedup against every other indexer. Never hand-roll normalization — use `scripts/sip01.py` (or the upstream `sip01-utils.ts`), and always run `selftest` (spec §13 vectors) after touching anything.
2. **`d` MUST equal `widx:sha256(normalized u)[0:32]`** — readers and validating relays recompute it; mismatches are dropped. Same for `x` vs. the *published* (truncated) title/description.
3. **Indexer keys are separate, local, pseudonymous** — never a user's personal key, never uploaded. Key rotation = a new indexer; old observations stay valid.
4. **The protocol carries observations, never search queries.** No user identity, no query strings. Auto-indexing publishes document metadata only.
5. **Ranking, moderation, and trust are engine-local** — never published into the event. Application signals (stakes, votes, badges) go in *separate events referencing the `d` tag*.
6. **Verify before indexing** (§18): exactly-one checks, `d`↔`u`, `v === "1"`, `x`↔content; drop failures. Group by `d`; distinct pubkeys = the core signal; `x` disagreement = page changed or indexers disagree.
7. **`site:` over `domain:`** when the relay's nature is unknown — `domain:` collides with NIP-50's own NIP-05-author extension; SIP-01-aware relays give it URL-host semantics.
8. Baseline queries work on any stock relay (single-letter `#d`/`#t`/`#u`/`#x`/`#v`/`#l` filters). NIP-50 operators and the `uncaged_index` NIP-11 block are optional acceleration — check `supported_nips` first. Multi-letter tags are NOT relay-filterable on stock relays.

## Quick flows

Publish a crawler observation:
```bash
python3 scripts/sip01.py build "$URL" "$TITLE" --description "$DESC" --topic nostr --lang en --source mycrawler/1 \
  | python3 /path/to/nostr/scripts/nostr_key.py sign "$NSEC" /dev/stdin   # or jq pipe
# then nostr_query.py publish wss://<relay> '<signed>'  (2+ relays, incl. one SIP-01-aware)
```

Consume:
```bash
python3 /path/to/nostr/scripts/nostr_query.py query wss://<relay> '{"kinds":[39697],"#t":["privacy"],"limit":100}'
# group by d tag, count distinct pubkey values per d
```

Validate a live event: `python3 scripts/sip01.py validate '<event-json>'` (exit 0/1).
