# SIP-01 ecosystem, querying, and relay profile

Canonical site: https://github.com/NostrDanish/SIP-01 (spec `/spec`, tag registry `/registry`, query reference `/query`, live explorer + validator + d-tag calculator `/explorer`, stats `/dashboard`, audit `/audit`).

## Ecosystem repos

| Repo | Role | Key files |
|---|---|---|
| **SIP-01** | Canonical spec + docs site | `public/spec/SIP-01.md`, `docs/IMPLEMENTATION-GUIDE.md`, `src/lib/sip01-utils.ts` (byte-compatible browser port — lift this for TS work) |
| **UNCAGED-Index-Relay** | Validating index relay (NIP-50 operators, NIP-11 capabilities, NIP-77 federation) | `src/web-document.ts` (validator to match), `docs/SIP-01.md` (relay profile) |
| **Crwalstr** | Browser web crawler, pure SIP-01 publisher, per-device indexer keys | `src/crawler/` (mirrors `webIndex.ts` byte-compatibly) |
| **indexstr** | Distributed indexing network: sharded crawling + node heartbeats | `src/crawler/heartbeat.ts` (kind 16919 schema) |
| **UNCAGED-ENGINE** | Reference search-engine / forkable template | `src/lib/webIndex.ts`, `src/lib/indexerIdentity.ts` |
| **0xSearchstr** | Search engine, auto-indexes results via autosigner | reference client impl |
| **0xPresearchstr** | Search engine: clearnet + tor/i2p crawling, keyword stakes | |

## Crawler heartbeats — kind 16919 (replaceable)

Crawlstr and indexstr nodes publish liveness/health heartbeats as kind 16919;
the `source` tag distinguishes the software family (`crawlstr/...` vs
`indexstr/...`). Schema lives in indexstr `src/crawler/heartbeat.ts`. Consumers
(dashboards) read it; engines don't need it. SIP-01 itself publishes neither
heartbeats nor observations from the docs site.

## Querying

Baseline (every stock NIP-01 relay):
```json
["REQ", "s", { "kinds": [39697], "#t": ["privacy"], "limit": 100 }]
["REQ", "s", { "kinds": [39697], "#d": ["widx:..."] }]
["REQ", "s", { "kinds": [39697], "authors": ["<indexer hex>"], "since": 1786000000 }]
```
Relay-filterable = single-letter tags only: `#d`, `#t`, `#u`, `#x`, `#v`, `#l`
(+ `authors`, `since`/`until` on observation time, `limit`). Multi-letter tags
(`type`, `platform`, `country`, `mime`, `source`, `published`, `alt`) are NOT
indexed by stock relays — use NIP-50 operators on SIP-01-aware relays, or
filter client-side.

NIP-50 acceleration (SIP-01-aware relays):
```json
["REQ", "s", { "kinds": [39697],
  "search": "bitcoin privacy site:github.com lang:en after:2026-01-01", "limit": 50 }]
```

### SIP-01 NIP-50 operator table

`site:` `domain:` `url:` `inurl:` `title:` `topic:` `type:` `platform:`
`category:` `network:` `country:` `mime:` `filetype:` `source:` `lang:`
`before:` `after:` `distinct:domain` — each also in negated `-op:` form.

Precision rules:
- Operator support is per-relay. Check `supported_nips` contains `50` and look
  for the `uncaged_index` block (below) before relying on semantics.
- `domain:` collides with NIP-50's own registered extension (author NIP-05
  domain). SIP-01-aware relays = exact URL-host match; generic relays may read
  it as author-domain. **Prefer `site:` when unknown** (no upstream collision;
  it matches host + dotted parents via `url_domain_hierarchy`).
- NIP-50 sanctions `key:value` extensions and directs relays to ignore unknown
  ones (SHOULD) — these queries are safe to send anywhere.
- NIP-45 `["COUNT", ...]` gives cheap totals where supported; distinct-pubkey
  counting per `#d` remains client-side unless a relay advertises an extension.

### NIP-11 advertisement (`uncaged_index` block)

SIP-01-aware relays declare scope in their relay information document:

```json
{
  "supported_nips": [1, 11, 45, 50, 77],
  "uncaged_index": {
    "sip01": true, "nip50": true, "nip77": true,
    "document_kinds": [39697],
    "scope": "global",
    "domains": ["*"],
    "languages": ["en", "de"],
    "document_types": ["page", "repository"],
    "filters": ["site", "domain", "url", "inurl", "title", "topic", "type",
                "platform", "category", "network", "country", "mime",
                "filetype", "source", "lang", "before", "after",
                "distinct:domain"]
  }
}
```

NIP-11 requires clients to ignore unknown fields, so the block is always safe.

## Building a SIP-01 index relay (profile summary)

On top of a stock relay:

1. **Ingestion validation** — exactly the rule table in `scripts/sip01.py validate`
   (mirrors UNCAGED `src/web-document.ts`); reject with `OK false invalid: <reason>`.
   Unknown tags ignored; unknown `v` rejected.
2. **Structured index fields** — `url`, `url_host`, `url_domain_hierarchy`
   (host + dotted parents → powers `site:`), `file_ext`, `title`, `description`,
   `language`, `content_hash`, `published_at`, `observed_at` (= `created_at`),
   `source`; lowercased `doc_type`/`platform`/`category`/`network`/`content_type`,
   uppercased `country`.
3. **NIP-50 operators** mapped onto those fields (table above); list `50` in
   `supported_nips`.
4. **NIP-11 `uncaged_index` block** (above).
5. **Federation** — NIP-77 negentropy on `{"kinds":[39697]}` with peers:
   `["NEG-OPEN", "sync", {"kinds": [39697]}, <hex>]`. No master relay.

## Legacy data

Kind 30078 with `d:"0xsearchstr:cache:*"` (historical 0xSearchstr query caches)
is frozen legacy: consumers MAY merge by normalized URL; new indexing MUST use
kind 39697 (spec §17). No flag day.
