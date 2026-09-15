````markdown
# termin-bot

Watches `termine.duesseldorf.de` for **Umschreibung ausländischer Führerschein (sonstige Staaten)** and auto-books the first eligible slot.

- **Repo:** `kishanbala/termin-bot` (private)
- **Runtime:** GitHub Actions cron — no server, no local process
- **Status:** LIVE (autobooking enabled)

---

## 1. How it works

The city's booking wizard is plain HTTP, no JS/captcha required:

| Step | Request | Purpose |
|---|---|---|
| 1–2 | `GET /select2?md=3` | open session |
| 3 | `GET /location?mdt=402&select_cnc=1&cnc-5871=1` | select Anliegen (5871 = sonstige Staaten) |
| 4 | `POST /location` `loc=394` | select Höherweg 101 |
| 4→ | `/suggest` | either "keine freien Termine" or one form per open slot |
| 5 | `POST /suggest` (slot's signed `checksum`) | claim slot → `/personaldata` |
| 6 | `POST /personaldata` (name, email, DOB, consent) | reserve |

The script replays this chain every poll.

**Cadence:** cron fires every 5 min; each firing then polls internally every 45s for 4 min (`LOOP_SECONDS=240`, `POLL_INTERVAL=45`). Effective resolution ≈45s, not 5 min. GitHub does not guarantee cron timing under load — expect occasional delay.

**On finding a slot:**
1. Filter against `rules` (below)
2. Filter against calendar conflicts (below)
3. Book the earliest eligible slot
4. Email the result to `MAIL_TO`
5. Write `state.json` with `booked: true` → all future runs no-op until manually cleared

---

## 2. Configuration (embedded in `watch.yml`)

| Rule | Value | Meaning |
|---|---|---|
| `min_days_ahead` / `max_days_ahead` | 2 / 365 | booking window |
| `weekdays` | `null` | any day the city offers (site only opens weekdays anyway) |
| `earliest` / `latest` | 07:00 / 17:00 | time-of-day window |
| `duration_minutes` | 15 | city's allotted slot length |
| `buffer_minutes` | 45 | travel buffer either side of a calendar event |
| `blackout` | **2026-10-28 → 2026-12-03** | hard no-book window, overrides everything |
| `calendar_conflicts` | `prefer_free` | see §4 |

To change any of the above: edit the `CFG = {...}` block inside `watch.yml` (config is inlined in the workflow, not a separate file) and commit.

---

## 3. Secrets (Settings → Secrets and variables → Actions)

| Secret | Value | Notes |
|---|---|---|
| `BOOK_VORNAME` | Balakrishnan | |
| `BOOK_NACHNAME` | Ramakrishnan | |
| `BOOK_EMAIL` | ramakrishnanb050@gmail.com | sent to the city |
| `BOOK_DOB` | 01.11.1995 | dd.mm.yyyy |
| `BOOK_SEX` | m | m / w / u |
| `BOOK_PHONE` | *(unset)* | optional |
| `SMTP_USER` | ramakrishnanb050@gmail.com | Gmail sender |
| `SMTP_PASS` | Gmail **app password** | regenerate via Google Account → Security → App passwords if ever exposed |
| `MAIL_TO` | ramakrishnanb050@gmail.com | notification recipient |
| `ICS_URL` | iCloud "Busy" calendar public feed (`https://`, not `webcal://`) | read-only, unauthenticated link — treat as sensitive |

---

## 4. Calendar conflict handling

- Source: a manually maintained iCloud calendar named **Busy**, published from iPhone (Calendar app → Busy → Public Calendar → Share Link).
- **Not connected to Outlook.** Outlook publishing was unavailable on the corporate tenant. Any commitment that should block a booking must be **manually duplicated onto Busy**.
- `calendar_conflicts: prefer_free` — if a conflict-free slot exists, it's booked; if every open slot clashes with Busy, the bot books the clashing one anyway and flags it prominently in the confirmation email, rather than skipping the appointment entirely.
- The blackout window (§2) is the only conflict source that is never overridden.

---

## 5. Operations

**Run manually:**
Actions → `termin-watch` → **Run workflow** → toggle `dry_run` → Run.
- `dry_run` ✓: walks the full flow, stops before the final booking POST.
- `dry_run` ☐: live — will book a real appointment if one is found.

**Check status:** Actions tab → latest run → `watch` job → `Watch and book` step log.

**Stop it:** Actions → `termin-watch` → **···** → Disable workflow.

**Resume after a booking:** a successful booking sets `state.json`'s `booked` key, which halts all future runs. To watch again (e.g. appointment cancelled), delete that key from `state.json` in the repo.

**Editing the workflow:** the Python is embedded inline in `watch.yml` (single-file, for phone-only editing — no local dev environment used). Full text lives in this Claude conversation's history if a from-scratch rebuild is ever needed.

---

## 6. Known limitations

1. **Calendar detection is unverified end-to-end.** The `ICS_URL` feed itself was confirmed reachable and valid (HTTP 200, parses as ICS), but no real event has been added to Busy and confirmed to correctly block a slot. Test: add a throwaway event to Busy → run with `dry_run` ✓ → check the diagnostic step reports it → delete the event.
2. **Outlook calendar is not checked at all.** Only events manually copied to Busy count.
3. **GitHub disables a repo's cron after 60 days of no repo activity.** The state-commit step keeps it awake on runs that produce a change; a long silent stretch is worth a manual check-in.
4. **No captcha handling.** If the city adds one, the bot detects it, refuses to book, and emails instead of failing silently — but the appointment must then be claimed manually.
5. Confirmation email delivery (`SMTP_*`) has not yet been triggered by a real booking or alert — only exercised by the calendar diagnostic step, not the mail-sending path itself.

---

## 7. Security notes

- The GitHub personal access token used during setup has been revoked.
- The Gmail app password was regenerated after being visible in chat during setup.
- `ICS_URL` is an unauthenticated link; anyone holding it can read the Busy calendar. Revoke via iPhone Calendar → Busy → turn off Public Calendar (breaks the bot's calendar check until republished).
````
