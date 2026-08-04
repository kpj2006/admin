# Safe-Settings Adoption — Admin Review Document

**Purpose:** This document explains what we're proposing to adopt (`safe-settings`, policy-as-code for the AOSSIE-Org repos), how it actually works, what it needs from an org admin to go live, and every risk/decision point that needs sign-off before we merge this to `main` and let it start applying settings org-wide.

---

## 1. What this is, in plain terms

Today, changing branch protection, labels, or team access on an AOSSIE repo means clicking through GitHub's UI by hand, one repo at a time, with no record of who changed what or why.

**Safe-settings replaces that with policy-as-code:** we write the desired settings as YAML files in this `admin` repo, and an automated job reads those files and pushes the settings to every real AOSSIE repo. Every change becomes a reviewable Git commit instead of an invisible UI click.

### How it runs: GitHub Action + GitHub App (two separate pieces)

**The Action is the scheduler/engine — it has no permissions of its own.**
`.github/workflows/safe-settings-sync.yml` runs on a schedule (every 4 hours), on manual trigger, or when someone pushes a policy change to `main`. It checks out the `safe-settings` tool, installs it, and runs a full sync. The catch: every GitHub Actions workflow gets a free built-in token, but that token is locked to the one repo it runs in (`admin`) — it **cannot** touch any other AOSSIE repo.

**The GitHub App is the identity/credential that actually does the work.**
Because the Action's own token can't reach other repos, we need a separate "robot identity" — a GitHub App — installed org-wide with admin-level permissions on repo settings. The Action borrows this App's credentials (App ID + private key) each run to actually read/write settings across every repo.

Analogy: the Action is an employee who shows up on schedule to do paperwork; the App is the ID badge that gets that employee through the door of every repo (building) in the org, because we installed it with "All repos in the org."

**Why an App instead of a personal access token (PAT):**
| | GitHub App | Personal Access Token |
|---|---|---|
| Tied to | A bot identity ("safe-settings") | A specific human's GitHub account |
| If that person leaves the org | Nothing breaks | Token dies, sync silently stops |
| Audit log shows | "safe-settings bot did this" | A human's name — misleading, since they didn't act |
| Scope | Exactly what's granted | Inherits that person's full account access |

### The full flow, step by step
1. We write desired settings in `.github/settings.yml` (org-wide defaults), `.github/suborgs/*.yml` (per domain/team of repos), and `.github/repos/*.yml` (per-repo overrides).
2. Precedence: **Repo file > Sub-org file > Org default** — a more specific file always wins.
3. On trigger, the Action authenticates as the GitHub App, merges the three layers of config per repo, and diffs it against GitHub's live state.
4. Anything that differs gets force-corrected back to match the YAML. Anything *not* mentioned in any YAML file is left untouched.
5. `deployment-settings.yml`'s `restrictedRepos` list (currently `admin`, `.github`, `safe-settings`) is fully excluded from all of this — those repos are never touched.

---

## 2. What the admin needs to do to actually turn this on

None of this works yet — the config files exist, but there is no GitHub App installed, so the workflow currently has nothing to authenticate with.

- [ ] **Create a GitHub App** under the AOSSIE-Org (not a personal account), at `github.com/settings/apps/new` (or via the org's app settings).
  - Homepage URL: this `admin` repo's URL
  - Webhook: **turn off** ("Active" unchecked) — we're only using scheduled/manual sync, not a live server, so nothing needs to receive webhooks.
- [ ] **Grant these permissions** to the App:
  - Repository: Administration (R&W), Contents (R&W), Issues (R&W), Pull requests (R&W), Checks (R&W), Custom properties (R&W), Actions (Read-only), Metadata (Read-only)
  - Organization: Administration (R&W), Members (R&W), Custom properties (Admin)
- [ ] **Install the App** on the org, choosing **"All repos in the Org"** (this is what makes it cover every current and future repo).
- [ ] **Collect and store as repo secrets/variables** on this `admin` repo (Settings → Secrets and variables → Actions):
  - Secret `SAFE_SETTINGS_PRIVATE_KEY` = full contents of the App's generated private key (`.pem`)
  - Variable `SAFE_SETTINGS_APP_ID` = the App's ID
  - Variable `SAFE_SETTINGS_GH_ORG` = `AOSSIE-Org` (workflow already defaults to this if unset)
- [ ] **Decide who can merge to `admin`'s `main` branch.** This is the real access-control boundary going forward — merging here is equivalent to an org-admin action across every repo. Recommend requiring admin/owner review on this repo specifically (already partially enforced via `repos/admin.yml`: 2 approvals + code owner review + admin enforcement).

---

## 3. Plan/feature check — already verified, no action needed

- **GitHub Team plan** confirmed via API (`plan.name: "team"`, 95 seats).
- **GitHub Actions** — fine on any plan; public repos get unlimited free Actions minutes regardless of plan.
- **GitHub Apps** — not gated by plan at all.
- **Branch protection** — included in Team for private repos, free for public repos either way.
- **Custom properties** — confirmed working via a live read-only API call (`orgs/AOSSIE-Org/properties/schema` returned successfully).
- **Organization rulesets** — could **not** be confirmed from a member-level token (API returned a 404 that's ambiguous between "not supported on this plan" vs "needs admin:org scope to view"). **Ask an org owner to check `github.com/orgs/AOSSIE-Org/settings/rules` directly** — if they can create a ruleset there, it's supported. Low risk either way: if unsupported, only the `rulesets:` section of the config fails on sync; everything else (branch protection, labels, teams, collaborators) still applies fine.

---

## 4. Critical: the current `settings.yml` is NOT ready to merge

This is the most important part of this review. The org-wide default file (`.github/settings.yml`) is currently the **unedited example file from the safe-settings project's own documentation** — not a real AOSSIE policy. If merged as-is, the next sync would push this to **every AOSSIE repo** not already overridden by a suborg/repo file. Concretely:

| Field | Current value | Why it's dangerous |
|---|---|---|
| `collaborators` | `Zahnentferner` (admin), `beetlejuice` (pull), `thor` (push) | Literal example usernames from the docs. If these resolve to real GitHub accounts, this **grants unknown strangers admin/push access to every AOSSIE repo**. This is the single highest-risk line in the file. |
| `teams` | `core` (admin), `docss` (push), `docs` (pull), `globalteam` (push) | None of these teams exist in AOSSIE-Org. Will either error every sync or auto-create junk teams with these names. |
| `private` / `visibility` | `false` / `public` forced org-wide | Would flip any intentionally-private repo to public on the next sync. |
| `description` / `homepage` | `"description of the repo"` / `https://example.github.io/` | Overwrites every real repo's actual description/homepage with placeholder text. |
| `license_template` | `GPL-3.0` forced org-wide | Not appropriate uniformly across mobile/Solidity/docs repos; there's no per-suborg override for this today. |
| `archived` | `false` forced org-wide | Would **unarchive** any repo the org has intentionally archived. |
| `rulesets` (`Template`) | Fake actor IDs (`actor_id: number`), fake `repository_id: 123456`, targets only repos named `test*` | Will fail validation/API calls, and even if it worked, wouldn't apply to any real repo. |
| `labels` | Two entries (`first-timers-only`, `new-label`) both set `oldname: Help Wanted` | Duplicate rename target — leftover from the sample, not intentional. |
| `topics` / `custom_properties` / `autolinks` / `milestones` | `new-topic`/`another-topic`, `test: test`, `MYLINK-` → `mywebsite.com`, `milestone-title` | All junk placeholder values that would get pushed to every repo. |

**Recommendation:** do not merge `settings.yml` in its current form. It needs to be rewritten with AOSSIE's real teams/policies before this goes live (see §5).

---

## 5. Real data needed to write the actual config (already pulled from the org)

Org-wide teams that likely belong in `settings.yml`'s `teams:` block (replacing the 4 fake ones):

| Slug | Display name | Suggested scope |
|---|---|---|
| `admins` | ! Admins | `admin`, org-wide |
| `contributors` | ! Contributors | `push`, org-wide |
| `mentors` | ! Mentors | `push`/`maintain`, org-wide |
| `triagetaskforce2026` | ! TriageTaskForce2026 | `triage`, org-wide |

Domain-scoped teams that belong in the matching `suborgs/*.yml` file instead of the org default:
- `ai-tooling` → `ai-agentic-tools.yml`
- `blockchain` → `blockchain.yml`
- `repotooling` → `core-infra.yml` (likely governs `admin` itself)
- `landingpages` → `web-frontend.yml`

Plus ~35 per-project teams (`skillbot`, `pictopy`, `agora`, `resonate`, etc.) that map to individual repos and belong in `repos/<name>.yml`, not the org default.

**Decisions needed from the admin:**
- [ ] Confirm the permission level (`pull`/`triage`/`push`/`maintain`/`admin`) for each org-wide team above.
- [ ] Confirm whether any **individual** (non-team) collaborators are actually needed — recommendation is to avoid listing individuals and rely on team membership instead, since team-based access doesn't need a config change every time someone joins/leaves.
- [ ] Confirm the license/gitignore/visibility defaults actually wanted org-wide (or whether these should be removed from the org default and only set per-suborg/repo).
- [ ] Confirm whether the `rulesets` block should be dropped entirely for now or replaced with a real ruleset once org-ruleset support is confirmed.

---

## 6. Behavior the admin should understand before approving

- **Anything written in the YAML gets force-corrected back on every sync (up to every 4 hours), even if changed manually in the GitHub UI.** This is intentional (drift prevention) but means manual one-off changes to anything declared in these files will not stick.
- **Anything NOT written in the YAML is left alone.** Safe-settings has no opinion about teams/settings it was never told about.
- **Team *membership*** (who is inside e.g. the `contributors` team) **is never managed by safe-settings** — only *which teams have access to which repos*. Admins can freely add/remove people from teams without safe-settings interfering.
- **There is currently no automatic PR dry-run.** The workflow triggers on `schedule`, `workflow_dispatch`, and `push` to `main` only — there's no `pull_request` trigger, so a PR changing `settings.yml` is not automatically validated before merge, despite this being described as a goal. This is a gap we should either close (add a dry-run/nop-mode check on PRs) or acknowledge as a manual-review-only process for now.
- **The GitHub App's private key is a high-value secret.** Anyone with write access to this repo's Actions secrets effectively has org-admin-equivalent power. Treat merges to `main` here like production deploys.

---

## 7. Pros / Cons summary

**Pros**
- Centralized, Git-tracked, reviewable policy for every repo instead of manual per-repo clicking.
- Self-healing — manual drift in the UI gets corrected automatically.
- No hosting cost — runs entirely on GitHub-hosted Action runners.
- Scales automatically to new repos (App is installed org-wide).

**Cons**
- A bad merge to `main` has org-wide blast radius (see §4 for a live example).
- Up to 4-hour delay before catching drift/new repos — not real-time.
- No PR-time preview of actual effects today, only YAML review.
- Central, high-privilege credential (GitHub App private key) that needs careful secret handling.

---

## 8. Ask of the admin (summary checklist)

- [ ] Review and approve the GitHub App creation/permissions/installation plan in §2.
- [ ] Verify organization ruleset support directly in org settings (§3).
- [ ] Review and approve/adjust the real team-permission mapping in §5.
- [ ] Approve rewriting `settings.yml` before merge (do **not** approve merging it as-is — see §4).
- [ ] Decide on branch protection for who can merge to `admin`'s `main`.
- [ ] Decide whether to add PR dry-run validation now or treat this as a later improvement.
