# ManageEIPs — Automated Cleanup of Unassociated Elastic IPs (EIPs)

## Problem
Unassociated Elastic IPs incur hourly AWS charges and are easy to forget.

## Solution
This project deploys an **AWS Lambda** function that periodically scans Elastic IPs in a Region and releases **only** those that are:
- **unassociated** (not attached to an instance/network interface), and
- **in-scope** via tags (**ManagedBy=ManageEIPs**), and
- **not protected** (**Protection≠DoNotRelease**).

Scheduling is handled by **Amazon EventBridge** (cron rule).

## Safety model (Dry-Run first)
The Lambda supports a **Dry-Run / safety mode** where it evaluates and logs actions but makes **no changes**.

Recommended workflow:
1) Deploy → run Dry-Run → review logs/metrics  
2) Enable real execution (`dry_run=false`) only after validation

## Architecture (single Region)
- **EventBridge rule** (scheduled) → invokes Lambda
- **Lambda** lists EIPs in the Region
- Applies **tag-based scope + protection checks**
- Releases qualifying unused EIPs
- Emits **structured CloudWatch logs (JSON)** and optional metrics (FinOps angle)

## Scheduling (recommended)
For normal operation and cost control, a **monthly** schedule is sufficient (e.g. **02:00 UTC on the 25th**).

During testing, you can temporarily set a near-future cron time, validate behavior, then revert to the monthly schedule.

## Multi-Region (design-ready, optional)
The solution is **Region-agnostic**:
- Deploy the same stack independently in Region A / Region B
- No code changes required
- Each Region has its own EventBridge schedule and operates only on local EIPs
- IAM is global and can be reused; operational visibility and costs remain per-Region

## Operational standards enforced in this repo
This repository intentionally follows strict “portfolio-grade” operational discipline:

- **CLI-first (AWS CLI + jq)**: reproducible commands, minimal console dependency
- **No hardcoded IDs**: resource IDs are discovered dynamically and stored in variables
- **Strict JSONL output**: prefer `jq -c` with **one JSON object per line**
- **Reusable jq helpers**: shared filters to keep output contracts consistent
- **Consistent tagging on everything** (minimum set):
  - `Name`, `Project`, `Component`, `Environment`, `Owner`, `ManagedBy`, `CostCenter`
- **AWS resource name ≠ Name tag** where applicable
- **Tag-based scope control**:
  - `ManagedBy=ManageEIPs` → resource is managed by this automation
  - `Protection=DoNotRelease` → explicitly protected from release

## Repository contents
**Tracked (published) files**
- `ManageEIPs_Automation.md` — end-to-end build + verification (includes **Section 13: GitHub**)
- `lambda_function.py` — Lambda function source
- `manage-eips-policy.json` — least-privilege IAM policy documents used in the build
- `manage-eips-trust.json` — IAM trust policy document for the Lambda execution role
- `*.jq` — reusable jq filters (`flatten_tags.jq`, `tag_helpers.jq`, etc.)

**Not tracked (intentionally excluded via `.gitignore`)**
- secrets/keys (`*.pem`, `*.key`, `.env*`, etc.)
- build artifacts (`*.zip`)
- local scratch outputs (`response*.json`, `event.json`)
- IDE folders (`.vs/`, `.vscode/`, `.idea/`)

## Publishing to GitHub (summary)
Full, click-by-click + CLI instructions are in `ManageEIPs_Automation.md` (**Section 13**), including:
- local repo initialization, `.gitignore`, and first commit
- GitHub repo creation + HTTPS remote
- PAT-based authentication for `git push` (GitHub no longer accepts account passwords for HTTPS git operations)
- post-push verification and “no-secrets” sanity checks (`git ls-files` / `git grep` patterns)

## Release / versioning
The documentation is actively being refined. Until it is final:
- consider an annotated **pre-release tag** such as `v0.1.0` and mark it as a **pre-release** on GitHub
- create the final `v1.0.0` tag only when both `ManageEIPs_Automation.md` and this README are final

## Documentation
- Full step-by-step implementation and verification (including strict output conventions and GitHub publishing):  
  👉 `ManageEIPs_Automation.md`

## Status
✔ Working  
✔ Tested (manual + scheduled)  
✔ Safe-by-design via Dry-Run + tag-based scope
