---
name: pull-meeting
description: Pull a meeting from the Fathom API — transcript, AI summary, action items, speakers, share URL — and present a digest. Use whenever the user says "pull the meeting", "pull yesterday's call", "grab the fathom meeting", "get the last meeting with X", or anything that implies fetching a recorded call. Portable version distributed via the Stackflow skills marketplace.
---

# Pull Meeting

<!-- UPDATE TEST MARKER: v0.1.1 — if you see this line, the content update propagated -->

Fetch the most recent meeting (or a list) from Fathom's REST API and surface the parts that matter: who spoke, what it was about, action items, share URL, and the transcript on demand.

## Configuration

This skill reads the Fathom API key from the `FATHOM_API_KEY` **environment variable** — it does not read any local `.env` file, so it works on any machine where that variable is set. If `$FATHOM_API_KEY` is empty, stop and tell the user to set it.

## API basics

- **Base:** `https://api.fathom.ai/external/v1`
- **Auth:** header `X-Api-Key: $FATHOM_API_KEY`
- **List meetings:** `GET /meetings?include_transcript=true&include_summary=true&limit=10`
- **Useful query params:** `since=<ISO date>`, `cursor=<next_cursor>`, `limit` (default returns recent-first)
- **Response shape:** `{ items: [...], next_cursor }`. Each item has `recording_id`, `title`, `meeting_title`, `url`, `share_url`, `recording_start_time`, `recording_end_time`, `calendar_invitees`, `recorded_by`, `transcript` (array of `{speaker: {display_name}, text, timestamp}`), `default_summary.markdown_formatted`, `action_items`.

`calendar_invitees` is often empty for impromptu Meet calls. To find who was actually in the room, read `speaker.display_name` off the transcript turns.

## Workflow

1. Fetch the latest meetings into a scratch file. `include_transcript=true` is mandatory.

   ```bash
   mkdir -p tmp
   curl -s -H "X-Api-Key: $FATHOM_API_KEY" \
     "https://api.fathom.ai/external/v1/meetings?include_transcript=true&include_summary=true&limit=10" \
     > tmp/fathom-latest.json
   ```

2. Pick the right meeting based on what the user asked for (see "Selection" below). Default is the most recent (first item).

3. Extract and present a digest with a short python snippet — never dump the raw JSON into the conversation, it's huge:

   ```bash
   python3 -c "
   import json
   d = json.load(open('tmp/fathom-latest.json'))
   items = d.get('items') or []
   if not items:
       print('No meetings returned by the API.'); raise SystemExit(0)
   m = items[0]  # or whichever index matches the ask
   print('title:', m.get('title') or m.get('meeting_title'))
   print('start:', m.get('recording_start_time'))
   print('end:', m.get('recording_end_time'))
   print('url:', m.get('share_url') or m.get('url'))
   speakers = sorted({t['speaker']['display_name'] for t in (m.get('transcript') or [])})
   print('speakers:', ', '.join(speakers))
   print()
   print('=== SUMMARY ===')
   print((m.get('default_summary') or {}).get('markdown_formatted') or '(no summary)')
   print()
   print('=== ACTION ITEMS ===')
   for a in (m.get('action_items') or []): print('-', a)
   "
   ```

4. Report back with: title, start time, participants, share URL, the condensed summary, and action items. Don't paste the full transcript unless asked.

## Selection — which meeting does the user mean?

- **"Last meeting" / "yesterday's call"** → first item (API returns recent-first). Verify `recording_start_time` matches the expected day.
- **"Meeting with <name>"** → scan each item's transcript speakers for that name (case-insensitive `display_name`). Don't trust `calendar_invitees` alone.
- **"The one about X"** → grep `default_summary.markdown_formatted` or `title` for the topic.
- **Older than ~1 week** → paginate with `cursor` or narrow with `since=<ISO date>`.

If more than one matches, show a short numbered list (title, date, participants) and ask which one.
