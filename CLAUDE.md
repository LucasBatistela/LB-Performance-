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

### The server-side pieces
Two Edge Functions, both `verify_jwt = false` because in both cases the person is logged out by definition:
`criar-conta-aluno` (see **Primeiro acesso do aluno**) and `request-password-reset`.

`request-password-reset` runs with `verify_jwt = false` (whoever is resetting a password is
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

## Monitoramento de carga interna (PSE / PSR)

The training-load module rests on published instruments, and the wording of each question is part of the
instrument — don't paraphrase them casually:

- **PSR** (recuperação), 0–10, answered on arrival — Laurent et al., 2011, *J Strength Cond Res*
  ([DOI](https://doi.org/10.1519/JSC.0b013e3181c69ec6)), validated for resistance training by Tolusso et al.
  ([DOI](https://doi.org/10.1123/ijspp.2021-0360)).
- **PSE** (esforço), CR-10, answered at the end — the session-RPE method, reviewed by Foster et al.
  ([DOI](https://doi.org/10.1123/ijspp.2020-0599)) and by Haddad et al. ([DOI](https://doi.org/10.3389/fnins.2017.00612)).
- **Monotonia e strain** — Foster, 1998 ([DOI](https://doi.org/10.1097/00005768-199807000-00023)).
- **ACWR is deliberately absent.** It is contested in the literature and was left out until it can be grounded
  as firmly as the rest. Don't add it casually.

`training_sessions` is the central table. It exists **independently of `bookings`** — a session the student
trains alone still counts, because monotonia and strain are computed over the whole week and would be wrong if
half the sessions were missing. `carga` (PSE × minutos) is a generated column: don't recompute it in the client.
`session_pain_points` holds one row per body region reported. A partial unique index enforces one open session
per student.

**Two rules the dashboards must keep:**

1. **Never compare students to each other.** The PSR validation is explicit that equal scores across people do
   not mean equal recovery. Every deviation is measured against that student's own baseline, which needs
   `EVO_MIN_BASE` (10) sessions before any alert is shown.
2. **The student view carries no jargon.** `page-evolucao` renders two ways off `currentProfile.role`:
   the professor gets monotonia, strain, z-score badges and raw per-session answers; the student gets
   frequency, load progression and plain-Portuguese sentences. Keep that split when adding to either.

## Módulos desligados

`MODULOS` (in the main script) is a product switch, not a removal: training-plan building, chat and load
progression are hidden from navigation while their code and data stay put. `aplicarModulos()` hides every
`[data-modulo]` entry point at boot and `showPage()` blocks the routes. Turning one back on is flipping the
constant. **`progressaoCarga` depends on `montagemTreino`** — without a built exercise there is no load to log,
so they travel together.

## Grupos e turmas

The professor runs 1:1 personal training *and* a functional studio in the same account. `student_groups` are
his own labels (not a fixed enum — he intends to add more), a student can be in several, and the group can be
picked at pre-registration (it rides on `pending_registrations.group_id` and materializes in
`recover_student_profile` / `link_google_student_signup`) or assigned later on the Turmas tab.

**`student_groups.modalidade` decides what the student sees on the agenda**, and it is the only switch that
does: `agendamento` gives the booking calendar, `confirmacao` gives the studio block with Vou / Não vou and
hides booking entirely. A student in both sees both, stacked. The identity columns (`local_nome`, `cor`,
`logo_url`) only render for `confirmacao`, and `salvarConfigGrupo()` nulls them when a group leaves that
modality, so a stale studio name can't resurface later. The Grade de horários tab only offers `confirmacao`
groups, because a `class_slot` on a booking group would be a class no student could ever see.

On the student's side a confirmation is **one class per day per group**: confirming a time flips any other
time that student had confirmed that day to `vai = false` (not delete — by picking 07:00 they have said they
won't be at 06:00, and the professor should read "avisou", not "não respondeu"). The DB does not enforce this;
`desmarcarOutrosDoDia()` does, because the rule is about how this studio runs, not about the data.

`class_confirmations` is **intention**, `training_sessions` is **reality** — that separation is the whole point.
Crossing them is what `renderTurmaHoje()` shows as *confirmou e não veio*, the only absence worth a message.
Don't collapse the two tables into one "attendance" table.

`class_slots` models the studio as it actually works: **no booking and no capacity** — the slot exists and
students show up. The student's check-in figures out which class it belongs to from the clock, with an option
to correct it. Attendance comes from that check-in; `marcar_presenca` lets the professor add whoever trained
and forgot to answer, tagged `origem = 'professor'` so it registers presence without inventing PSR/PSE answers.
The reverse is deliberately blocked — a student's check-in is their data and is not deleted from a roll call.

**`schedule_exceptions.aplica_a` says which of the two schedules a closure hits** — `agendamento`, `turmas`
or `ambos` (the default, and what the form offers). Both places that build `profExceptionsMap` drop rows whose
scope is `turmas`, which is why `getAvailableSlots()` never had to learn that scope exists; `renderAulasFixas()`
and `renderTurmaHoje()` read the other side. A `type='open'` row is a booking window with a start and end time,
so a check constraint pins it to `aplica_a='agendamento'` — there is no "opening" a fixed class that isn't in
the grid.

**Watch for RLS recursion here.** `student_groups` and `student_group_members` reference each other, and naive
policies deadlock with `infinite recursion detected in policy`. `e_membro_do_grupo()` is `SECURITY DEFINER` for
exactly that reason: it reads membership without triggering RLS. Any new policy spanning these two tables
should go through it rather than re-querying the other table directly.

## Primeiro acesso do aluno

The student's first login used to be: password **equals their own CPF**, which is also their login, and the
"change it now" dialog had a *Depois* button and closed on a backdrop click. On top of that,
`pending_registrations` — CPF, full name, phone, gender, e-mail — was readable by `anon` with `USING (true)`,
so the publishable key in the page source was enough to dump the list.

Now: the professor sets a shared first-access password in Configurações
(`professor_settings.senha_padrao_alunos`), and **the browser never learns it**.
`iniciar_primeiro_acesso(cpf, senha)` checks both server-side and returns `null` for a wrong password *and* for
an unknown CPF — same answer on purpose, so the screen isn't an "is this CPF registered?" oracle. Then
`concluir_primeiro_acesso()` creates profile + student + **group membership** + clears the pending row in one
transaction. The group link has to live there: only the professor can write `student_group_members`, so the
old client-side path silently dropped whatever group was chosen at pre-registration. The password-change
dialog no longer has any way out except logging out.

**The account is created server-side**, by the `criar-conta-aluno` Edge Function (`verify_jwt = false`, same
reason as the other one). The client never calls `sb.auth.signUp` any more. That exists so public sign-ups can
be **off** in the dashboard: while that endpoint answered, anyone who knew a registered CPF could create that
student's account with a password of their own and skip every check above. `iniciar_primeiro_acesso` also caps
wrong guesses at 5 per CPF per 15 minutes, because the first-access password is shared across the professor's
students and is therefore worth guessing.

## Escritas que ninguém espera

A Supabase query builder is a **thenable that resolves with `{ data, error }`** — a failed write does not
reject. So `.then(()=>{})` discards the error and a chained `.catch` never runs. Twelve fire-and-forget writes
in this file were written that way, which is how `update({ timezone })` against a column that did not exist ran
on every single login for months in silence. Any write you don't await ends in
`.then(avisarFalhaDeFundo, avisarFalhaDeFundo)`; anything you do await checks `error` before claiming success.

## Imagens que o usuário envia

`profiles.avatar_url` and `student_groups.logo_url` hold a **public Storage URL**, not image bytes and not a
`data:` URI. The avatar used to live only in `localStorage`, which meant it died with the browser profile and
nobody but its owner ever saw it; don't reintroduce that. Both flows share `comprimirImagem()` (canvas resize →
`toBlob`) and both upload to a bucket whose policy pins `(storage.foldername(name))[1]` to `auth.uid()`, so the
path's first folder is the owner — keep that shape for any new bucket. Avatars go out as JPEG; **logos go out as
PNG**, because a transparent logo flattened to JPEG turns into a black rectangle. The upload path is fixed per
owner, so both append `?v=<timestamp>` to bust the CDN, and every URL is filtered through `safeUrl()` on write
and on render.

**Never put `capture` on an image input.** `capture="environment"` makes the phone open the rear camera directly
and, on Android, removes the gallery option entirely — students could not pick a photo they already had.

## `[hidden]` is global, don't re-declare it

`[hidden] { display: none !important; }` sits in the reset. It is there because the `hidden` attribute loses to
any declared `display` — inline or from a class — and this file has a dozen one-off
`.thing[hidden] { display: none }` rules that were each written after someone hit that. Toggling `el.hidden`
now just works; don't add another per-element rule.

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
- **Public sign-ups must be OFF** — Authentication → Sign In / Providers → Email → *Allow new users to sign up*.
  `criar-conta-aluno` creates accounts with the service role and is unaffected by it. Leaving it on re-opens
  exactly the hole that Edge Function exists to close.
- **Password policy** — Supabase dashboard → Authentication → Sign In / Providers → Email (not "Policies").
  Off/minimal by default. Two caveats before treating it as a fix: **leaked-password protection requires the
  Pro plan**, and this org is on Free; and more importantly, **the dashboard policy does not reach this app's
  students at all.** `professor_reset_student_password` and `confirm_student_password_reset` write
  `auth.users.encrypted_password` directly via `crypt()`, bypassing GoTrue, so the only rule in force for a
  student password is the `length < 6` check inside those two functions. Hardening student passwords means
  editing those functions; the dashboard setting only covers the professor's own account.

## Claude Code setup in this repo

`.claude/skills/` has 19 skills installed from `obra/superpowers`, `vercel-labs/skills`,
`supabase/agent-skills`, and `pbakaus/impeccable` (plus 4 companion agents from `impeccable` in
`.claude/agents/`). `.claude/settings.json` wires `impeccable`'s `PostToolUse`/`Stop` hooks so it runs
automatically after UI edits. `superpowers`' own session-start automation (auto-injecting its
`using-superpowers` skill at session start) was **not** installed — its hook relies on `${CLAUDE_PLUGIN_ROOT}`,
which is only set when a skill is installed as a real Claude Code plugin, not when its `skills/` folder is
copied in directly as it was here.
