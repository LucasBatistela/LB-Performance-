# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page marketing + booking site for a personal trainer (Lucas Batistela), plus a full student/professor
portal (scheduling, training plans, health questionnaire, chat, notifications). There is no build system, no
package manager, and no test suite — the entire application is one file: [index.html](index.html) (~8,100 lines:
inline `<style>`, inline `<script>`, no external JS/CSS files).

## Commands

There is nothing to install or build. To work on the site, edit `index.html` directly and open it in a browser
(or serve the directory with any static file server — the app makes no assumptions about a specific host).
There is no linter, formatter, or test runner configured in this repo.

**Serving it locally is worth the trouble**, because with no test suite the only way to check a UI change is to
load the page: a syntax error anywhere in the single inline `<script>` kills the whole app, and the browser
console is what tells you. This machine has no `node` and no real `python` (the `python.exe` on PATH is the
Microsoft Store stub, which fails), but Git Bash ships `perl`, which can serve a directory from core modules
alone. Write a small `IO::Socket::INET` loop to a scratch file and run it against the repo root — don't commit
the server into the project. Loading `index.html` over `http://` rather than `file://` also matters: Supabase
auth and the CDN scripts behave differently on a `file://` origin.

## Architecture

### Single-file SPA
All "pages" are `<div id="page-<name>">` elements inside `index.html`, toggled by `showPage(id)`
(`index.html:3456`) which adds/removes an `active` class — there is no client-side framework or router library.
Key mechanics:
- **Route protection**: `showPage()` holds a `protectedRoutes` map (`id -> required role | null`) and redirects
  to `login` or `home` when the current session doesn't satisfy it. Roles are `student` and `professor`.
- **Hash sync**: `syncHashForPage()` / the `popstate` listener keep `location.hash` in sync with the current
  page so browser back/forward, deep links (from notifications, email, WhatsApp), and refresh all work.
- **Per-page init hooks**: `showPage()` calls a page-specific render function when navigating in (e.g. `agendar`
  → `renderCalendar()`, `anamnese` → `iniciarAnamnese()`, `duvidas` → `renderDuvidasPage()`). When adding a new
  page, wire its init call here.
- **Boot sequence**: `init()` at the bottom of the file (`index.html:8059`) initializes Supabase, handles
  password-recovery/OAuth-callback URLs, restores an existing session, then opens whatever page the URL hash
  (or session state) indicates.

### Backend: Supabase, called directly from the browser
There is no server/API layer. `index.html` loads `@supabase/supabase-js` from a CDN and talks to Supabase
directly using the `publishable` (anon) key (`index.html:5033-5045`, project ref `upmealfqrodhpqdpceec`,
region `sa-east-1`). This means:
- **Auth** (email/password, Google OAuth, password recovery) goes through `sb.auth`.
- **Data access** goes through PostgREST via `sb.from('<table>')` calls scattered across the file. Tables in use:
  `profiles`, `students`, `professors`, `bookings`, `fixed_booking_skips`, `schedule_exceptions`, `exercises`,
  `training_divisions`, `training_exercises`, `load_logs`, `anamnese_responses`, `anamnese_drafts`,
  `notifications`, `professor_settings`, `chat_messages`, `pending_registrations`, `motivational_phrases`,
  `password_reset_codes`.
- **Privileged operations** (deleting a student, resetting a student's password, etc.) go through Postgres
  `SECURITY DEFINER` RPC functions rather than raw table writes, so the real authorization logic lives in the
  Supabase project's SQL, not in this file. Anything touching auth/permissions should be checked against the
  live function definitions in Supabase (via the Supabase MCP connector), not assumed from the client code.
- Access control for direct table reads/writes is enforced by Postgres **Row Level Security** policies on the
  Supabase project, not by anything in this repo.

### Email: EmailJS, mostly from the browser
Booking confirmations/cancellations are sent client-side via `@emailjs/browser`, using
service/template/public-key IDs stored in `professor_settings` and in the seed `site-state` JSON block at the
top of the file (`index.html:25-41`).

**Password-reset codes are the exception** — they are sent by the `request-password-reset` Edge Function,
which calls the EmailJS REST API server-side with a private key held in the `EMAILJS_PRIVATE_KEY` secret. The
code must never be returned to the client; that is the whole point of the function existing. See below.

### The one server-side piece: `request-password-reset`
The only Edge Function in the project. It runs with `verify_jwt = false` (whoever is resetting a password is
logged out by definition) and protects itself instead by: rate-limiting to 3 codes per student per 15 minutes,
answering identically whether or not the CPF exists (so it can't be used to enumerate students), and never
putting the code in the response. It writes the code through the `store_reset_code` RPC, which bcrypt-hashes
it. `confirm_student_password_reset` returns a status string (`ok` / `invalid` / `locked` / `weak`) rather than
raising, because raising would roll back the failed-attempt counter that caps brute force at 5 tries per code.

`service_role` has no blanket DML grant in this project; it was granted only `SELECT` on the four tables that
function reads. If a future Edge Function needs another table, grant it explicitly rather than restoring the
Supabase default.

### Theming
CSS custom properties on `:root` define a light palette; `@media (prefers-color-scheme: dark)` and an explicit
`:root[data-theme="dark"]` override it. `toggleTheme()` persists the user's choice to `localStorage`
(`lb_theme`), and a small inline `<script>` in `<head>` applies it before first paint to avoid a flash.

## Rendering user text

Anything a student can type reaches the professor's screen: their own `full_name`, their anamnese answers, and
the notification titles built from their name. Every one of those goes through `escHtml()` before it touches
`innerHTML`, and through `jsAttr()` inside an `onclick="fn('…')"` attribute. `esc()` is a thin wrapper over
`escHtml()` that also maps null/empty to `''` — it used to be a stringifier that escaped nothing, which is how
a stored XSS reached the professor's session. Keep new render sites consistent with this.

## Setup the repo can't do for you

Two settings live outside the code and are not in version control:

- `EMAILJS_PRIVATE_KEY` — Edge Function secret, from EmailJS → Account → API Keys. Without it,
  `request-password-reset` returns `email_nao_configurado` and no reset email goes out.
- **Leaked password protection** — Supabase dashboard → Authentication → Policies. Off by default.

## Claude Code setup in this repo

`.claude/skills/` has 19 skills installed from `obra/superpowers`, `vercel-labs/skills`,
`supabase/agent-skills`, and `pbakaus/impeccable` (plus 4 companion agents from `impeccable` in
`.claude/agents/`). `.claude/settings.json` wires `impeccable`'s `PostToolUse`/`Stop` hooks so it runs
automatically after UI edits. `superpowers`' own session-start automation (auto-injecting its
`using-superpowers` skill at session start) was **not** installed — its hook relies on `${CLAUDE_PLUGIN_ROOT}`,
which is only set when a skill is installed as a real Claude Code plugin, not when its `skills/` folder is
copied in directly as it was here.
