# ALTDSI TikTok Submission Site — Design

**Date:** 2026-06-30
**Status:** Approved for implementation planning
**Owner:** Aki
**Related:** [Project CLAUDE.md](../../../CLAUDE.md), Dataset Collection wiki at `~/Documents/Obsidian/schoolMem/wiki/AY2526-T3/THSST1-Thesis-in-Software-Technology-1/topics/dataset-collection.md`

## Purpose

Build the submission step of the SFV-thesis data acquisition flow as a secured website hosted on ALTDSI infrastructure. The site replaces the previously-used Google Form with a self-hosted page that accepts donated TikTok analytics CSVs from invited Filipino micro-creators.

```
Consent → Extension Export → SUBMISSION (this site) → Video Download → Anonymization → Verification
```

The site accepts the two CSV files produced by the companion Chrome extension. It does not log into TikTok, scrape, or accept video files. Researchers (admins) issue access codes by email; participants enter the code on the site, accept the inline consent text, fill a short form, and upload the two CSVs.

## Scope

### In scope

- Participant-facing flow: code entry → consent → form → confirmation
- Researcher (admin) flow: invite issuance (single + bulk), submissions list, submission detail with download, withdraw, purge
- Server-side CSV validation (filename + headers; trust the extension for content)
- Audit log of all participant and admin events
- Single-VM deploy on ALTDSI (Linux VM with root)

### Out of scope

- URERB ethics review gate (the site itself is not URERB-blocked)
- AES-256 at-rest encryption (deferred; relying on VM-level protections)
- MP4 / video file uploads (researchers fetch videos from TikTok separately)
- Roster categorical fields (`niche`, `follower_bracket`, `audience_geo`) — filled by researchers in `roster.csv`, not collected at submission
- Real-time pipeline integration (researchers pull zips manually from /admin)
- Background workers / cron (purge is an explicit admin action)
- Multi-tenancy or multi-study generalization

### Deferred to ALTDSI sysadmin

| Decision | Default assumed for spec |
|---|---|
| Admin identity provider | OIDC (Auth.js, provider parameterized via env) |
| Outbound SMTP | nodemailer with configurable SMTP transport |
| Site domain | Placeholder `submit.altdsi.dlsu.edu.ph` (TBD) |

## Constraints (from CLAUDE.md)

- Receive only — do not collect. Do not call TikTok APIs or hold creator credentials.
- CSV uploads only. No MP4 / video-content uploads.
- Consent gate. Explicit informed-consent step before the upload form.
- Withdrawal path. Documented mechanism for participants to request deletion.
- Server-side validation. Reject files that don't match expected filename pattern or headers.
- No emojis in code, comments, or UI strings.
- ISO `YYYY-MM-DD` for dates in UI and logs.
- No `Co-Authored-By: Claude` trailers on commits.

## Stack

**Approach A (chosen): Next.js App Router, full-stack.**

| Concern | Choice |
|---|---|
| Runtime | Node.js (LTS) |
| Framework | Next.js 15+ App Router with Server Actions |
| Language | TypeScript (strict) |
| ORM / migrations | Drizzle |
| Database | Postgres 16 (local on the VM, Unix socket) |
| Schema validation | Zod |
| CSV parser | `csv-parse` (strict mode) |
| Admin auth | Auth.js (Generic OIDC provider, parameterized) |
| Participant session | `iron-session` (signed cookie, 1 h idle, sliding) |
| Email | nodemailer with configurable SMTP transport |
| Rate-limit | `rate-limiter-flexible` (in-memory) |
| Markdown rendering (consent) | `marked` + `DOMPurify` |
| Tests | vitest + `@testcontainers/postgresql`; Playwright for one smoke test |
| Reverse proxy / TLS | nginx (Let's Encrypt or institutional cert) |
| Process management | systemd |

Rejected alternatives: Hono on Node + custom UI (too much wheel-reinvention for forms, SSR, OIDC); SvelteKit (smaller ecosystem for OIDC; team unfamiliar).

## Deployment shape

Single Linux VM at ALTDSI. Three systemd-managed processes:

```
nginx (TLS termination, static, reverse-proxy)
  └─ Next.js (Node, App Router) — port 3000, systemd unit
       └─ Postgres 16 (local, Unix socket) — systemd unit
```

Backups: nightly `pg_dump` to a researcher-controlled machine via SCP. Out of band; not in the app.

Releases: manual (`git pull`, `pnpm install`, `pnpm build`, `systemctl restart sfv-site`). No CI deploy automation in MVP.

## Project layout

```
sfv-tiktok-data-collection/
├── app/
│   ├── (participant)/
│   │   ├── page.tsx              # landing + code entry
│   │   ├── consent/page.tsx      # version-stamped ICF text + accept
│   │   ├── submit/page.tsx       # form: handle, email, 2 CSV slots
│   │   └── confirmed/[id]/page.tsx
│   ├── (admin)/
│   │   └── admin/
│   │       ├── page.tsx          # submissions list
│   │       ├── [id]/page.tsx     # submission detail
│   │       ├── invites/page.tsx  # invite mgmt: list + issue + bulk
│   │       └── login/page.tsx
│   ├── api/
│   │   └── auth/[...nextauth]/route.ts
│   └── middleware.ts             # /admin/* auth gate
├── lib/
│   ├── db/                       # drizzle schema + client + migrations
│   ├── csv/
│   │   ├── schemas.ts            # header arrays + EXPECTED_SCHEMA_VERSION
│   │   ├── validate.ts           # validate(buffer, expectedKind)
│   │   ├── defensive.ts          # formula-injection sanitizer (export-time)
│   │   └── kinds.ts              # inferKindFromFilename
│   ├── code/
│   │   ├── generate.ts
│   │   ├── verify.ts
│   │   └── rateLimit.ts
│   ├── auth/
│   │   ├── participantSession.ts
│   │   └── adminGuard.ts
│   ├── mail/
│   │   ├── transport.ts
│   │   └── templates/
│   └── audit/log.ts
├── consent/
│   └── v1.0.md                   # version-stamped ICF text
├── tests/
│   ├── fixtures/                 # real extension outputs (verbatim)
│   ├── unit/
│   ├── integration/
│   └── smoke/
├── docs/superpowers/specs/
└── deploy/
    ├── nginx.conf.example
    ├── systemd/sfv-site.service.example
    └── README.md
```

## Architecture boundaries

- `lib/csv/` knows nothing about HTTP. Pure functions; unit-testable in isolation.
- `lib/code/` is the only source for access-code generation, formatting, and verification.
- `lib/auth/` separates the two auth realms: participant (cookie-bound code) vs admin (OIDC session).
- `/admin/*` is route-gated by middleware. Defense-in-depth: every admin server action also calls `requireAdmin()`.
- `lib/mail/` exposes a transport interface. The transport is configurable; if SMTP env vars are unset, it writes to a dev log file instead of sending.

## Data model

Five Postgres tables. Soft-delete by default; hard-delete is an explicit admin action.

```
invites
─────────────────────────────────────────────
  id                uuid PK
  access_code       text UNIQUE              -- "R7K9-M2NP-X8WL"
  creator_handle    text                     -- researcher's note, not enforced
  email             text                     -- where the code was sent
  created_at        timestamptz
  created_by        text                     -- admin email from OIDC
  revoked_at        timestamptz NULL
  last_used_at      timestamptz NULL


submissions
─────────────────────────────────────────────
  id                       uuid PK
  invite_id                uuid FK → invites(id)
  handle_at_submission     text              -- what the participant typed
  contact_email            text
  consent_version          text              -- "v1.0"
  consent_hash             text              -- sha256 of consent markdown at accept time
  consent_signature_name   text              -- full name typed at consent step
  consent_accepted_at      timestamptz
  submitted_at             timestamptz
  superseded_at            timestamptz NULL
  withdrawn_at             timestamptz NULL
  withdrawn_by             text NULL
  withdrawn_reason         text NULL


csv_files
─────────────────────────────────────────────
  id                uuid PK
  submission_id     uuid FK → submissions(id)
  kind              text   -- 'videos' | 'followers'
  original_filename text
  size_bytes        integer
  sha256            text
  row_count         integer
  bytes             bytea NULL  -- NULLed on supersede; metadata retained
  UNIQUE (submission_id, kind)


admin_allowlist
─────────────────────────────────────────────
  email             text PK
  role              text     -- 'admin' (single role for MVP)
  added_at          timestamptz
  added_by          text


audit_log
─────────────────────────────────────────────
  id                uuid PK
  at                timestamptz
  actor_type        text     -- 'participant' | 'admin' | 'system'
  actor_id          text     -- invite_id (uuid) or admin email
  event             text
  target_id         text NULL
  ip                inet NULL
  user_agent        text NULL
  metadata          jsonb
  INDEX (at), INDEX (target_id), INDEX (event)
```

### Audit events

`invite.issued`, `invite.revoked`, `invite.email_failed`, `code.entered`, `code.rejected`, `code.locked_out`, `consent.accepted`, `submission.created`, `csv.validated`, `csv.rejected`, `submission.superseded`, `admin.viewed`, `admin.downloaded`, `admin.denied`, `submission.withdrawn`, `submission.purged`, `system.error`.

### Storage policy for superseded submissions

On a new submission for the same invite, the prior row gets `superseded_at = now()` and its `csv_files.bytes` is set to NULL. Metadata (size, sha256, row_count) stays for audit.

### Hard-delete

`withdrawn_at` flips the row out of the default admin list. Admin can purge (hard-delete row + cascaded `csv_files`) from the detail page. No background cron — explicit action only.

### Allowlist seeding

Admin allowlist is seeded from env `ADMIN_ALLOWLIST` (comma-separated emails) on first boot via a migration. After that, edits are SQL-direct until a `/admin/users` page exists.

### Indexes

- `invites(access_code)` unique
- Partial index on `submissions` where `superseded_at IS NULL AND withdrawn_at IS NULL`
- `audit_log(at)`, `audit_log(target_id)`, `audit_log(event)`

## Participant flow

```
GET  /                          landing + code entry
POST /  (action: enter-code)    verify code, set session, redirect
GET  /consent                   render markdown ICF
POST /consent (action)          record acceptance, redirect
GET  /submit                    form
POST /submit (action)           validate, insert, redirect
GET  /confirmed/:id             receipt page
```

### Code entry

- Trim, uppercase, strip dashes; look up `invites.access_code`
- Rejected → bump rate-limit counters (per IP, per code), log `code.rejected`, redisplay landing with generic "code not recognized"
- Rate-limited → 429 page with retry time
- Revoked → same generic message; log `code.rejected` with metadata flag
- Accepted → create `iron-session` cookie payload `{ invite_id, nonce, started_at }`, log `code.entered`, redirect to `/consent`

### Consent

The participant must complete the consent step before the submission form is reachable. The page renders `consent/v1.0.md` (or the current version) as the body of the ICF. Below the body:

- **Full name** (text input, required, trimmed, non-empty) — typed signature
- **Checkbox**, required: "I have read the above and consent to participating in this research."
- **Continue** button (disabled until both name is filled and checkbox is ticked)

`consent/v1.0.md` body content is **TBD** — researchers will supply the detailed informed-consent text. Until then, the file is a single placeholder line; the bytes still get hashed so the version pin is real.

On accept, the server action:
- Reads consent markdown from disk, computes sha256
- Stashes `{ consent_version, consent_hash, consent_signature_name, consent_accepted_at }` in the session cookie (no draft DB row)
- Logs `consent.accepted` with `{ consent_version, consent_hash }` in metadata (the typed name is **not** put in `audit_log.metadata` — it lives on the submission row only)
- Redirects to `/submit`

If the participant reaches `/submit` without a valid consent payload in the cookie, they get redirected back to `/consent`.

### Submit form

Fields:
- TikTok handle (text, pre-filled from `invites.creator_handle`, editable, required)
- Contact email (text, pre-filled from `invites.email`, editable, required, valid-email format)
- Video analytics CSV (file input, required, `.csv` accept)
- Follower history CSV (file input, required, `.csv` accept)

Server action on submit, single transaction:
1. Re-check session cookie carries `{ invite_id, consent_version, consent_hash, consent_signature_name, consent_accepted_at }`; redirect to `/consent` if any are missing
2. Validate both CSVs (see Validation section)
3. Any failure → re-render form with per-field errors, log `csv.rejected`, save nothing
4. Find prior current submission for this `invite_id`; mark `superseded_at`, NULL its `csv_files.bytes`
5. Insert new `submissions` row, including `consent_signature_name` from the cookie
6. Insert two `csv_files` rows (with bytes, sha256, row_count, size)
7. Update `invites.last_used_at`
8. Log `submission.created` + two `csv.validated`
9. Redirect to `/confirmed/:id`

### Confirmation page

Shows submission UUID (full + short prefix), handle, ISO date, signed name, two filenames + row counts. Guidance: "You can resubmit any time with the same access code; the latest version will replace this one." + "To withdraw, email <research-contact>."

Session stays alive until cookie idles out — participant can hit `/submit` again from `/confirmed/:id` without re-entering the code.

### Rate-limit policy

- Per-IP: 5 code attempts / 15 min, then 30-min lockout
- Per-code: 5 bad attempts → code temporarily locked for 30 min
- Per-invite: max 20 submissions per day (defensive cap)

## Researcher (admin) flow

```
GET  /admin/login                    sign-in button
GET  /admin/auth/callback            OIDC callback (Auth.js)
GET  /admin                          submissions list
GET  /admin/[submissionId]           submission detail
POST /admin/[submissionId]/withdraw  soft-delete
POST /admin/[submissionId]/purge     hard-delete (must be withdrawn first)
GET  /admin/[submissionId]/download  per-submission zip
GET  /admin/download-all             bulk zip of all current submissions
GET  /admin/invites                  list + issue + bulk import
POST /admin/invites/issue            create one invite + send email
POST /admin/invites/import           parse CSV, batch-create, batch-email
POST /admin/invites/[id]/revoke      flip revoked_at
POST /admin/invites/[id]/reissue     revoke old + create new + resend
```

### Auth gate

`middleware.ts` on every `/admin/*` request:
1. No Auth.js session → redirect to `/admin/login`
2. Session present but email claim not in `admin_allowlist` → 403, log `admin.denied`
3. Otherwise → allow, attach `req.admin = { email, name }`

If OIDC env vars are missing, `/admin/login` returns a 503 page reading "Admin auth not configured — set OIDC env vars".

### Submissions list

Default filter: `superseded_at IS NULL AND withdrawn_at IS NULL`. Toggles to include superseded and withdrawn. Columns: short ID, handle, email, submitted_at (ISO), status badge, video row count, follower row count. Per-row "Download zip"; top-of-page "Download all current".

### Submission detail

Metadata + consent panel (`consent_version`, `consent_hash`, `consent_signature_name`, `consent_accepted_at`) + per-file panels (filename, size, sha256, row_count, "Download CSV") + audit timeline scoped to this `target_id`.

Actions:
- **Withdraw** — modal asks reason; sets `withdrawn_at`, `withdrawn_by`, `withdrawn_reason`, NULLs `csv_files.bytes`. Logs `submission.withdrawn`.
- **Purge** — only visible if already withdrawn. Hard-deletes row + cascaded `csv_files`. Logs `submission.purged` with metadata snapshot in `metadata` jsonb.

Superseded submissions show "Superseded by [short id]" link instead of action buttons.

### Per-submission zip

```
submission_<shortid>_<handle>/
  ├─ submission.json
  ├─ tiktok_videos_<handle>_<date>.csv
  ├─ tiktok_followers_<handle>_<date>.csv
  ├─ consent_<version>.txt
  └─ audit.json
```

`consent_<version>.txt` is the exact bytes of the ICF the participant accepted, followed by a signature block appended at export time:

```
--- SIGNATURE ---
Signed by: <consent_signature_name>
Accepted at: <consent_accepted_at, ISO 8601 UTC>
Document hash (sha256): <consent_hash>
```

`submission.json` includes `consent_signature_name`, `consent_accepted_at`, `consent_version`, and `consent_hash` as top-level fields.

### Bulk zip

Top-level zip with one folder per current submission (same layout) plus a top-level `roster.csv` with `pseudonymous_id, creator_handle, email, submission_id, submitted_at, consent_version`. Drops into `pipeline/build_dataset.py` after researchers add the TBD categorical columns.

### Invite issuance — single

Form: handle + email → server action generates a code (`lib/code/`), inserts row, sends email via `lib/mail/`, returns a confirmation panel that re-displays the code once. Logs `invite.issued`.

### Invite issuance — bulk

CSV upload `{handle, email}`. Server parses, validates each row, shows preview ("3 ready, 1 invalid email"). Send button creates rows + emails as a batch. Result page reports per-row outcome.

### Reveal code

Each row in `/admin/invites` has a "Reveal" action that re-displays the code via a server roundtrip — convenient when a participant lost the email.

### Email content

| Email | Subject | Body essentials |
|---|---|---|
| Invite | "Your SFV research data donation — access code" | Greeting with handle, the code (dash-formatted), site URL, two lines of context, help contact |
| Confirmation | "Your data donation was received" | Short submission ID, two filenames saved, "resubmit anytime with the same code", withdrawal contact |
| Reissue | "Your replacement access code" | New code; prior one is no longer valid |

`lib/mail` env: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM`. Unset → dev log file output. Production gated on these being set.

## CSV validation

Trust the extension. Validate identity, not content.

### Pipeline (per file)

1. **File-level**
   - Size ≤ 5 MB
   - Filename matches `^tiktok_(videos|followers)_[A-Za-z0-9_]+_\d{4}-\d{2}-\d{2}\.csv$`
   - Kind inferred from the `videos|followers` segment
2. **Encoding**
   - UTF-8 (allow BOM); reject non-UTF-8 bytes
3. **Parse**
   - `csv-parse` in strict mode; reject only on parser-level failure
4. **Header check (the only content check)**
   - First row must equal the expected header array for the kind, in order

No row schemas, no row caps, no value-token checks, no cell-length checks.

`row_count` is still computed during parse and stored in `csv_files.row_count` for /admin display.

### Schema pinning

```ts
export const EXPECTED_SCHEMA_VERSION = "v2026-06-30";

export const VIDEO_HEADERS = [
  "video_id","post_date","post_time","caption","duration_ms","comments","shares","ECR",
  "avg_watch_time_s","NAWP","watched_full_pct","traffic_foryou_pct","traffic_follow_pct",
  "traffic_profile_pct","traffic_search_pct","new_followers","data_quality",
] as const;

export const FOLLOWER_HEADERS = [
  "date","follower_count","daily_net","creator_handle","creator_uid","data_quality",
] as const;
```

When the extension bumps its CSV columns, bump `EXPECTED_SCHEMA_VERSION` and the header arrays together. This is a coupled change documented in `deploy/README.md`.

### Formula-injection sanitizer (export-time only)

In `lib/csv/defensive.ts`, applied to per-submission zips and bulk zips:

- Any cell whose first character is `=`, `+`, `-`, `@`, `\t`, `\r` gets prefixed with a single quote `'`
- Each zip includes an `EXPORT-NOTES.txt` explaining the transformation and how to recover the original

Bytes in the DB stay untouched. The sanitizer only runs on copies being handed out.

### Logging on validation

- Accepted: `csv.validated` with `{kind, row_count, sha256}`
- Rejected: `csv.rejected` with `{kind, reason_codes}` (codes, not full text — avoids logging file contents)

## Error handling

### Server-action contract

Every server action returns `{ ok: true, data } | { ok: false, errors }`. The UI maps error codes to user-facing text; server actions never throw raw exceptions across the boundary. A wrapper catches unhandled exceptions, logs them with stack + request ID + actor context, and returns a generic error to the UI.

### Participant-facing

| Where | Failure | Behavior |
|---|---|---|
| Code entry | Invalid / revoked code | Generic "code not recognized"; bump rate-limit counter |
| Code entry | Rate-limited | 429 page with retry time; no specifics on which limit tripped |
| Consent | Cookie missing/expired | Redirect to `/` with "your session expired" |
| Submit | Bad email format | Inline error under field; form state preserved |
| Submit | Missing CSV | Inline error under slot; other slot's filename label shown |
| Submit | Filename pattern mismatch | Per-file error: shows pattern + what was uploaded |
| Submit | Header mismatch | Per-file error: expected vs got + "re-export from latest extension" |
| Submit | File > 5 MB | Per-file error with size + cap |
| Anywhere | Server exception | Generic message + request ID for support |

### File-input UX wrinkle

File inputs cannot be repopulated across a server round-trip. If one CSV fails validation, the user re-picks both. Mitigations:
- Client-side regex check on file select (immediate "this filename doesn't match" before submit)
- Server's per-file error panel makes it explicit which file failed and why
- Both files required at submit; no partial save

Not adding a multi-stage upload flow for MVP. Audience is 30-50 cooperative participants.

### Admin-facing

| Where | Failure | Behavior |
|---|---|---|
| `/admin/login` | OIDC env vars missing | 503 page |
| OIDC callback | Email not in allowlist | 403 page with rejected email + contact line; log `admin.denied` |
| `/admin/*` | Auth.js session expired | Redirect to `/admin/login` |
| Issue invite | SMTP fails | Row created; flagged "email_pending" in list with "Resend"; log `invite.email_failed` |
| Bulk import | Some rows invalid | Preview page shows per-row status; user confirms only valid rows |
| Bulk import | SMTP fails mid-batch | Per-row success/fail on result page; failed rows resendable |
| Withdraw / purge | DB error | Action rolled back; admin sees toast; nothing logged as withdrawn |

### Email decoupled from submission success

If a participant submits and the confirmation email fails, the submission is still saved and the confirmation page still renders. The email failure shows as a warning badge on that submission in `/admin`, with a "Resend confirmation" button.

### Security defaults

| Concern | Handled by |
|---|---|
| CSRF on server actions | Next.js built-in |
| XSS in consent markdown | `marked` + `DOMPurify`; no raw HTML allowed |
| Authorization escalation | Middleware on `/admin/*` + defense-in-depth `requireAdmin()` |
| Header / cookie tampering | `iron-session` signs participant cookie; Auth.js signs admin session |
| Brute-force on codes | Per-IP + per-code rate-limit |
| Direct ID guessing on `/admin/[id]` | Admin session check; submission IDs are UUIDs |
| Formula injection on re-export | `lib/csv/defensive.ts` export sanitizer |

## Testing

Three layers, weighted toward unit tests for parts where bugs would silently corrupt research data.

### Unit (vitest)

| Module | What it tests |
|---|---|
| `lib/csv/validate` | golden inputs (real extension fixtures) → both validate; mutated copies (truncated header, renamed column, extra column, non-UTF-8 byte, oversize) → each rejection reason fires |
| `lib/csv/kinds` | filename pattern: accepts valid, rejects malformed |
| `lib/csv/defensive` | export sanitizer: `=SUM(…)` → `'=SUM(…)`; benign caption passthrough |
| `lib/code` | generation: no ambiguous chars, correct length, uniqueness; normalization for verify |
| `lib/code/rateLimit` | counters trip at thresholds; lockout window respected; decay |
| `lib/auth/participantSession` | cookie roundtrip; expired rejected; tampered rejected |
| Drizzle schema | migrations apply clean on empty DB and on a previously-migrated DB |

Fixtures (verbatim extension outputs in `tests/fixtures/`):
```
tiktok_videos_unknown_2026-06-30.csv
tiktok_followers_unknown_2026-06-30.csv
```
Plus generated mutants for negative cases.

### Integration (vitest + `@testcontainers/postgresql`)

Server actions against a real DB, no HTTP:
- `enterCode` invalid → counter bumps, no session
- `enterCode` valid → session set, log row written
- `acceptConsent` with empty name → rejected, session unchanged
- `acceptConsent` with name + checkbox → session carries `consent_signature_name`, log row written
- `submitForm` without consent in session → redirect to `/consent`, nothing inserted
- `submitForm` first submission → row + 2 csv_files inserted, `consent_signature_name` persisted, `invites.last_used_at` updated
- `submitForm` second submission → prior superseded, prior bytes NULLed, prior metadata kept
- `submitForm` validation failure → nothing inserted, both errors returned
- `issueInvite` → row inserted, mock mailer called once with the code
- `withdrawSubmission` → `withdrawn_at` set, csv bytes NULLed, audit row written
- `purgeSubmission` → only allowed if `withdrawn_at IS NOT NULL`

Postgres container per suite; migrations run before each suite; no shared fixtures across tests.

### Smoke (Playwright, one happy path)

Against a locally-running app + DB:
1. Seed an invite via SQL
2. Browser opens `/`, enters the code, accepts consent, uploads both fixture CSVs, sees confirmation
3. Browser opens `/admin` (admin session pre-set), sees the submission with the typed signature, downloads the per-submission zip
4. Assert the zip contains both CSVs + `submission.json` (with `consent_signature_name`) + `consent_*.txt` (with the appended SIGNATURE block) + `audit.json`

### Out of scope for testing

- Real SMTP delivery (mocked)
- Real OIDC handshake (admin session seeded directly via test helper)
- Visual regression

### CI

GitHub Actions: lint → typecheck → unit + integration → smoke. Postgres service in the workflow for integration; Playwright runs against the built Next.js dev server. No deploy automation in CI — releases are manual.

## Open dependencies

These must be resolved before the site goes live with real participants:

**ALTDSI sysadmin**
1. **OIDC identity provider** — Google Workspace, ALTDSI custom OIDC, or local accounts behind VPN. Provider URL + client ID/secret needed.
2. **Outbound SMTP** — DLSU/ALTDSI relay credentials, or approved third-party (Postmark/Resend) sender domain with SPF/DKIM.
3. **Domain + TLS** — final hostname for invite emails and admin OIDC callback URL.

**Research team**
4. **Informed-consent body text** — `consent/v1.0.md` body content is TBD. The site renders whatever bytes are in that file and hashes them per-submission, so the file can land any time before recruitment starts.

Spec uses sensible defaults for sysadmin items; implementation lands as env-configurable so the deploy step doesn't require a code change.

## Future work (not in MVP)

- `/admin/audit` log viewer UI (researchers can `psql` in the meantime)
- `/admin/users` allowlist management page (SQL-direct for now)
- Automated pipeline integration (researchers download zips manually for now)
- Background purge of long-withdrawn submissions (manual purge for now)
- Multi-language UI / Filipino translation
- Per-event email notifications to researchers on submission

## References

- [Project CLAUDE.md](../../../CLAUDE.md) — scope constraints, conventions
- Dataset Collection wiki: `~/Documents/Obsidian/schoolMem/wiki/AY2526-T3/THSST1-Thesis-in-Software-Technology-1/topics/dataset-collection.md` — participant profile, ethics framing, downstream pipeline
- TikTok Analytics Exporter wiki: `~/Documents/Obsidian/schoolMem/wiki/AY2526-T3/THSST1-Thesis-in-Software-Technology-1/topics/tiktok-analytics-exporter.md` — companion Chrome extension producing the CSVs
