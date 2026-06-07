---
name: list-meetings
description: List recent Fathom meetings — titles, dates, and participants — without pulling full transcripts. Use when the user says "list my meetings", "what calls did I have", "show recent meetings", "last N Fathom meetings", or wants a quick roster of recordings rather than one meeting's detail. Companion to pull-meeting (use that one to drill into a single call).
---

# List Meetings

Give a quick, scannable roster of the most recent Fathom recordings: title, date, and who was in the room. This is the lightweight counterpart to `pull-meeting` — no transcripts, no summaries, just the list so the user can pick what to drill into.

## Configuration

Reads the Fathom API key from the `FATHOM_API_KEY` **environment variable** (no local `.env`). If `$FATHOM_API_KEY` is empty, stop and tell the user to set it.

## API basics

- **Base:** `https://api.fathom.ai/external/v1`
- **Auth:** header `X-Api-Key: $FATHOM_API_KEY`
- **List:** `GET /meetings?include_transcript=true&limit=<N>` (transcript needed only to derive real participants; we don't print it)

## Workflow

1. Fetch the recent meetings (default N=10; honor the user's number if they gave one):

   ```bash
   mkdir -p tmp
   curl -s -H "X-Api-Key: $FATHOM_API_KEY" \
     "https://api.fathom.ai/external/v1/meetings?include_transcript=true&limit=10" \
     > tmp/fathom-list.json
   ```

2. Print a numbered roster — one line per meeting, most recent first:

   ```bash
   python3 -c "
   import json
   d = json.load(open('tmp/fathom-list.json'))
   items = d.get('items') or []
   if not items:
       print('No meetings returned by the API.'); raise SystemExit(0)
   for i, m in enumerate(items, 1):
       title = m.get('title') or m.get('meeting_title') or '(untitled)'
       start = (m.get('recording_start_time') or '')[:16].replace('T', ' ')
       speakers = sorted({t['speaker']['display_name'] for t in (m.get('transcript') or [])})
       who = ', '.join(speakers) if speakers else '(no speakers parsed)'
       print(f'{i}. {start}  |  {title}  |  {who}')
   "
   ```

3. Present the roster. If the user then wants detail on one of them, hand off to the `pull-meeting` skill for that specific meeting.
