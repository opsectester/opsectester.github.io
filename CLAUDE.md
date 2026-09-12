# Unpacked — blog authoring guide

This repo is **Unpacked**, a personal security-research blog built with Jekyll + the Chirpy theme and published via GitHub Pages at `https://opsectester.github.io`. When asked (via an `@claude` issue or comment) to write or edit a post, follow these rules exactly.

## Where posts live

- Every post is a Markdown file in `_posts/`, named `YYYY-MM-DD-short-slug.md` (lowercase, hyphenated slug).
- One post per pull request. Open a PR with the new file; don't commit straight to `main` unless explicitly told to.

## Front matter (required)

Each post starts with YAML front matter: a quoted `title`, a `date` in `YYYY-MM-DD HH:MM:SS +1000` (Australia/Brisbane) form, `categories` (1-2 broad buckets like [Malware Analysis, Android]), `tags` (specific, lowercase, hyphenated), a one-sentence `description`, and `toc: true`. Never date a post in the future.

## Voice — write like the author, not like an AI

- First person, plainspoken, dry, a little Aussie edge. Confident, curious about the hunt, happy to admit what didn't work.
- Banned constructions: the "it's not just X, it's Y" flip; rule-of-three lists for rhythm; em-dash pile-ups; filler like "crucial", "robust", "seamless", "leverage", "delve", "here's where it gets interesting", "in the world of". Cut them.
- Short paragraphs. Plain sentences. Technical precision over polish.

## Content ethos (non-negotiable)

- Full transparency: publish complete IOCs, un-redacted. Defang for click-safety only (hxxps, [.]) — never withhold. The point is that other researchers can pivot off them.
- Responsible disclosure: if a campaign is live, note that it's been reported (ReportCyber / hosting provider / registrar). Report alongside publishing, not instead of it.
- Honesty about limits: if the analysis didn't fully resolve (e.g. a C2 left encrypted), say so plainly and describe the wall. Don't fake a tidy ending.
- Everything is lab work: samples handled on an isolated bench, nothing run on real hardware. Say so.

## Identity / OPSEC

- The author is pseudonymous: **Opsec Tester**. Never include a real name, employer, client names, or the ajcit domain anywhere in a post, commit message, or PR.

## Preferred structure for an analysis post

1. Short version / TL;DR — what it is in a few lines.
2. Narrative walkthrough — how it was found, the method, each finding in order.
3. Indicators of Compromise — Markdown tables (delivery, sample, injected payload, sandbox refs).
4. MITRE ATT&CK mapping where relevant.
5. A closing lab-disclaimer line and an invitation to compare notes.

Keep code, hashes, and IOCs in fenced blocks or tables so they render cleanly with Chirpy's syntax highlighting.
