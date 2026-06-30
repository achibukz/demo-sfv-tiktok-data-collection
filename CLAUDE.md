# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Maintenance

Keep this file up to date. When stack choices are made, the data flow changes, or scope shifts, update the relevant section before finishing the task.

## Status

**Pre-implementation.** No application code exists yet. This file captures the scope, constraints, and intended role of the site so a future session can pick up the build without re-deriving context.

## Project Overview

A secured website that receives **donated TikTok analytics CSVs** from creators participating in a research study. It is the **submission step** of a creator-side data acquisition flow:

```
Consent → Extension Export → SUBMISSION (this site) → Video Download → Anonymization → Verification
```

The two file types the site accepts (produced by the companion Chrome extension):

- `tiktok_videos_{handle}_{YYYY-MM-DD}.csv` — per-video engagement metrics
- `tiktok_followers_{handle}_{YYYY-MM-DD}.csv` — 365-day follower history

The site replaces a previously-used Google Form. Reason for the change (adviser direction, 2026-06-27 sprint):

- **Accessibility** — direct upload, no Google account requirement.
- **Data privacy** — hosting on the **ALTDSI domain** is a recognizable, trusted endpoint for participants.
- **Automation** — submissions can flow straight into the downstream pipeline instead of needing manual export from Forms.

## Hosting

Target deployment is **ALTDSI infrastructure** (DLSU). Stack and exact host/runtime are not yet decided. Stack selection should account for what ALTDSI supports out of the box.

## Scope Constraints (do not relitigate)

These bound what the site is allowed to do:

- **Receive only — do not collect.** The site accepts CSVs that creators choose to upload. It does not log into TikTok, does not call TikTok APIs, does not scrape, and never holds creator credentials.
- **CSV uploads only.** No MP4 / video-content uploads. Video files are retrieved separately by researchers from the public TikTok platform using video IDs from the CSV.
- **Consent gate.** A submission flow must include an explicit informed-consent step before the upload form is shown.
- **Withdrawal path.** Participants must have a documented way to request deletion of their submission. The site needs an identifier (e.g. submission ID or contact email) that supports this without exposing other participants' data.
- **Server-side validation.** Reject files that don't match the expected filename pattern (`tiktok_videos_*.csv` / `tiktok_followers_*.csv`) or whose headers don't match the expected schema. Untrusted CSV input must be parsed defensively (no formula injection on re-export, size caps, MIME checks).

## Sensitive Data

Uploaded CSVs are research data. Treat them as sensitive:

- Never commit any uploaded file, sample participant data, or production secrets.
- `.env`, `.env.local`, and any keys/credentials are gitignored — keep it that way.
- Storage decisions (where uploads land, retention window, access control) need to align with the study's ethics approval before the site goes live.

## Conventions

- No emojis in code, comments, or UI strings.
- No comments unless the WHY is non-obvious. Don't narrate WHAT; well-named identifiers do that.
- Don't add features, error handling, or abstractions beyond what the task requires.
- Don't add backwards-compat shims or unused `_vars` for removed code — delete it.
- Never commit `Co-Authored-By: Claude` trailers.
- Date strings in any UI / log output are ISO `YYYY-MM-DD`.

## Open Decisions

To be settled before / during implementation:

- Web stack (framework, language, runtime).
- Storage backend for uploaded CSVs (object storage vs filesystem vs DB blob).
- Authentication model for participants (link-based token vs account vs anonymous with submission ID).
- Admin / researcher view: separate app, or scoped route on the same app.
- Exact ALTDSI deployment model (container, VM, reverse-proxy setup).
