<!-- Don't delete it -->
<div name="readme-top"></div>

<!-- Organization Logo -->
<div align="center" style="display: flex; align-items: center; justify-content: center; gap: 16px;">
  <img alt="AOSSIE" src="public/aossie-logo.svg" width="175">
</div>

&nbsp;

<!-- Organization/Project Social Handles -->
<p align="center">
<!-- Telegram -->
<a href="https://t.me/StabilityNexus">
<img src="https://img.shields.io/badge/Telegram-black?style=flat&logo=telegram&logoColor=white&logoSize=auto&color=24A1DE" alt="Telegram Badge"/></a>
&nbsp;&nbsp;
<!-- X (formerly Twitter) -->
<a href="https://x.com/aossie_org">
<img src="https://img.shields.io/twitter/follow/aossie_org" alt="X (formerly Twitter) Badge"/></a>
&nbsp;&nbsp;
<!-- Discord -->
<a href="https://discord.gg/hjUhu33uAn">
<img src="https://img.shields.io/discord/1022871757289422898?style=flat&logo=discord&logoColor=white&logoSize=auto&label=Discord&labelColor=5865F2&color=57F287" alt="Discord Badge"/></a>
&nbsp;&nbsp;
<!-- LinkedIn -->
<a href="https://www.linkedin.com/company/aossie/">
  <img src="https://img.shields.io/badge/LinkedIn-black?style=flat&logo=LinkedIn&logoColor=white&logoSize=auto&color=0A66C2" alt="LinkedIn Badge"></a>
&nbsp;&nbsp;
<!-- Youtube -->
<a href="https://www.youtube.com/@AOSSIE-Org">
  <img src="https://img.shields.io/youtube/channel/subscribers/UCKVVLbawY7Gej_3o2WKsoiA?style=flat&logo=youtube&logoColor=white%20&logoSize=auto&labelColor=FF0000&color=FF0000" alt="Youtube Badge"></a>
</p>

---

<div align="center">
<h1>🛡️ AOSSIE Admin Repository (Safe-Settings Policy-as-Code)</h1>
</div>

The **`admin`** repository centrally manages and enforces repository policies, branch protection rules, issue labels, team access permissions, custom properties, rulesets, and environments across all public repositories in the **[AOSSIE](https://github.com/AOSSIE-Org)** organization.

> [!NOTE]
> This repository strictly adheres to the official **[GitHub Safe-Settings](https://github.com/github-community-projects/safe-settings)** specification (`main-enterprise` branch), utilizing standard GitHub Actions workflows ([docs/github-action.md](https://github.com/github-community-projects/safe-settings/blob/main-enterprise/docs/github-action.md)) for policy evaluation, dry-run PR checks, and scheduled drift prevention.

---

## 🚀 Key Features & Capabilities

- **Centralized Policy-as-Code**: Store and manage all organization settings in Git-tracked YAML configuration files under `.github/`.
- **Three-Tier Precedence Hierarchy**: Precedence order `Repository > Sub-Organization > Organization` allows domain-specific customization while enforcing global compliance.
- **Dry-Run PR Validation (Nop Mode)**: When a PR is opened in `admin`, safe-settings runs in dry-run mode to evaluate and validate proposed policy changes before merging.
- **Scheduled Drift Prevention**: Automated GitHub Actions (`.github/workflows/safe-settings-sync.yml`) run on a 4-hour schedule to prevent manual configuration drift in GitHub.
- **Restricted Scope Protection**: Configured via `deployment-settings.yml` (`restrictedRepos`) to safeguard core administrative repositories (`admin`, `.github`, `safe-settings`) from unintended bot operations.
- **Fine-Grained Glob Scoping**: Supports `include` and `exclude` glob patterns for scoping teams, labels, collaborators, and repository lists.
- **External Status Checks Preservation**: Supports `{{EXTERNALLY_DEFINED}}` token under status checks to allow external CI checks configured via the GitHub UI.

---

## 📊 Visual Architecture & System Flows

### Configuration Precedence Hierarchy

```mermaid
graph TD
    A[Organization Defaults<br/>.github/settings.yml] --> B[Sub-Organization Policies<br/>.github/suborgs/*.yml]
    B --> C[Repository Overrides<br/>.github/repos/*.yml]
    
    style A fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#000
    style B fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#000
    style C fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px,color:#000
```

**Precedence Order**: `Repository > Sub-Organization > Organization`

### System Request & Processing Flow

```mermaid
sequenceDiagram
    participant GH as GitHub Organization
    participant SS as Safe-Settings Engine (GHA)
    participant AR as Admin Repo (.github/)
    participant TR as Target Repositories
    
    Note over GH,TR: Event-Driven & Scheduled Processing
    
    GH->>+SS: Trigger Event (Push, Cron, PR, Repo Created)
    SS->>+AR: Fetch Configuration Files
    AR-->>-SS: Return settings.yml, suborgs/*.yml, repos/*.yml
    
    SS->>SS: Merge Hierarchy (Org → Suborg → Repo)
    SS->>SS: Compare Config with Active GitHub Settings
    
    alt Active Sync (Push / Cron / Repo Created)
        SS->>+TR: Apply Settings (Branch Protection, Labels, Teams, Rulesets)
        TR-->>-SS: Confirm Applied Changes
        SS->>GH: Update Check Run (Success)
    else PR Validation (Dry-Run Mode)
        SS->>SS: Run Dry-Run (Nop Mode) & Custom Validators
        SS->>GH: Update PR Check Run + Dry-Run Summary Comment
    end
    
    SS-->>-GH: Processing Complete
```

---

## 📁 Repository Directory Structure

```
admin/
├── deployment-settings.yml           # Defines restrictedRepos (admin, .github, safe-settings)
├── .coderabbit.yaml                  # CodeRabbit AI code review configuration for policy repo
├── .github/
│   ├── settings.yml                  # Organization-wide settings (default branch protection, labels, teams)
│   ├── suborgs/                      # Sub-organization policies (grouped by domain)
│   │   ├── ai-agentic-tools.yml
│   │   ├── blockchain-web3.yml
│   │   ├── web-frontend.yml
│   │   ├── education-learning.yml
│   │   ├── mobile-apps.yml
│   │   └── core-infra.yml
│   ├── repos/                        # Repository-specific override files
│   │   └── admin.yml
│   ├── workflows/
│   │   └── safe-settings-sync.yml    # GitHub Actions workflow for scheduled full-sync
│   └── dependabot.yml                # Dependabot configuration (monitoring github-actions)
├── public/
│   └── aossie-logo.svg
└── README.md                       
```

---

## 🏛️ Sub-Organization Domain Breakdown

All public repositories in **AOSSIE-Org** are grouped into 6 sub-organization policy categories:

| Sub-Org Domain | Description | Key Managed Repositories |
| :--- | :--- | :--- |
| **`ai-agentic-tools`** | AI agents, LLMs, Discord bots, PR analyzers, and automated assistants | `SkillBot`, `PullRequestDashboard`, `Skills`, `OpenVerifiableLLM`, `CodingAgent`, `Gitcord`, `EduAid`, `DebateAI` |
| **`blockchain-web3`** | Decentralized apps, smart contracts, and Web3 dashboards | `Djed-Solidity-WebDashboard`, `IndexedDB-Import-Export` |
| **`web-frontend`** | Web applications, dashboards, frontend components, and social widgets | `OrgExplorer`, `SocialShareButton`, `SupportUsButton`, `Website`, `Resonate-Website`, `PictoPy-Website` |
| **`education-learning`** | Educational tools, Rust ML bindings, knowledge bases | `Social-Street-Smart`, `SciKitLearn-Rust`, `Info`, `LibrEd` |
| **`mobile-apps`** | Android, Flutter, iOS, and cross-platform mobile apps | `CarbonFootprint-Mobile`, `Starcross-Android`, `Agora-Android`, `Agora-iOS`, `Resonate`, `Monumento`, `PictoPy` |
| **`core-infra`** | Core infrastructure, template repositories, blogs, and org tools | `admin`, `Template-Repo`, `.github`, `AOSSIE-Blogs`, `ContributorAutomation`, `Scavenger`, `Skeptik` |

---

## ⚙️ Configuration Specification Guide

### 1. Organization Defaults (`.github/settings.yml`)
Applies globally to all repositories unless overridden by a suborg or repo file:
```yaml
repository:
  has_issues: true
  has_projects: true
  allow_squash_merge: true
  delete_branch_on_merge: true

branches:
  - name: main
    protection:
      required_pull_request_reviews:
        required_approving_review_count: 1
        dismiss_stale_reviews: true
      required_status_checks:
        strict: true
        contexts: []
      required_conversation_resolution: true

labels:
  - name: "bug"
    color: "d73a4a"
  - name: "gsoc"
    color: "f9d0c4"

teams:
  - name: admins
    permission: admin
  - name: maintainers
    permission: push
```

### 2. Sub-Organization Policies (`.github/suborgs/*.yml`)
Defines policies for a collection of repositories specified in `suborgrepos` (supports glob patterns like `test*`):
```yaml
suborgrepos:
  - "SkillBot"
  - "PullRequestDashboard"
  - "Skills"

repository:
  has_issues: true
```

### 3. Repository Overrides (`.github/repos/<repo-name>.yml`)
Applies specific overrides for a single repository (e.g. `admin.yml` enforcing strict 2-reviewer approvals):
```yaml
branches:
  - name: main
    protection:
      required_pull_request_reviews:
        required_approving_review_count: 2
        require_code_owner_reviews: true
      enforce_admins: true
```

### 4. Scope Restrictions (`deployment-settings.yml`)
Controls which repositories safe-settings can manage or exclude:
```yaml
restrictedRepos:
  - admin
  - .github
  - safe-settings
```

---

## 🛠️ Policy Change Workflow

1. Create a branch and modify policy files in `.github/settings.yml`, `.github/suborgs/`, or `.github/repos/`.
2. Open a Pull Request. Safe-settings automatically runs in **dry-run mode** (`nop` mode) to evaluate proposed changes and posts a validation report.
3. Upon approval and merge to `main`, the `.github/workflows/safe-settings-sync.yml` workflow triggers and applies the settings live across target repositories.

---

## 📍 License

This project is licensed under the **GNU General Public License v3.0**. See the [LICENSE](LICENSE) file for details.

© 2026 **AOSSIE**
