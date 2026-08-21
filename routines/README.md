# Morning routine: Outlook calendar fix

The morning brief reported "no meetings detected" every day while the Outlook
calendar was full. This documents why, what was fixed live, and the one step
that still needs a human.

## Root cause

Three independent faults stacked up. Any one of them alone produces an empty
meeting list.

**1. The routine never read a calendar at all.** The v5 prompt's brief section
gathered `get_todays_meetings`, `get_pipeline_summary` and `detect_stale_items`,
then called `generate_morning_brief`. `get_todays_meetings` reads CloseDeals'
own meeting store, not a live calendar. Nothing in the routine ever called
`outlook_calendar_search`, and nothing ever called `sync_calendar_event`, so
nothing was ever written to that store. The brief read an empty table and
reported it accurately. This is the primary fault.

**2. `sync_calendar_event` defaults to Google.** Its `calendar_provider`
argument is documented as *"Defaults to google_calendar for legacy tasks."*
Any sync that omits the argument files the meeting under a provider this
account does not have. This is the Google wiring in the system: not a
connector anyone attached, just a default nobody overrode.

**3. The CloseDeals setup never reached the calendar step.** `get_setup_guide`
reports "Step 7 of 12 - Email Mining", and the calendar step comes after email
mining. So `connector_expectations.calendar` is `[]`, with inventory status
`unverified`. `generate_morning_brief` treats an unregistered provider
inventory as unverified coverage, which is why a brief can never state a
verified zero even once the events are flowing.

Worth stating plainly: no Google Calendar was ever connected to this account.
Some invites arrive *from* Google Calendar as forwarded mail (the Street and
TechCXO invites are examples), but they live on the Outlook calendar like
everything else.

## Fixed live on 2026-08-21

- Queried the Outlook calendar for the half-open window today 00:00 to the day
  after tomorrow 00:00, America/New_York. 10 events found, pagination
  exhausted.
- Full `read_resource` on all 9 active events for authoritative attendees, the
  opaque calendar id, and Teams `meetingTranscriptUrl`.
- Synced all 9 with `calendar_provider: microsoft_365` explicitly. Attendees
  matched existing contacts; one stub created for a second address of a person
  already on file (`nixon.james@gopitcrew.com`, same person as the
  `aluruintel.ai` record, worth a human merge).
- Pre-synced Mon 8/24 and Tue 8/25 external meetings as a bridge, so briefs
  stay useful until the prompt below is pasted.
- Re-ran `generate_morning_brief` with a complete source ledger.

Result: `summary` went from "no meetings" to **"8 known meetings today"**, and
`calendar_verification.complete_sources` is 1.

## What still needs a human

**Paste `morning-routine-v6.txt` into the morning routine's prompt.** The
routine was created through the API, so an agent cannot edit it:
`update_trigger` returns *"this routine was created via http_api, not by an
agent."* Until the prompt is replaced, each scheduled run repeats the v5
behaviour and reports an empty calendar again, because the live fix above
seeded data but did not change what the routine does each morning.

Routine: "Morning routine (Umergence Enterprise, v5)", cron `30 10 * * *`.
Replace the whole prompt and rename to v6.

## What v6 changes

Parts 1 and 2 are unchanged except for capturing `email_query_as_of`. The old
Part 3 becomes Part 4. The new Part 3 reads the calendar:

- Window computed with the offset actually in effect (-04:00 EDT / -05:00 EST).
  Outlook returns wall-clock plus a named zone; the pair must be converted
  together, never read as UTC.
- `afterDateTime` filters on start and `beforeDateTime` on end, so bounds are
  widened a day and trimmed after reading, or a meeting straddling midnight
  disappears.
- Pagination follows `nextOffset` to exhaustion.
- Full read per event before syncing.
- `calendar_provider: "microsoft_365"` on every sync, called out as never
  optional, with `source_account_id` so provenance keys carry the mailbox.
- A `calendar_sources` ledger, and `complete` claimed only when enumeration,
  pagination and full reads all ran clean.
- `items_supplied` counts only what is passed. Cancelled events are dropped
  server-side, so counting them logs a
  `source_item_attribution_mismatch` (this was observed and corrected).
- A sanity check: events returned but `get_todays_meetings` empty means the
  sync did not land, and the likely cause is `calendar_provider`.
- Zero meetings is reportable only against `query_status: complete`. Anything
  else is stated as unverified rather than rendered as "no meetings detected".

## Two things left open

**Finish CloseDeals setup steps 8-12.** Until the calendar step registers the
provider inventory, every brief carries
`expected_source_inventory_unverified` and cannot assert a verified empty day.
No exposed tool writes `connector_expectations` directly, and it did not
persist from a complete ledger, so this runs through the connector's own setup
flow.

**The auto-follow-up routine needs no calendar change.** It is Otter-triggered
and reads transcripts, not calendars. It does benefit indirectly: meetings now
exist in CloseDeals for it to attach to, and the full reads expose
`meetingTranscriptUrl` on Teams meetings, which is a second transcript source
alongside Otter if the Otter race ever needs a fallback.
