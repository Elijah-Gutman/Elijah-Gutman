# CloseDeals: five issues found 2026-08-21

From Elijah Gutman, Umergence / Hartford AI Partners. Enterprise instance,
account elijah.gutman@umergence.com, Microsoft 365 connector, no Google
Workspace on the account.

Found while diagnosing a morning brief that reported "no meetings detected"
every day against a full Outlook calendar. Firm-confidential specifics
(deal values, client names, pipeline totals) are omitted; everything below is
reproducible without them.

---

## 1. `sync_calendar_event` silently defaults to the wrong provider

**Severity: high.** `calendar_provider` is documented as *"Defaults to
google_calendar for legacy tasks."* On an account with no Google Calendar,
omitting the argument files the meeting under a provider that does not exist
there. Nothing errors. The write succeeds, the meeting is simply invisible to
`get_todays_meetings`, and `generate_morning_brief` then reports zero
meetings accurately from an empty store.

This is the single highest-leverage fix on the list. A silent default that
misfiles data is worse than a required argument.

**Suggested fix:** drop the default and require `calendar_provider`
explicitly, or default it from the account's registered connector rather than
to a constant. If the default has to stay for back-compat, return a warning
in the response when it was applied implicitly.

## 2. `connector_expectations` cannot be reached without email mining

**Severity: high.** `generate_morning_brief` will not treat an empty calendar
as authoritative unless the expected provider inventory is registered.
The inventory lives in `connector_expectations`, which is populated by the
guided setup's calendar step. That step sits behind email mining, and the
guide enforces "complete each step before moving to the next."

The result: an account that declines a six-month email backfill can never
register its calendar inventory, so every brief it generates carries
`expected_source_inventory_unverified` and can never state a verified empty
day. We submitted a fully clean ledger (`resource_enumeration_complete`,
`pagination_complete`, `full_reads_complete` all true,
`query_status: complete`). It was accepted, `complete_sources` went to 1, and
the inventory still did not persist.

No exposed tool writes it either. `update_company_hub` sections are
company_overview, products, icp, sales_process, tone_prefs, scheduled_tasks.

**Suggested fix:** either persist the inventory from a complete source ledger,
or expose the calendar/email/transcript/document inventory as a directly
writable section, independent of step order.

## 3. Vector search is down: the embeddings key is rejected, and the raw error reaches the client

**Severity: high, with a security note.** Every embeddings-backed call is
failing against OpenAI with an invalid key. `team_search` surfaces it directly:

```
OpenAI embeddings request failed: 401 {"error": {"message": "Incorrect API
key provided: sk-proj-****...der). You can find your API key at
https://platform.openai.com/account/api-keys.", "code": "invalid_api_key"}}
```

The key is CloseDeals-side, not something a tenant supplies, so this is
very likely affecting every account rather than just ours. Flagging it as the
most time-sensitive item here for that reason.

Three separate problems fall out of it.

**Semantic search is silently degraded product-wide.** Plain `search` still
returns results, but every row comes back with `embedding: null` alongside a
populated `search_vector` and an `_rrfScore`, so the full-text half of the
fusion is carrying the query alone. Callers get results and no signal that
recall dropped. A `degraded: true` flag or a warning in the response would let
a consumer decide whether to trust a negative result.

**`team_search` hard-fails where `search` degrades.** Same backend, two
different behaviours. This matters because scheduled routines lean on search
for duplicate-deal and duplicate-contact guards before writing records. A
graceful degrade keeps those guards working with reduced recall; a hard
failure risks duplicates being created. `team_search` should fall back the way
`search` already does.

**The upstream error body is passed through to the client.** It carries the
key prefix and trailing characters. Even mostly masked, provider credentials
should not reach a tenant's client. Worth catching provider errors and
returning a generic message.

## 4. Two completion meters disagree, and the optimistic one is the visible one

**Severity: medium.** `get_company_hub` reports `completeness_pct: 100` with
`incomplete_sections: []`. `get_setup_guide` reports `total_steps: 12` and
`progress: "Step 7 of 12"`. Both are correct: the percentage covers the six
profile sections, which are genuinely done, and the guide counts twelve steps
including ingestion and automation.

We read the 100% and believed onboarding was finished. It was half finished,
and the half that was missing is the half that matters operationally
(issue 2 above is downstream of it).

**Suggested fix:** have `completeness_pct` cover all twelve steps, or rename
it to something scoped like `profile_completeness_pct` and surface step
progress alongside it in the same response.

## 5. `items_supplied` must exclude cancelled events, undocumented

**Severity: low.** `generate_morning_brief` drops cancelled events
server-side, so counting them in `items_supplied` produces
`source_item_attribution_mismatch:reported=10:observed=9`. Correct behaviour,
but the field description says only "normalized items_supplied matching the
payload," which reads as the count of what you passed.

**Suggested fix:** state in the description that cancelled events are
excluded from `items_supplied`.

---

## What we changed on our side

Nothing in the above blocks us. The morning routine now queries
`outlook_calendar_search`, does a full `read_resource` per event, syncs each
with `calendar_provider: "microsoft_365"` explicitly, and passes a
`calendar_sources` ledger. Verified working: the brief went from reporting no
meetings to listing the day correctly.

Issues 1 and 3 are the two we would most like fixed upstream. Issue 1 because
the next person to hit it will lose the same week we did, and issue 3 because
it is live right now.
