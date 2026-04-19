# World Cup 2026 Prediction App — Requirements Document

**Version:** 6.0
**Date:** April 18, 2026
**Status:** Final — AI-buildable, Oracle APEX stack, $100 budget ceiling

---

## 1. Overview

### 1.1 Purpose
A fully autonomous web application that lets friends and the public predict FIFA World Cup 2026 matches, compete in groups, and receive LLM-driven insights on their predictions and performance.

### 1.2 Scope
Full FIFA World Cup 2026 tournament (48 teams, 104 matches) from group stage to final. All match data is pulled automatically from a third-party football API — no manual score entry. LLM-generated insights are funded from a fixed budget.

### 1.3 Build Constraint
Every part of this application must be generated end-to-end by AI. The entire deployed system must cost no more than **USD 100** across its lifetime (tournament window plus development). Features requiring human labor (legal drafting, customer support, ongoing content moderation, contract negotiation) or paid services outside the approved budget lines are excluded.

---

## 2. Core Principles

- Fully autonomous operation.
- Server-side enforcement of all business logic.
- Event-driven architecture (`DBMS_SCHEDULER` jobs + triggers).
- External data cached locally; app degrades gracefully when APIs are unavailable.
- LLM features are optional polish — the app must function fully if the LLM is disabled.
- Hard budget ceiling enforced via feature flags and rate limits.

---

## 3. Technology Stack

### 3.1 Platform
- **Database:** Oracle Database 26ai on Oracle Autonomous Database, Always Free tier.
- **REST layer:** Oracle REST Data Services (ORDS) for both inbound endpoints and outbound calls to the football API and LLM via `APEX_WEB_SERVICE` / `UTL_HTTP`.
- **Application layer:** Oracle APEX 24.x or newer, co-hosted with the Autonomous Database.
- **Primary language:** PL/SQL. SQL for the data model and queries. JavaScript only where APEX Dynamic Actions are insufficient.
- No separate backend service — APEX, ORDS, and PL/SQL packages are the entire backend.

### 3.2 Deployment
- Oracle Cloud Infrastructure (OCI) Always Free tier.
- Region: **Frankfurt**.
- Single Autonomous Database instance hosting ORDS and APEX.
- Custom domain: `wc2026.united-codes.com`.

### 3.3 Scheduling
`DBMS_SCHEDULER` jobs handle:
- Football API sync (fixtures, live scores, standings, top scorers).
- Scoring recalculation when matches finish.
- LLM batch jobs (nightly AI Coach, Smart Alerts).
- Budget monitoring and kill-switch enforcement.

---

## 4. External Data Source — Football API

### 4.1 Provider
**API-Football** (api-sports.io).

- **Free tier** (100 req/day): used during development and off-tournament periods.
- **Pro tier** ($19/month, 7,500 req/day): activated for the two-month window covering June and July 2026 to support 30-second live polling. Estimated cost: **~$38**.

### 4.2 Required Data
- Fixture list with kickoff time (UTC), venue, group, round, teams, flags/crests.
- Live and final scores, including extra time and penalty shootouts.
- Match status (scheduled, live, half-time, finished, postponed, cancelled).
- Group stage standings (W/D/L, GF, GA, GD, points).
- Match events: goals, yellow/red cards, substitutions.
- Top scorers list.

### 4.3 Sync Strategy
- **Fixtures:** hourly outside match windows; every 5 minutes during match windows.
- **Live scores and events:** every 30 seconds while any match is in the "live" state.
- **Standings and top scorers:** every 15 minutes while the tournament is active.

### 4.4 Data Integrity
- All raw API responses stored in `API_CALL_LOG` for debugging and quota tracking.
- Duplicate detection via SHA-256 checksum of the payload.
- Valid match-state transitions: `scheduled → live → finished`, plus `scheduled → postponed → scheduled` and `scheduled → cancelled`. Unexpected transitions log a warning.
- Match and result data persisted in local tables so the app keeps working if the API is briefly unavailable.

### 4.5 Resilience
- Graceful degradation when the football API is down: cached data shown with a "last updated" timestamp.
- `DBMS_SCHEDULER` job failures logged and retried with exponential backoff.

---

## 5. Authentication

### 5.1 Supported Login Methods
- **Google** (OAuth 2.0 / OpenID Connect) — APEX Social Sign-In scheme.
- **Facebook** (OAuth 2.0) — APEX Social Sign-In.
- **Email + one-time code** — custom APEX authentication scheme. A PL/SQL package generates a 6-digit code, stores it hashed (SHA-256 + salt) with a 10-minute expiry, sends the email via `APEX_MAIL` using Oracle Cloud Email Delivery as the SMTP gateway, and verifies the code on return.

No traditional password login.

### 5.2 Account Rules
- One account per verified email. Multiple social identities can link to the same account (matched by verified email).
- APEX session cookies are HTTP-only and signed; session timeout configurable in APEX application settings.
- Sign out from all devices from the profile page.

### 5.3 Security
- Login codes stored hashed; raw codes never persisted.
- ORDS rate limiting on the email-code request endpoint: **max 3 requests per email per 10 minutes**.

### 5.4 Profile Data
- Display name (unique)
- Email (verified)
- Avatar URL (from social provider; otherwise an initials-based avatar generated in PL/SQL)
- Favorite team (optional; one of the 48 participants)
- Timezone (auto-detected, overridable)

---

## 6. Core Features

### 6.1 Groups / Private Leagues
- Any user can create a group.
- Each group has a shareable invite link and a 6-character code.
- Groups can be public (listed in a public directory) or private (invite-only).
- Creator is the admin — can rename the group, remove members, and adjust scoring rules within allowed ranges.
- A user can be in multiple groups.
- **Late join:** a user joining mid-tournament starts scoring from their join date; no backfill of missed predictions is allowed, and their historical points for earlier matches are zero in that group's leaderboard.

### 6.2 Match Predictions
- For every scheduled match, users submit a predicted score (home goals, away goals) via an APEX form.
- Editable until kickoff; locked automatically at kickoff on the server side (enforced by a PL/SQL check, not by client JavaScript).
- Bulk entry page shows all upcoming matches of a matchday in a single APEX Interactive Grid.
- Missed matches score zero; no auto-default.
- **Confidence score:** users can optionally tag each prediction with a 0–100% confidence value, stored alongside the prediction. Display-only in v1 — does not affect scoring. Enables future features and personal analytics.
- **Countdown to lock** on every upcoming-match card.
- **Post-match points breakdown:** each prediction shows how points were earned (exact/diff/result + multiplier applied) in §7.4 format.

### 6.3 Knockout Bracket Predictions
- Before the tournament, users fill out the knockout bracket (Round of 32 through Final) plus:
  - Tournament winner
  - Top scorer (Golden Boot)
- Locks at the kickoff of the tournament's first match.

### 6.4 Joker / Double-Up
- Once per tournament stage, each user can flag one match prediction as a "joker" to double the points earned on it.
- Joker can be reassigned within the same stage until the kickoff of the chosen match.

### 6.5 Leaderboards
- Global (all public users).
- Per group.
- Per stage (group stage only, knockout only, etc.).
- Rendered as APEX Interactive Reports with rank, name, avatar, total points, exact-score count, rank change since last match.
- Tiebreakers: total points → exact scores → correct results → earliest signup.
- Rank recomputed via a materialized view refreshed after each match finishes.
- **Rank evolution graph** per user (line chart of rank over matchdays).
- **Points history** per user.
- **Tie-break explanation** tooltip showing which rule decided a tie.

### 6.6 Activity Feed
- Per-group feed showing:
  - Predictions submitted (without revealing the actual prediction until kickoff).
  - Joker usage (revealed at kickoff).
  - Rank changes after each match.
  - High-scoring events (user scored 8+ in a single match, 3-in-a-row exact scores, etc.).

### 6.7 Personal Dashboard
- APEX Home Page with cards for:
  - Upcoming matches needing predictions, with countdown to lock.
  - Recent results and points earned.
  - Current rank in each group.
  - Personal stats: total points, accuracy %, exact-score count, best single match, current correct-prediction streak, risk profile (average confidence, share of unusual predictions).

### 6.8 Head-to-Head Comparison
- Select another member of a shared group; see prediction-by-prediction comparison across all matches.
- Similarity percentage (share of matches with identical predictions).
- Win/loss record (per match, whoever scored more points).

### 6.9 Match Schedule & Results
- All 104 matches in an APEX Calendar or Interactive Report, with kickoff converted to the user's local timezone.
- Filter by date, group, team, stage, venue, status.
- Match detail page: teams and flags, venue, kickoff, live score, event timeline (goals, cards, subs), user's prediction vs. result.
- **Upset indicator:** flag matches where the result ran against the crowd prediction by a wide margin.
- **Popular predictions:** per-match chart showing the distribution of user predictions, revealed at kickoff.

### 6.10 What-If Simulator
- User enters hypothetical results for remaining matches.
- Server recomputes leaderboard assuming those results using pure SQL and shows projected rank.
- Interactive; no persistence — each simulation is ephemeral.

### 6.11 Crowd Insights
- Aggregate stats over the app's own users: percentage predicting home/draw/away, average predicted score, most popular exact score per match.
- Pure SQL aggregation — no LLM cost.

### 6.12 Rival Mode
- Each user can nominate one rival per group.
- A mini two-person leaderboard tracks the head-to-head points differential over time.
- Rival can be changed once per stage.

### 6.13 Share Cards
- On-demand PNG generation via an ORDS endpoint that renders SVG server-side.
- Covers: leaderboard position, post-match summary, final end-of-tournament card.
- Downloadable from the app. No external rendering service.

### 6.14 Timezone Handling
- Kickoff times stored as `TIMESTAMP WITH TIME ZONE`.
- User timezone auto-detected from the browser, stored on profile, overridable.
- All APEX reports display kickoff converted to the user's timezone.

### 6.15 In-App Notifications
- Real-time in-app notifications for: group invitations, new members joining a user's group, rank changes, joker reminders, match-lock warnings.
- No bulk external email beyond the login code. Users who want kickoff reminders receive them via the in-app notification bell.
- Optional single opt-in daily digest email via `APEX_MAIL`: one summary per active user per matchday, rate-limited server-side to stay within Oracle Cloud Email Delivery free-tier limits. Off by default.

---

## 7. Scoring System

### 7.1 Base Points
Default (configurable per group within fixed ranges):

| Outcome | Points |
|---|---|
| Exact score correct | 5 |
| Correct result + correct goal difference | 3 |
| Correct result only | 1 |
| Wrong | 0 |

### 7.2 Stage Multipliers
- Round of 16 and Quarter-finals: ×1.5
- Semi-finals and Third-place match: ×2
- Final: ×3

### 7.3 Bracket and Special Bonuses
- Each correct knockout qualifier: +2
- Correct finalist: +5
- Correct tournament winner: +15
- Correct Golden Boot winner: +10

### 7.4 Per-Prediction Storage
Each row in `PREDICTIONS` stores:
- `exact_points` (0 or 5)
- `diff_points` (0 or 3)
- `result_points` (0 or 1)
- `multiplier_applied` (number)
- `joker_applied` (Y/N)
- `points_awarded` (final total for that row)

Makes audits, recomputation, and per-prediction breakdowns trivial.

### 7.5 Penalty Shootouts
Matches decided on penalties count as a draw for the "result" line; the shootout winner advances in bracket predictions.

### 7.6 Scoring Engine
All scoring logic lives in a single PL/SQL package (`PKG_SCORING`). Triggered by `DBMS_SCHEDULER` whenever a match status transitions to "finished", so leaderboards update automatically.

---

## 8. LLM Features

### 8.1 Provider
**Claude Haiku 4.5** via the Anthropic API. Chosen because it has the lowest current-generation pricing ($1/M input, $5/M output), supports prompt caching (90% off cached reads) and batch API (50% off non-urgent calls), and matches Sonnet 4 on quality for tasks of this complexity.

### 8.2 Features

#### 8.2.1 AI-Assisted Predictions
- On demand, from the match prediction page.
- Input: recent form of both teams, head-to-head summary, top scorers, kickoff context.
- Output: suggested score + 2–3 sentence explanation citing the data.
- User can accept, edit, or ignore the suggestion.
- **Prompt caching** on the per-match context (1-hour TTL) so repeated user requests for the same match cost ~10% of the first call.

#### 8.2.2 AI Coach
- Weekly per-user summary of prediction patterns: team bias, draw/home/away tendencies, risk profile, notable successes and misses.
- Run as a batch job on Sunday nights during the tournament using Anthropic's Batch API (50% off).
- Output displayed in the personal dashboard.

#### 8.2.3 Joker Recommendation
- Once per stage, per user.
- Input: user's accuracy history, confidence distribution, upcoming matches in the stage.
- Output: recommended match to joker + one-line justification.

#### 8.2.4 Smart Alerts
- Pre-match (morning of match day), system-wide — one alert per match, shared across all users.
- Input: form, odds proxies from team stats, recent results.
- Output: headline ("Possible upset alert") + one-line rationale.
- Runs as a batch job, so cost is trivial.

### 8.3 Budget Controls
- **Hard monthly cap** set in the Anthropic dashboard at $60.
- **Per-user rate limits:** max 5 AI prediction suggestions per user per day; 1 AI coach run per user per week; 1 joker recommendation per user per stage.
- **Prompt caching** on static context for every call.
- **Batch API** for all non-interactive calls (AI Coach, Smart Alerts).
- **Kill switch** in `APP_CONFIG` (key `llm_enabled`): when false, all LLM features are hidden from the UI and the app continues to function without them. Toggled automatically if the monthly cap is reached.
- Every LLM call logged to `LLM_CALL_LOG` with token counts and cost, so spend is visible in the admin dashboard.

### 8.4 Graceful Degradation
- If the Anthropic API is unavailable, LLM features show "Insight temporarily unavailable" — core prediction and scoring features are unaffected.

---

## 9. Data Model

All tables live in a dedicated application schema (e.g. `WC2026`).

### 9.1 Core Tables
- **USERS** — `id` (PK), `email` (unique), `display_name` (unique), `avatar_url`, `timezone`, `favorite_team_id`, `theme_preference` (light/dark/system), `risk_profile` (computed), `created_at`.
- **AUTH_IDENTITIES** — `id` (PK), `user_id` (FK), `provider` (google/facebook/email), `provider_user_id`, `linked_at`.
- **TEAMS** — `id` (PK), `external_api_id`, `name`, `country_code`, `flag_url`, `flag_blob` (BLOB, cached), `group_code`.
- **VENUES** — `id` (PK), `external_api_id`, `name`, `city`, `country`.
- **MATCHES** — `id` (PK), `external_api_id`, `home_team_id` (FK), `away_team_id` (FK), `kickoff_utc`, `venue_id` (FK), `stage`, `group_code`, `status`, `home_score`, `away_score`, `pens_home`, `pens_away`, `updated_at`.
- **MATCH_EVENTS** — `id` (PK), `match_id` (FK), `minute`, `event_type`, `player_name`, `team_id` (FK).
- **GROUPS** — `id` (PK), `name`, `description`, `invite_code` (unique), `admin_user_id` (FK), `scoring_rules` (JSON), `visibility`, `created_at`.
- **GROUP_MEMBERSHIPS** — `group_id` (FK), `user_id` (FK), `joined_at`. Composite PK.
- **PREDICTIONS** — `id` (PK), `user_id` (FK), `match_id` (FK), `pred_home`, `pred_away`, `confidence_pct`, `is_joker`, `exact_points`, `diff_points`, `result_points`, `multiplier_applied`, `points_awarded`, `submitted_at`, `locked_at`. Unique index on (`user_id`, `match_id`).
- **BRACKET_PREDICTIONS** — `id` (PK), `user_id` (FK), `stage`, `slot`, `predicted_team_id` (FK), `points_awarded`.
- **SPECIAL_PREDICTIONS** — `id` (PK), `user_id` (FK), `type` (winner/top_scorer), `predicted_value`, `points_awarded`.
- **RIVALS** — `user_id` (FK), `group_id` (FK), `rival_user_id` (FK), `set_at`. Composite PK.

### 9.2 Supporting Tables
- **ACTIVITY_LOG** — `id` (PK), `group_id` (FK), `user_id` (FK), `event_type`, `payload` (JSON), `created_at`.
- **NOTIFICATIONS_QUEUE** — `id` (PK), `user_id` (FK), `type`, `payload` (JSON), `read_at`, `created_at`.
- **USER_STATS** — `user_id` (PK/FK), `total_points`, `accuracy_pct`, `exact_scores`, `best_match_points`, `current_streak`, `last_refreshed`.
- **PREDICTION_ANALYTICS** — `match_id` (FK), `home_pct`, `draw_pct`, `away_pct`, `avg_home_score`, `avg_away_score`, `most_common_score`, `last_refreshed`.
- **AI_SUGGESTIONS** — `id` (PK), `user_id` (FK), `match_id` (FK, nullable), `feature` (prediction/coach/joker/alert), `suggestion` (JSON), `created_at`.
- **API_CALL_LOG** — `id` (PK), `endpoint`, `status_code`, `called_at`, `latency_ms`, `checksum`, `error_message`.
- **LLM_CALL_LOG** — `id` (PK), `feature`, `model`, `input_tokens`, `output_tokens`, `cached_tokens`, `cost_usd`, `called_at`.
- **SYSTEM_HEALTH** — `subsystem` (PK: api_sync, scoring, llm, leaderboard, retention), `last_success_at`, `last_status_code`, `last_error_message`, `expected_interval_seconds`, `updated_at`.
- **AUDIT_LOG** — `id` (PK), `user_id` (FK), `action`, `entity`, `before` (JSON), `after` (JSON), `created_at`.
- **APP_CONFIG** — `key` (PK), `value`, `updated_at`. Holds API keys, OAuth secrets, `llm_enabled` flag, monthly spend counters, global settings.

### 9.3 Key PL/SQL Packages
- `PKG_API_SYNC` — football API calls, JSON parsing, upserts.
- `PKG_SCORING` — point calculation for predictions, brackets, specials.
- `PKG_AUTH` — email one-time-code generation and verification; social identity linking.
- `PKG_LEADERBOARD` — materialized-view refresh and ranking with tiebreakers.
- `PKG_LLM` — LLM prompt construction, caching, batch submission, budget tracking, kill switch.
- `PKG_NOTIFICATIONS` — in-app notification fan-out.
- `PKG_ADMIN` — administrative overrides and announcements.

---

## 10. Edge Case Handling

- **Postponed match:** predictions remain editable until the rescheduled kickoff.
- **Cancelled match:** excluded from scoring; associated predictions refunded to zero points; any joker on that match is returned to the user for the remainder of the stage.
- **API failure:** fallback to cached data with "last updated" timestamp.
- **Duplicate API payload:** ignored via checksum match.
- **Score correction from the API:** post-finish score change triggers scoring recalculation and a notification to affected users.
- **LLM API failure:** feature hidden; app unaffected.

---

## 11. Non-Functional Requirements

### 11.1 Performance
- Page load under 2 seconds on broadband.
- Live score updates visible within 60 seconds of the source API update.
- Database indexes on: `PREDICTIONS(user_id, match_id)`, `PREDICTIONS(match_id)`, `MATCHES(kickoff_utc, status)`, `GROUP_MEMBERSHIPS(group_id)`, `ACTIVITY_LOG(group_id, created_at)`.
- Materialized views for leaderboards and per-match analytics.

### 11.2 Security
- HTTPS enforced by Oracle Autonomous Database / ORDS.
- API keys and OAuth secrets stored in `APP_CONFIG`, readable only by the application schema.
- One-time login codes stored hashed (SHA-256 + salt).
- Prediction lock enforced server-side by a PL/SQL check; client-side lock is cosmetic.
- ORDS rate limiting on the email-code request endpoint (max 3 per email per 10 minutes).
- SQL injection prevented by bind variables throughout; no dynamic SQL with concatenated user input.
- Output escaping on all user-generated content via APEX built-in sanitization.
- Prediction rows checksummed to detect tampering.
- Audit log of prediction edits and administrative overrides.

### 11.3 Device Support
- APEX Universal Theme provides responsive design: mobile, tablet, desktop.
- Last two major versions of Chrome, Safari, Firefox, Edge.
- No native mobile app.

### 11.4 Accessibility
- APEX Universal Theme is WCAG-oriented by default.
- Keyboard navigation on all interactive elements.
- Results never conveyed by color alone.

---

## 12. Integrity, Observability & Retention

### 12.1 Prediction Lock Integrity
- A database-level assertion ensures: **no prediction inserts or updates are allowed where `SYSTIMESTAMP >= MATCHES.kickoff_utc`**. Enforcement happens in the database, not the application layer.
- All lock validations use **database time** (`SYSTIMESTAMP` / `SYSDATE`), never application-server or client time.
- Prediction updates use `SELECT ... FOR UPDATE` to acquire a row lock before writing, preventing concurrent writes from racing.

### 12.2 Scheduler Idempotency
- Every `DBMS_SCHEDULER` job must be idempotent — re-running a job on the same input produces the same result and has no harmful side effects.
- Achieved by:
  - Using `MERGE` statements instead of bare `INSERT`.
  - Status flags on work items (`processed_at`, `scored_at`).
  - Explicit "has this already been handled?" guards at the start of each job.
- **Scoring job contract:** only processes matches where status transitioned to `finished` **and** `points_awarded` is NULL on the prediction row. A re-run finds no unprocessed work and is a no-op.

### 12.3 Leaderboard Consistency
- Leaderboard positions are always derived from source-of-truth tables (`PREDICTIONS`, `BRACKET_PREDICTIONS`, `SPECIAL_PREDICTIONS`).
- No ranks or totals are persisted as writable state — `USER_STATS` and the leaderboard materialized view are caches that can be fully rebuilt from the predictions table at any time.
- Recalculation must be deterministic: rebuilding from scratch produces identical leaderboards. A nightly integrity job rebuilds and compares against the live cache, logging any divergence.

### 12.4 Prediction Visibility
- **Before kickoff:** a user can only see their own prediction for a match. Other users' predictions (including in groups, head-to-head views, and crowd insights) are hidden.
- **At/after kickoff:** all predictions for that match become visible to all members of the user's groups, and the match contributes to crowd-insight aggregates.
- **After the match is finished:** points and the per-row points breakdown (§7.4) become visible alongside each prediction.
- Visibility enforced in the database views/PL/SQL used by APEX reports, not only in the UI layer.

### 12.5 Abuse Protection
- **Per-user caps:**
  - Maximum 20 groups per user.
  - Maximum 200 members per group.
- **Invite-code brute-force protection:** ORDS rate-limits join-by-code attempts to a sensible ceiling (e.g. 10 attempts per IP per 10 minutes); repeated failures on distinct codes also counted and throttled per user.
- Invite codes are 6 characters from a 32-character alphabet (~1 billion combinations) and are re-generatable by the group admin if suspected compromised.
- Failed join attempts logged to `AUDIT_LOG`.

### 12.6 Data Retention
- **`API_CALL_LOG`:** raw entries older than 30 days are purged nightly. Summary counters (daily totals per endpoint, error counts) are retained in an aggregate table indefinitely for admin analytics.
- **`LLM_CALL_LOG`:** detailed rows are aggregated into a monthly summary at tournament end; detail rows older than 30 days post-tournament are purged, summary rows kept indefinitely.
- **`ACTIVITY_LOG`:** retained in full — volume is low and users revisit their history in the post-tournament archive.
- **`AUDIT_LOG`:** retained in full — needed for integrity checks and dispute resolution.
- Retention jobs run as `DBMS_SCHEDULER` tasks and respect the idempotency rules in §12.2.

### 12.7 Observability
- **`SYSTEM_HEALTH`** table, single row per subsystem, updated on every run:
  - `last_successful_api_sync` (per endpoint)
  - `last_successful_scoring_run`
  - `last_successful_llm_call`
  - `last_successful_leaderboard_refresh`
  - `last_successful_retention_job`
  - Plus status code and error message of the most recent failure.
- Admin dashboard (§13.2) prominently highlights any subsystem whose last successful run is older than its expected interval — red banner on the home page of the admin app.
- Failed `DBMS_SCHEDULER` runs are written to `AUDIT_LOG` with stack traces.
- A weekly digest of system health is surfaced in the admin dashboard (no external alerting in v1; a human operator checks the admin dashboard).

---

## 13. Administrative Features

### 13.1 Group Admin (in-app)
- Rename group, change description.
- Remove members.
- Adjust scoring rules within allowed ranges.
- Close group to new members.

### 13.2 Platform Admin (in-app)
- APEX admin application showing:
  - Football API call counts and error rates.
  - LLM token usage, spend-to-date, projected monthly spend.
  - Scheduler job status.
  - Active users, predictions submitted, groups created.
  - System health summary from §12.7 with failure highlighting.
- Override a match result (writes an audit record and retriggers scoring).
- Toggle feature flags in `APP_CONFIG`, including the LLM kill switch.
- Post an announcement banner visible to all users.

---

## 14. Post-Tournament Mode

- After the final whistle on July 19, 2026:
  - Freeze all scores and leaderboards.
  - Generate a shareable PNG summary for each user.
  - Archive data; the app stays online in read-only mode indefinitely so users can revisit their history.
  - Football API sync job disabled; API-Football Pro subscription cancelled.

---

## 15. Tournament Timeline

- **Development window:** from project start until end of May 2026.
- **Soft launch:** early June 2026.
- **Bracket predictions open:** at launch, lock at first match kickoff.
- **Tournament start:** June 11, 2026.
- **Tournament end:** July 19, 2026.
- **Post-tournament archive:** permanent, read-only.

---

## 16. Future Extensions

Out of scope for v1 but structurally supported:
- Multi-tournament support (UEFA Euro 2028, FIFA World Cup 2030).
- Additional club competitions (Champions League, Europa League).
- Premium groups and sponsorships — **note:** these would reintroduce payments and therefore lift the "AI-buildable, no contracts" constraint. Flagged as future work only.

---

## 17. Budget

Total ceiling: **USD 100** across the application's lifetime.

| Item | Provider | Estimated cost |
|---|---|---|
| Football data | API-Football Pro, 2 months (June + July 2026) | **~$38** |
| LLM inference | Claude Haiku 4.5 (with prompt caching + batch API) | **~$30–45** |
| Hosting, database, email | Oracle Cloud Always Free tier | **$0** |
| Domain | `wc2026.united-codes.com` (existing subdomain) | **$0** |
| **Estimated total** | | **~$68–83** |
| Headroom for overruns / user-growth / upgrading some calls to Sonnet 4.6 for high-stakes matches | | **~$15–30** |

Budget is enforced by:
- Hard monthly caps in the Anthropic dashboard ($60/month).
- `APP_CONFIG.llm_enabled` kill switch that auto-flips when the cap is reached.
- Per-user rate limits on LLM features.
- Monitoring dashboard visible to the platform admin.

---

## 18. Design Specification

### 18.1 Visual Identity
Clean and editorial — the closest references are ESPN, FIFA.com, and The Athletic. The data is the hero; decoration is minimal. Generous whitespace, strong typographic hierarchy, card-based layout for everything that can be a card (matches, groups, activity items, share cards).

### 18.2 Color Tokens

All colors defined as CSS custom properties so a user's light/dark preference swaps only the variable values, not the component definitions.

**Light theme (default):**

| Token | Hex | Use |
|---|---|---|
| `--color-primary` | `#006B7F` | Deep teal. Headers, primary buttons, active tabs, links |
| `--color-primary-hover` | `#005566` | Primary hover/active state |
| `--color-accent` | `#E8523C` | Warm red-orange. Live indicators, joker highlights, lock-imminent warnings |
| `--color-success` | `#2E8B57` | Correct predictions, points earned, finished matches |
| `--color-warning` | `#E8A33C` | Amber. Draws, warnings, pending actions |
| `--color-danger` | `#C0392B` | Errors, destructive actions |
| `--color-bg` | `#FFFFFF` | Page background |
| `--color-bg-subtle` | `#F5F5F5` | Card backgrounds, alt row banding |
| `--color-border` | `#E0E0E0` | Dividers, card borders |
| `--color-text` | `#1A1A1A` | Primary text |
| `--color-text-muted` | `#666666` | Secondary text, metadata |

**Dark theme (user toggle):**

| Token | Hex |
|---|---|
| `--color-primary` | `#4FB3C8` (lightened teal for contrast) |
| `--color-primary-hover` | `#6FC8DB` |
| `--color-accent` | `#FF7258` (lightened red-orange) |
| `--color-success` | `#4FBF7F` |
| `--color-warning` | `#F5B860` |
| `--color-danger` | `#E57368` |
| `--color-bg` | `#0F1418` |
| `--color-bg-subtle` | `#1A2128` |
| `--color-border` | `#2A323A` |
| `--color-text` | `#F0F0F0` |
| `--color-text-muted` | `#9AA5B0` |

**Contrast:** every text-on-background combination meets WCAG AA (4.5:1 for body text, 3:1 for large text). Verified during theme definition; re-checked if any token is changed.

**Theme switching:** toggle in the header. Preference stored on the user profile (`USERS.theme_preference`: `light` / `dark` / `system`). Default: `system` (follows OS preference).

### 18.3 Typography

System font stack — no external webfonts. Keeps performance tight and avoids a CDN dependency.

```css
--font-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
             "Helvetica Neue", Arial, sans-serif;
--font-mono: "SF Mono", Monaco, Consolas, "Liberation Mono", monospace;
```

Type scale:

| Role | Size | Weight | Line height |
|---|---|---|---|
| Display (hero numbers on share cards) | 48 px | 700 | 1.1 |
| Page title (h1) | 32 px | 700 | 1.2 |
| Section title (h2) | 24 px | 600 | 1.3 |
| Card title (h3) | 18 px | 600 | 1.4 |
| Body | 16 px | 400 | 1.5 |
| Small / metadata | 13 px | 400 | 1.4 |
| Score numbers on match cards | 28 px | 700 | 1.0 |

Scores use tabular numerals (`font-variant-numeric: tabular-nums`) so columns of numbers line up.

### 18.4 Spacing & Layout

4 px base grid. Spacing scale: `4, 8, 12, 16, 24, 32, 48, 64` px.

**Page layout:**
- Max content width: 1200 px, centered.
- Responsive breakpoints: `640 px` (tablet), `1024 px` (desktop).
- Mobile: single column, bottom-fixed nav bar.
- Tablet/desktop: left sidebar nav + main content area.

### 18.5 Component Inventory

**Match Card** — the app's core unit.
- Two team rows (flag, team name, predicted score, actual score).
- Kickoff time + venue line.
- Status chip (SCHEDULED / LIVE / FT / POSTPONED / CANCELLED).
- Joker star icon if the user jokered this match.
- Points badge in the corner once the match is scored.
- Tap/click opens the match detail page.

**Status Chips:**
- `SCHEDULED` — neutral grey background, dark text.
- `LIVE` — red-orange background with pulsing dot, white text.
- `FT` (finished) — teal background, white text.
- `LOCKED` — amber background, dark text (shown in the minute before kickoff).
- `POSTPONED` — slash pattern, muted colors.

**Prediction Entry:**
- Two number steppers for home/away (0–15 range), large tap targets on mobile (44 px minimum).
- Optional confidence slider (0–100%), labelled "How confident?"
- Joker toggle only visible on the bulk-entry page (one per stage).
- Countdown to lock shown prominently.

**Leaderboard Row:**
- Rank number, trend arrow (↑ / ↓ / —) from last match.
- Avatar, display name.
- Points (large, tabular figures).
- Exact-score count (small, muted).
- Gold/silver/bronze highlight on top 3.

**Group Header:**
- Group name, member count, visibility pill (public/private).
- Invite code with one-tap copy button (for admins).
- Stage-filter tabs below.

**Activity Feed Item:**
- Avatar, name, action verb ("joker used on", "scored 8 points in", "joined").
- Relative timestamp ("2h ago").
- Optional context link (to the match or another user).

**Share Card** (rendered as SVG via ORDS):
- 1200 × 630 px (Open Graph standard).
- Teal background with subtle tournament-themed geometric pattern.
- User's avatar and name.
- Headline statistic (rank, total points, or tournament summary).
- Small `wc2026.united-codes.com` footer.

### 18.6 Iconography

Font APEX (bundled with APEX Universal Theme). 500+ icons cover every need; no third-party icon library. Icons sized to match the text baseline — 16 px for inline, 20 px for buttons, 24 px for navigation.

### 18.7 Flags & Imagery

- Country flags sourced from API-Football's flag URLs.
- Cached locally as BLOBs in `TEAMS.flag_blob` (Oracle `BLOB`) to avoid hotlinking and to keep the app working if the CDN is slow.
- Flag aspect ratio 4:3, displayed at 24 × 18 px in match cards, 32 × 24 px in headers.
- No other external imagery in v1 — no stadium photos, no player photos. Keeps the app simple, royalty-free, and fast.

### 18.8 Motion

- Subtle only. Fade-in on load (150 ms). Score updates animate from old to new value (300 ms count-up). Live indicator pulses at 1 s cadence.
- Respects `prefers-reduced-motion`: all transitions disabled if the user's OS requests reduced motion.

### 18.9 Empty & Loading States

Every list view has a defined empty state (illustration-free — a short message + a clear next action) and a skeleton loading state (grey placeholder bars at the card's final dimensions). Skeletons prevent layout shift when data arrives.

### 18.10 Accessibility

- All interactive elements reachable via keyboard; visible focus ring in `--color-primary` at 2 px.
- All images have `alt` text (team flags: "Brazil flag"; decorative imagery: `alt=""`).
- All form inputs have associated `<label>` elements.
- Color never the sole signal — status chips always carry a label, correct/incorrect predictions always carry an icon alongside color.
- Live regions (`aria-live="polite"`) on the live score area so screen readers announce score changes.
- Prediction lock countdown announced via live region in the last 60 seconds.

### 18.11 APEX Implementation Notes

- Built on APEX Universal Theme 42+ (Vita style).
- CSS custom properties defined in a single application-level CSS file, loaded via the APEX Shared Components.
- Theme roller used for the initial light palette; dark palette hand-tuned and applied via a `data-theme="dark"` attribute on `<html>` set by a small JavaScript snippet reading the user's preference.
- No custom JS frameworks — APEX Dynamic Actions and the Universal Theme's built-in components cover every UI need.
- Page templates: use **Standard** for most pages, **Left Side Column** for pages with filters (match schedule, group browser).
- Cards use the built-in **Cards** region type with the "Float" template.
- Leaderboards use **Interactive Report** regions with the default template.

---

## 19. Onboarding Flow

First-time users must land on a clear, short path. No welcome wizard, no tooltip tour — just sensible defaults plus one essential decision at each step.

### 19.1 Sign-up Path
1. Landing page at `wc2026.united-codes.com` — headline, short value proposition, "Sign in" button (Google / Facebook / Email code).
2. On first successful authentication: redirect to **Set Up Profile** page.
3. **Set Up Profile** — display name (pre-filled from social provider, editable), optional favorite team picker (searchable dropdown of 48 teams), theme preference (system / light / dark). "Continue" button.
4. **Join or Create a Group** page — three equal options as cards:
   - Enter an invite code.
   - Browse public groups.
   - Create a new group.
   - Skip for now (lands on dashboard without group membership).
5. If the tournament has not started yet: **Bracket Prediction Prompt** — a banner on the dashboard nudging the user to complete their bracket. Dismissible; re-appears once per week until bracket is submitted.
6. Land on **Personal Dashboard** (§6.7).

### 19.2 Returning Users
- Go straight to the dashboard.
- If there are upcoming matches needing predictions, the dashboard's top card highlights them.

### 19.3 Empty-State Prompts
- **No group membership:** dashboard shows a "Join or create your first group" card.
- **No predictions yet:** upcoming-matches card shows a one-click "predict now" shortcut.
- **No bracket submitted before tournament start:** persistent banner with a "Complete your bracket" button.

### 19.4 Abandoned Sign-up
- A user who starts sign-up (authenticates with a provider) but doesn't complete the profile step is a valid user with a blank display name placeholder (`user_<id>`). They can finish at any time. No separate "pending" state.

---

## 20. Error Handling & Copy

### 20.1 Principles
- Every user-facing error uses consistent tone: brief, plain English, no jargon, no stack traces.
- Every error states what happened + what the user can do next.
- All error strings stored in APEX Text Messages for future localization (see §22).
- Technical details logged to `AUDIT_LOG` or `API_CALL_LOG`; never shown to the user.

### 20.2 Error Catalogue

| Situation | User message | Action offered |
|---|---|---|
| Football API unavailable | "Live data is temporarily unavailable. Showing last known results from [timestamp]." | Auto-retry; no user action required |
| Prediction failed to save | "Your prediction couldn't be saved. Please try again." | Retry button |
| Prediction attempted after kickoff | "This match has already started. Predictions are locked." | Link to match detail page |
| Joker already used this stage | "You've already used your joker in this stage." | Link to the jokered match |
| Invite code invalid | "That invite code doesn't match any group. Check the code and try again." | Retry input |
| Group full (200 members) | "This group is full and can't accept new members right now." | Link to browse public groups |
| User has reached group cap | "You're a member of the maximum 20 groups. Leave one before joining another." | Link to group list |
| Email code expired | "This code has expired. Request a new one." | Resend button |
| Email code incorrect (3 attempts) | "Too many incorrect attempts. Please request a new code." | Resend button, rate-limit warning |
| LLM feature disabled | "AI insights are temporarily unavailable." | None; feature hidden |
| LLM call failed | "We couldn't generate a suggestion right now. Try again in a minute." | Retry button |
| OAuth provider rejected login | "Sign-in with [provider] failed. Try another method." | Back to login page |
| Session expired | "Your session has ended. Please sign in again." | Redirect to login |
| Generic 500 | "Something went wrong on our side. Please try again." | Retry, plus "report a problem" link |
| Generic 404 | "We couldn't find that page." | Link to dashboard |

### 20.3 Global Error Layout
- Inline errors: shown under the relevant form field, in `--color-danger`, with a small icon.
- Toast notifications for transient errors (auto-dismiss after 5 seconds).
- Full-page errors (500, 404, auth failure): centered card on subtle background, with a primary action button and a secondary "Back to dashboard" link.

---

## 21. Testing Strategy

### 21.1 Test Environments
- **Development:** the built application running against API-Football's free tier, reading real World Cup data as it becomes available.
- **Simulation:** API-Football's `/fixtures?live=all` endpoint on normal days has near-zero World Cup fixtures in progress outside the tournament window. For pre-tournament testing, use the provider's historical endpoint (past World Cup / Euro data) pointed at a past tournament to exercise scheduled → live → finished state transitions.

### 21.2 Automated PL/SQL Tests
- **utPLSQL** (free Oracle unit-testing framework) for `PKG_SCORING`, `PKG_LEADERBOARD`, `PKG_AUTH`, `PKG_LLM`, and `PKG_API_SYNC`.
- Minimum coverage targets:
  - Scoring: every combination of exact/diff/result × stage multiplier × joker, plus penalty-shootout handling, cancelled-match refund, score-correction recalculation.
  - Leaderboard: tiebreaker ordering, rank recomputation after retroactive scoring change.
  - Auth: code generation, expiry, rate limiting, identity linking, hash verification.
  - API sync: idempotent re-run, duplicate checksum rejection, invalid state transition logging.

### 21.3 End-to-End Test Harness
A PL/SQL script `sim_match.sql` that:
1. Inserts a fake fixture into `MATCHES`.
2. Inserts predictions for a set of seed users.
3. Walks the match through `scheduled → live → finished` by calling `PKG_API_SYNC` with synthetic payloads.
4. Asserts scoring, leaderboard, activity feed, and LLM batch jobs all produce correct output.
5. Runs clean — leaves no residue in production tables (uses a dedicated test schema or rolls back in a savepoint).
A Playwright script to do UI testing of the APEX app and all pages.

### 21.4 Seed Data
- `seed_dev.sql` — 20 fake users, 5 groups, predictions scattered across past World Cup fixtures. Enough to render every page meaningfully during development.
- `seed_test.sql` — minimal seed used by utPLSQL tests.

### 21.5 Manual Pre-Launch Testing
- Click-through of every page on mobile (Safari on iOS, Chrome on Android via real devices or emulators) and desktop (Chrome, Firefox, Safari).
- Sign in with each of the three auth methods.
- Create a group, join a group by code, join a public group, leave a group.
- Submit a prediction, edit it, verify it locks at kickoff using a short-fuse fake match.
- Request an AI suggestion, verify caching on repeat request, verify kill switch works.
- Verify dark mode on every page.

---

## 22. Localization Strategy

### 22.1 v1 Language
English only.

### 22.2 Future-Proofing
- All user-facing strings stored in **APEX Text Messages** (Shared Components → Text Messages), not hardcoded in page HTML or PL/SQL.
- Error messages reference Text Message keys (e.g. `ERR_PREDICTION_LOCKED`) rather than containing literal strings.
- Date/time formatting uses the user's locale via APEX built-in formatting.
- Translating to a new language in v2 is then a single task: duplicate the Text Messages set for the new language code. No code changes.

### 22.3 Things Not Localized in v1
- Team names (kept in the API provider's canonical spelling).
- Venue names.
- User-generated content (display names, group names).

---

## 23. Data Privacy Controls (Self-Serve)

Not a formal GDPR compliance programme — a minimum set of features any user would reasonably expect.

### 23.1 Data Export
- "Download my data" button on the profile page.
- Generates a JSON file containing: profile, group memberships, all predictions (match + bracket + special), activity feed entries, LLM suggestions received.
- File delivered via an ORDS endpoint that streams the result and logs the export to `AUDIT_LOG`.

### 23.2 Data Deletion
- "Delete my account" button on the profile page, behind a confirmation dialog.
- Purges: `USERS` row, all `AUTH_IDENTITIES`, all `PREDICTIONS`, `BRACKET_PREDICTIONS`, `SPECIAL_PREDICTIONS`, `GROUP_MEMBERSHIPS`, `NOTIFICATIONS_QUEUE`, `AI_SUGGESTIONS`, `RIVALS`.
- Does **not** purge: `ACTIVITY_LOG` entries referencing the user (replaced with "deleted user" placeholder to avoid breaking group history), `AUDIT_LOG` (needed for integrity).
- Orphaned group admin role transferred to the earliest remaining member; if none remain, group is soft-deleted.
- Deletion logged to `AUDIT_LOG` (keyed by prior email hash, not by cleartext email).

### 23.3 Placeholder Legal Pages
Required because OAuth providers (Google in particular) reject consent screens without a privacy policy URL.
- **`/privacy`** — plain-English placeholder: "This is a personal project built for friends and the public. We store the data needed to run the app (account, predictions, group membership). We do not sell or share data. You can export or delete your data from your profile page."
- **`/terms`** — plain-English placeholder: "Use of this app is free and unmoderated. We make no guarantees about availability. Don't abuse the service. We can remove accounts or groups that violate reasonable norms."
- **`/about`** — short credit line and a contact email.
- These pages are Markdown files rendered by a simple APEX page; content reviewable before launch.

---

## 24. Progressive Web App

APEX supports PWA out of the box from 21.2+.

### 24.1 PWA Enablement
- Enabled via the APEX application settings: **Progressive Web App → Enable PWA**.
- **App name:** "WC2026 Predictions"
- **Short name:** "WC2026"
- **Theme color:** `#006B7F` (matches `--color-primary` light theme)
- **Background color:** `#FFFFFF`
- **Display mode:** `standalone`
- **Start URL:** `/` (dashboard for authenticated users, landing page otherwise)

### 24.2 Icons
- 192 × 192 and 512 × 512 PNG icons generated from a simple mark: teal circle with "WC26" wordmark.
- Generated server-side during build; stored as `icon-192.png` and `icon-512.png` in static files.
- Maskable variant for Android adaptive icons.

### 24.3 Favicon
- 32 × 32 PNG favicon using the same mark.
- `apple-touch-icon` 180 × 180 PNG for iOS home-screen.

### 24.4 Offline Behaviour
- APEX-generated service worker caches shell assets (CSS, JS, icons).
- Dynamic data (predictions, scores) not cached — requires network.
- Custom offline fallback page: "You're offline. Reconnect to see live scores."

### 24.5 Install Prompts
- Browser's native install prompt is shown; no custom install UI in v1.
- "Install this app" hint in the user profile page for users who haven't installed.

---

## 25. Launch Readiness Checklist

Non-code items that must be verified before the tournament starts on June 11, 2026. Work backwards from the launch date; items marked ⚠ have lead times longer than a day.

### 25.1 Accounts & Credentials
- ⚠ **Google OAuth consent screen** — submitted for verification (takes 3–7 business days; longer if sensitive scopes). **Begin at least 2 weeks before launch.**
- ⚠ **Facebook App** — created, app review completed if needed. **Begin at least 1 week before launch.**
- **Anthropic API account** — payment method added, monthly cap set to $60, API key generated, stored in `APP_CONFIG`.
- **API-Football account** — free tier activated for development; Pro plan billing set up for activation on June 1.
- **Oracle Cloud tenancy** — Always Free Autonomous Database created in Frankfurt region, APEX workspace provisioned.
- **Oracle Cloud Email Delivery** — configured, sender address verified, SPF/DKIM records published on `united-codes.com`.

### 25.2 Domain & DNS
- `wc2026.united-codes.com` CNAME pointing to the APEX URL.
- HTTPS certificate in place (Oracle Autonomous Database provides this automatically).
- OAuth redirect URIs configured on both Google and Facebook apps pointing to the production domain.

### 25.3 Application State
- All 48 teams loaded and seeded with flags cached as BLOBs.
- All 104 fixtures loaded from API-Football with correct UTC kickoff times.
- All 16 venues loaded.
- Scoring rules and stage multipliers confirmed in `APP_CONFIG`.
- All `DBMS_SCHEDULER` jobs created, enabled, and verified running.
- All utPLSQL tests passing.
- Seed data removed from production schema.

### 25.4 Content
- Privacy policy, terms, about pages live at their URLs.
- Error messages reviewed for tone consistency.
- PWA icons, favicon, app-icon generated and installed.
- Open Graph tags on the landing page (for social sharing previews).

### 25.5 Operational
- Admin account created, with the platform admin application visible.
- System health table seeded with all subsystems.
- LLM kill switch tested (manual flip + monthly cap auto-flip).
- Retention jobs scheduled but the first run after tournament end.
- Backup restore tested at least once (Oracle handles automatic backups; verify you can actually restore from one).

### 25.6 Dress Rehearsal
- Two days before launch: full click-through on production, with the pre-tournament bracket submission flow.
- One day before launch: verify Pro plan is active on API-Football; confirm first-match fixture details against FIFA's official schedule.
- On launch day: watch the first match go live → finished → scored end-to-end. Every subsystem exercised.
