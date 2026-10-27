# MatchFlow — Sports Tournament Scheduling & Bracket Management

HTML + CSS + vanilla JavaScript front end, Supabase (Auth + PostgreSQL) back end.
Two account types: **Organizer** (full control) and **Participant** (read-only).

## Folder structure
```
matchflow/
├─ index.html
├─ css/styles.css
├─ js/config.js            <- put your Supabase URL + anon key here
├─ js/app.js
└─ supabase/
   ├─ setup_all.sql        <- RUN THIS ONE (it contains files 01-06 in order)
   ├─ 01_tables.sql        tables, keys, constraints
   ├─ 02_auth_roles.sql    sign-up role assignment + access code check
   ├─ 03_validation_triggers.sql   status rules, registration rules, activity log
   ├─ 04_procedures.sql    scoring rules, bracket, results, scheduling
   ├─ 05_views.sql         standings
   ├─ 06_security_rls.sql  Row Level Security + permissions
   ├─ 07_demo_data.sql     optional: demo tournament + 8 teams
   └─ 00_reset.sql         optional: wipe everything and start again
```
The numbered files are the same code as `setup_all.sql`, split by topic so each part is easy to read and edit.

## Setup
1. Create a project at https://supabase.com.
2. **SQL Editor → New query** → paste all of `supabase/setup_all.sql` → **Run**.
   The result at the bottom shows your **Organizer access code** (like `MF-1A2B3C4D`). Save it.
3. (Optional) run `supabase/07_demo_data.sql` for demo data.
4. **Project Settings → API**: copy *Project URL* and *anon public key* into `js/config.js`.
5. **Authentication → Providers → Email**: for testing, turn off *Confirm email*.
6. Open `index.html` (or use VS Code *Live Server*).

## Using the two account types
On the login screen pick **Participant** or **Organizer** first.

| | Participant | Organizer |
|---|---|---|
| Create account | name, email, password | name, email, password **+ access code** |
| Sign in | choose *Participant* | choose *Organizer* |
| Can do | view published tournaments, brackets, schedules, results | create tournaments, manage teams, generate brackets, schedule, record scores, publish |

- Signing in with the wrong account type is blocked with a clear message.
- The access code is checked again inside the database, so it cannot be bypassed from the browser.
- To see your code again or change it (SQL Editor):
  ```sql
  select value from public.app_settings where key = 'organizer_access_code';
  update public.app_settings set value = 'MY-NEW-CODE' where key = 'organizer_access_code';
  ```
- Fallback: promote an existing account with
  `update public.profiles set role = 'organizer' where email = 'you@example.com';`

## Demo flow
Organizer: Tournaments → create → open registration → Participants → approve teams → Brackets → Generate →
Match schedule → Auto-schedule → Score recording → finalize results → Results & standings → **Publish**.
Participant: sees the published tournament, bracket, schedule and results (read-only).

## What the database enforces
| Rule | Where |
|---|---|
| Valid dates, capacity, sport, format | CHECK constraints (01) |
| Role assignment, access code | `handle_new_user`, `check_organizer_code` (02) |
| Status transitions, locks after progress, duplicate teams, capacity | triggers + unique indexes (03) |
| Bracket, byes, third place, round robin, scores, advancement, no double finalize | procedures (04) |
| Team / court overlap | `set_match_schedule`, `generate_schedule` (04) |
| Standings from finalized matches only | view (05) |
| Who can read / write what | RLS (06); `matches` has no client write policy |

## Sport scoring rules used (assumptions — edit `validate_score()` and `scoreRuleError()` to change)
- **Basketball** — total points, no ties.
- **Volleyball / Badminton** — enter sets won, best of 3 (2-0 or 2-1).
- **Pickleball** — to 11, win by 2 (above 11 the margin must be exactly 2).

## Tested
`setup_all.sql` was run on PostgreSQL 16 with a stand-in for Supabase Auth. Verified: role assignment and wrong-code rejection,
duplicate teams, byes, third-place match, round robin, court/team conflicts, tie and pickleball score rules, double-finalize
protection, and Participants being blocked from every write. Not tested against a live Supabase project.
