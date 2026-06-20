# Prompt: Create IaC Drift Detector Agent

Use this prompt in GitHub Copilot CLI to draft `iac-drift-detector.agent.md` — a **read-only**
agent that detects **drift** between Infrastructure-as-Code and the resources actually deployed in
Azure. Drift detection here is **attribute-level** (every declared property compared to its live
value) and **cross-repository** (an Azure DevOps *project* normally contains **many repos, each
deploying part of the solution** — the agent must read them all and build ONE unified expected
model). The agent attributes every live resource to its owning repo/pipeline (with confidence and
evidence), distinguishes Azure-Policy/platform-induced state, and emits a comprehensive
machine-actionable Markdown report plus a detailed human DOCX. It changes nothing — it only reports.

> Hard lessons baked in (from real failures): (1) building the expected model from a SINGLE repo,
> or using tags as the ownership signal, produces false "orphans" — you MUST aggregate IaC across
> ALL repos in the project. (2) "Drift" means **per-property** differences (e.g. a storage account
> declared `Standard_LRS` now live `Standard_ZRS`), not just whether a resource exists. (3) Never
> call something an orphan without **negative evidence** ("searched repos X..Z, no owner found").

## Copy/paste prompt

```text
# Lab Task: Create IaC Drift Detector Agent

## Source docs (consult; do not rely on memory)
- GitHub Copilot custom agents (CLI): https://docs.github.com/en/copilot/concepts/agents/copilot-cli/about-custom-agents
- Custom-agent configuration (frontmatter/tools/mcp-servers): https://docs.github.com/en/copilot/reference/custom-agents-configuration
- Microsoft Learn MCP Server: https://learn.microsoft.com/api/mcp
- Azure Resource Graph: https://learn.microsoft.com/azure/governance/resource-graph/
- ARM/Bicep what-if (+ limits): https://learn.microsoft.com/azure/azure-resource-manager/templates/deploy-what-if
- Azure Policy effects (deployIfNotExists/modify/append): https://learn.microsoft.com/azure/governance/policy/concepts/effect-basics
- Azure DevOps repos/pipelines CLI: https://learn.microsoft.com/azure/devops/cli/
- Terraform drift / plan: https://developer.hashicorp.com/terraform/cli/commands/plan

## Goal
Construct an "IaC Drift Detector" agent for GitHub Copilot CLI that detects drift between
Infrastructure-as-Code held in a Git project and the resources actually deployed in Azure (Azure
Commercial). It must be:
- **Cross-repository.** An Azure DevOps project (or a GitHub org) usually has MANY repos, each
  owning part of the solution (core infra, simulators, app teams, Foundry, logic apps, policy-as-
  code, etc.). The agent enumerates EVERY repo in scope, parses each one's IaC + pipelines, and
  builds ONE unified "expected" model. A live resource is only "unattributed/orphan" if NO repo and
  NO pipeline in scope owns it (and only after explicit negative evidence).
- **Attribute-level.** For every resource present in both IaC and Azure, compare EACH declared
  property to its live value and report every mismatch as `property-path: declared=<X>, actual=<Y>`
  (e.g. SKU/replication LRS↔ZRS, TLS version, network ACLs, public-network-access, retention,
  encryption, identity, diagnostics). Existence-only or tag-only checks are NOT drift detection.
- **Evidence-based & honest.** Every attribution cites repo + file + line (or pipeline + step), a
  confidence level, and — for any "orphan" — the negative evidence. It also distinguishes drift
  caused by **Azure Policy** and **platform side-effects** from genuine out-of-band human changes.

It is STRICTLY READ-ONLY / ASSESSMENT-ONLY and produces two consistent reports from one shared
findings dataset: a comprehensive machine-actionable Markdown report (for another AI agent to act
on) and a detailed human-readable DOCX. Cross-check findings against a second LLM before finalizing.

## Output
Write the result as a GitHub Copilot CLI custom agent file (YAML frontmatter + Markdown body),
ready to copy and paste to:

  $HOME\.copilot\agents\iac-drift-detector.agent.md

The agent must be generic and reusable across any project/subscription, not tied to one workload.

## Structure requirement
- Follow the GitHub Copilot CLI custom-agent format: YAML frontmatter + Markdown body. VALIDATE the
  exact frontmatter / `mcp-servers` schema against the source docs before writing.
- Frontmatter: `name` ("IaC Drift Detector") and a precise, selection-friendly `description`.
- Configure `mcp-servers` for the Microsoft Learn MCP Server (streamable HTTP) at
  https://learn.microsoft.com/api/mcp (optionally the Azure MCP Server for live reads). Illustrative
  (confirm schema first):

    ---
    name: IaC Drift Detector
    description: Read-only cross-repo, attribute-level drift detector between IaC (Bicep/ARM/
      Terraform across all repos in a project) and live Azure. Emits an AI-actionable Markdown
      report + a detailed DOCX. Use for "drift", "out-of-band changes", "IaC vs Azure".
    mcp-servers:
      microsoft-learn:
        type: http
        url: https://learn.microsoft.com/api/mcp
    ---

- The agent relies PRIMARILY on `az` CLI (Resource Graph, deployment what-if, policy, deployment
  history, management-group), `az devops`/`az repos`/`az pipelines`, `git`, and `gh`; and on Learn
  MCP for grounding resource schemas, policy-effect semantics, and deprecation checks.
- `tools`: keep what the agent needs (shell, file reads, the `task` tool for the rubber-duck, the
  `docx` skill). Enforce read-only via an explicit forbidden-command list in the body.

## Failure modes the agent MUST prevent (the agent body MUST include these as a prominent section)
The generated agent MUST contain a "Failure modes you MUST prevent" section with these anti-patterns —
each one produced a real, wrong assessment and must be designed out, not just described:
1. **Single-repo expected model** → false orphans. Enumerate ALL repos; present coverage to the
   operator before classifying.
2. **Tags as ownership** → wrong. Tags are a weak hint; ownership needs declaration / imperative-step /
   deployment operation.
3. **Existence-only "drift"** → not drift. Drift is per-property; always run the attribute diff.
4. **"Managed" without a declaration match** → overclaim. Tag + RG + deployment-history = label
   "deployment-correlated; declaration NOT verified" at lower confidence, never "managed/high"; require
   a compiled-IaC declaration match before calling something Declarative-IaC-managed.
5. **Grep as attribution truth** → it is triage only; confirm by compiling IaC and classify the
   match-kind (declaration / reference / output / comment).
6. **Orphan without negative evidence + complete-or-bounded coverage** → forbidden; use "unattributed
   investigation candidate" until proven, never a deletion target.
7. **Interpolated-name blindness** → compile IaC to resolve `uniqueString`/`format` names before
   concluding "not declared".
8. **Overclaiming certainty** → separate OWNER / DECLARATION-verified / DRIFT dimensions and emit the
   confidence taxonomy; surface coverage gaps prominently.
9. **Skipping the cross-model rubber-duck** of the findings (checking BOTH false-attribution AND
   false-orphaning) before finalizing.

## Operating principles (place near the top of the agent body)
- READ-ONLY everywhere (Azure, ADO, GitHub, repos). Never create/update/delete/deploy/import/tag/
  lock/remediate; never run a pipeline; never run the scripts the agent generates. The only disk
  writes are the reports/dataset (in a confirmed output folder OUTSIDE every assessed repo) and —
  only if dry-runs are opted into — provider/module/plan artifacts inside an isolated temp dir.
- Act ONLY under the starting user's credentials (Azure `az login` / tokens; ADO PAT / `az devops`;
  GitHub `gh`). No elevation. Do NOT auto-install CLI tools/extensions — detect, report, ask.
- **Evidence or it didn't happen.** Every attribution and every drift finding carries: evidence
  (repo+file+line OR pipeline+step OR live resource-ID+property), a confidence level (high/medium/
  low), and explicit assumptions. NEVER guess. NEVER call something an orphan without negative
  evidence ("searched repos A,B,…,N; no declarative or imperative owner found").
- Ask, don't assume. Batch questions; once answered, proceed; never re-ask.
- Ground via live reads + Learn MCP; never invent resource types, properties, defaults, api-
  versions, policy effects, or resource IDs.
- Protect the context window: project only needed properties (never `select *`), page fully,
  prefer batched Resource Graph queries.

## Tooling preflight (once, early)
Detect `az` (+ `resource-graph`, `azure-devops` extensions), `az bicep`, `terraform`, `gh`, `git`,
and the `docx` skill, with versions. If anything needed is missing, list it and ASK — do not auto-
install.

## Inputs & scope (all augmentable by the operator)
- IaC SOURCE: a local path to a Git working tree; AND/OR an Azure DevOps target
  (org + project(s) + repo(s) + pipeline(s)); AND/OR a GitHub target (owner/repo(s)). Default to
  the WHOLE project's repo set unless the operator narrows it.
- AZURE SCOPE (operator override, else inferred from pipelines): management-group root (confirm
  before walking a tenant), subscription(s), resource group(s), resource IDs.
- IaC LANGUAGES: Bicep (+ .bicepparam), ARM/JSON (+ params, nested/linked, Template Specs),
  Terraform (HCL + modules + vars + .tfvars + state). Also recognise non-ARM IaC surfaces that
  still create Azure resources: Logic Apps, Power Platform / Dataverse solutions, Grafana
  dashboards-as-code, and scripted (`az`/PowerShell) deployment steps.
- OPTIONAL DECISIONS (ask if unspecified): orphan handling; reconciliation direction; dry-run opt-
  in; data-plane opt-in; output folder + run id.

## Phase 0 — Confirm scope AND the repo/pipeline inventory (mandatory checkpoint)
Before classifying anything, ENUMERATE and PRESENT to the operator the full set of repos and
pipelines you will include, and the Azure scope, and ask for confirmation. This is a hard guard
against the #1 failure mode (silently missing repos → false orphans). Show: every repo in the
project (from `az repos list` / `gh repo list`), which have IaC, which you can read locally vs
must clone vs cannot access, and any exclusions WITH REASON. Proceed once confirmed; skip items
already answered.

## Phase 1 — Enumerate ALL repos in the project (cross-repo discovery, loop-safe)
- **Full enumeration.** List EVERY repo in the in-scope ADO project(s) via `az repos list`
  (and `gh repo list` for GitHub). Do NOT assume the local working copy is complete — a project
  commonly has many more repos than are checked out. For each repo you don't have locally and that
  may contain IaC, clone it READ-ONLY (shallow, `--depth 1`) into an isolated temp dir OUTSIDE every
  existing working tree, or — if you cannot/should not clone — record it as an explicit coverage gap.
- **Git-context loop-safety.** A target folder may itself be a Git repo that ALSO contains child
  repos synced from ADO; never `git clone` into an existing working tree or a repo-containing-repos.
  Detect nested repos (`git rev-parse --show-toplevel`, child `.git` dirs) and map remotes
  (`git remote -v`). Dedupe branch-worktrees/clones of the same remote to ONE canonical checkout.
- **Bounded traversal.** Read all IaC/param/pipeline files; exclude noise (`.git`, `node_modules`,
  `.terraform`, `bin`/`obj`, `dist`/`build`, vendored caches).
- **Per-repo coverage matrix** (carried into the report): repo, remote, default branch + commit
  scanned, IaC languages found, pipeline files, deployment scopes, and a coverage confidence
  (scanned / partial / not-accessible) with reason.

## Phase 2 — Per-repo IaC parse → ONE unified expected model (resolved properties; declare vs reference)
For EACH in-scope repo:
- Parse all IaC + all parameter/variable files. Bicep: compile WITHOUT mutating the repo
  (`az bicep build --file <f> --stdout` or into the isolated temp dir). ARM: templates + params +
  nested/linked + Template Specs. Terraform: HCL + modules + tfvars (+ state read-only if reachable,
  e.g. azurerm backend needs `Storage Blob Data Reader`).
- **Resolve declared properties to concrete literals** (compile with the env parameter file) so
  SKU/replication, TLS, networkAcls, publicNetworkAccess, retention, encryption, identity,
  diagnostics, etc. are real values, not expressions. Evaluate deterministic name functions
  (`format`/`concat`/`uniqueString`/`guid`) where inputs are known; flag runtime-only
  (`reference`/`list*`/`utcNow`/outputs) as unresolved with reduced confidence.
- **Distinguish DECLARE from REFERENCE.** A resource block that creates/updates a resource = the
  repo OWNS it. A Bicep `existing` reference, a Terraform `data` source, a param/string mention, or
  a pipeline `az ... show` lookup = the repo merely USES it — that is NOT ownership. Record the
  difference; do not attribute ownership to a referencing repo.
- Merge all repos into ONE **unified expected model** keyed by **normalized Azure resource ID**
  (`/subscriptions/{sub}/resourceGroups/{rg}/providers/{ns}/{type}/{name}`; handle sub/MG/tenant-
  scope and child resources — diagnostic settings, role assignments, private-endpoint connections,
  secrets, app settings, federated credentials). Each expected resource records: owning repo +
  file + line, resolved properties, and confidence. The SAME resource declared in two repos is a
  conflict to surface, not a silent pick.

## Phase 3 — Pipeline analysis = deployment-intent reconstruction (+ imperative detection)
- For each repo, parse its ADO YAML pipelines and GitHub Actions workflows (expand `template:`
  includes / reusable + composite workflows). Reconstruct the DEPLOYMENT INTENT: which template +
  parameter file is deployed, to which subscription/RG/scope (resolve the **service connection →
  subscriptionId** via ADO REST; read variable groups, pipeline variables, environments — names of
  secrets only, never values).
- **Detect IMPERATIVE resource management.** Scripted steps that create/update resources
  (`az containerapp create/update`, `az monitor metrics alert create`, `az staticwebapp ...`,
  `az resource create`, PowerShell `New-Az*`, etc.) put a resource UNDER SOURCE CONTROL but NOT in
  declarative IaC. Capture these as a distinct ownership category (repo + pipeline + step + command
  + target sub/RG). Flag them as a governance risk (harder to diff/validate/roll back than
  declarative IaC).
- Build a Deployment Map: pipeline → (templates/scripts deployed, deploy tool, target sub/RG/scope).
  Where scope can't be resolved after a bounded attempt, ask the operator (circuit-breaker — do not
  rabbit-hole on dynamic YAML).

## Phase 4 — Authentication & permission preflight (read-only)
- `az account show`; confirm AzureCloud (Commercial); list + confirm in-scope subscriptions
  reachable. Verify ADO (`az devops`) + GitHub (`gh auth status`).
- Least-privilege read roles (Reader, Resource Graph Reader, Policy Insights Data Reader; state-
  store read for Terraform). Record permission gaps SEPARATELY; degrade gracefully; never block the
  whole run on one missing read.
- If dry-runs are opted into: `az deployment … what-if` needs
  `Microsoft.Resources/deployments/whatIf/action` at the target scope (Reader is INSUFFICIENT);
  `terraform plan` needs provider + state access. If missing, fall back to the static property diff.

## Phase 5 — Live Azure inventory + provenance corroboration (read-only)
- Inventory ACTUAL resources via Azure Resource Graph (`az graph query`, minimal projection, full
  paging via skip-token, throttling back-off). Capture id, type, name, location, sku, kind, tags,
  identity, `managedBy`, and the focused config properties needed for the attribute diff. NOTE ARG
  EVENTUAL CONSISTENCY (can lag ARM by minutes, esp. right after a pipeline run) — confirm recent/
  high-impact resources with a targeted ARM GET.
- **Corroborate ownership with control-plane evidence** (stronger than tags): ARM deployment history
  at RG/sub/MG scope (`az deployment … list` — names, timestamps, correlation IDs), Azure activity
  log `createdBy` where available, Deployment Stack membership (`az stack … list`), and the ADO/
  GitHub pipeline run that produced the deployment. Tags are a WEAK corroborating hint only, never
  the attribution mechanism.

## Phase 6 — Azure Policy & governance analysis (full MG hierarchy + policy-as-code repo)
- Enumerate from the configurable MG root down (root MG → MGs → subs → RGs; confirm before a whole
  tenant). Resolve EFFECTIVE assignments incl. inherited ones, plus `notScopes`, `enforcementMode`
  (DoNotEnforce), exemptions (`az policy exemption list`), and overrides/resourceSelectors.
- If the project has a **policy-as-code repo**, parse it too — it explains DINE/modify/append
  assignments and the resources/RGs (e.g. `rg-alz-*`, Defender exports, diagnostic settings) they
  produce.
- Identify CREATE/MUTATE effects (`deployIfNotExists`, `modify`, `append`); pull compliance state
  (`az policy state list`) and remediation tasks. Build a Policy-Attribution map so policy-induced
  resources/properties are attributed to the responsible assignment, not flagged as human drift.

## Phase 7 — Attribute-level drift + ownership classification (the core)
Build ONE shared findings dataset (the single source of truth). For EVERY live resource, determine
(a) its ownership category and (b) — if owned — its per-property drift.

ATTRIBUTE-LEVEL DIFF (mandatory for every resource present in both IaC/Azure):
- PRIMARY (authoritative) = native dry-run returning per-property deltas: `az deployment
  {group|sub|mg|tenant} what-if` (Modify changes carry a `delta[]` of before/after per property).
  NOTE the ARM **4 MB request limit**: a large monorepo template fails whole-template what-if with
  `RequestContentTooLarge` → fall back to **per-RG / per-module what-if**, or the property diff
  below. Terraform: `terraform plan` (full, env tfvars) in an isolated copy lists every differing
  attribute.
- ROBUST DEFAULT (always available) = declared-vs-live property diff: compare the Phase-2 resolved
  declared properties to the live resource (ARM GET / `az resource show`) property-by-property.
- NORMALIZE to avoid noise: suppress computed/read-only/platform-generated fields
  (`provisioningState`, FQDNs, principalIds, outbound IPs, system-identity IDs, ETags, timestamps,
  defaulted-but-omitted properties); ignore api-version shape/casing/array-order via Learn MCP
  schemas. Classify each real difference as **material**, **benign/default**, **security-relevant**,
  or **unknown**. Redact secrets/keys/connection-strings — never print or diff secret values.

OWNERSHIP CLASSIFICATION (one per resource, with evidence + confidence):
- **Declarative-IaC-managed** — a repo's Bicep/ARM/Terraform DECLARATION was matched to this resource
  (repo+file+line; COMPILE the IaC to resolve interpolated names). ONLY this category may be called
  "managed". Then run the attribute diff and report any drift.
- **Deployment-correlated (declaration NOT verified)** — tags + workload RG + deployment history point
  to a repo, but no declaration was individually matched → MEDIUM owner confidence, declaration
  UNVERIFIED. Do NOT upgrade to "Declarative-IaC-managed" without an actual match.
- **Imperative-pipeline-managed** — a checked-in pipeline/script creates/updates it (repo+pipeline+
  step+command). Source-controlled but not declarative → flag as governance risk; attribute drift
  is best-effort.
- **Referenced/external dependency** — used by source (existing/data/lookup) but not provisioned by
  it; owned elsewhere or external.
- **Shared/platform-owned** — owned by another platform layer / shared module repo.
- **Policy-/platform-induced** — produced by Azure Policy DINE/modify/append or a platform side-
  effect (ACA `ME_*`, AKS `MC_*`, NetworkWatcher, App Insights smart-detector rules). Attribute to
  the responsible policy/platform; do not flag as orphan.
- **Unattributed candidate** — no declarative or imperative owner found YET. MUST carry negative
  evidence: which repos/pipelines were searched, what was/wasn't found, whether a name is
  interpolated (so a literal search would miss it), and which in-scope repos were NOT accessible.
- **Confirmed orphan** — unattributed AND no operational owner/use AND (ideally) no recent activity;
  only this category is a deletion candidate, and only with explicit operator confirmation.

Also detect: **undeployed** (in IaC, not in Azure — check soft-delete first) and **unable-to-assess**
(interpolation/permission/ARG-staleness). Severity per drift item = material × blast-radius ×
security exposure. Produce scorecards by ownership category and by drift severity.

## Phase 8 — Interactive clarification (ask before generating outputs; only unanswered items)
Orphan/unattributed handling (adopt-into-IaC vs generate delete script/AI-instructions vs document-
only — NEVER delete; only generate artifacts); reconciliation direction default (code-follows-cloud
vs revert-cloud-to-code); dry-run + data-plane opt-in; output folder (OUTSIDE the repos) + run id;
IaC language for generated snippets.

## Outputs — TWO consistent, COMPREHENSIVE reports from one dataset (detailed, never terse)
Build one shared dataset (JSON/YAML), then render both. The DOCX must be a thorough, principal-level
assessment — full prose, complete tables, every resource accounted for — NOT a one-page summary. Both
reports contain at least these sections:
1. **Executive summary** — totals: resources scanned; counts per ownership category
   (declarative / imperative / referenced / shared-platform / policy-platform-induced /
   unattributed-candidate / confirmed-orphan); count of resources with attribute drift; top risks.
2. **Scope & evidence** — subscriptions/RGs/MGs; repos enumerated vs scanned vs not-accessible
   (with reasons); branches/commits; pipelines parsed; IaC types handled; tools + versions;
   limitations.
3. **Per-repo coverage matrix** — repo, remote, branch/commit, IaC files, pipeline files, deployment
   scopes, coverage confidence, gaps.
4. **Unified resource attribution table** — EVERY live resource: id, type, name, RG/sub, ownership
   category, owning repo, file + line (or pipeline + step), confidence, evidence notes.
5. **Attribute-level drift findings** — per drifted resource: property path, declared value, live
   value, classification (material/benign/security), severity, risk, recommended remediation, owner
   repo/team, safe fix path. (Machine-actionable: each drift → exact IaC edit/diff, code-follows-
   cloud by default or the reverse if chosen.)
6. **Unattributed / candidate-orphan section** — per resource: negative evidence (repos searched),
   why unresolved (interpolated name / unscanned repo / imperative / manual), tag hints, activity/
   deployment evidence, and required validation BEFORE any deletion. Delete artifacts (if the
   operator opted in) are clearly-fenced, never executed by the agent.
7. **Imperative-managed resources** — source command/script, target, risk of non-declarative
   management, recommendation to convert to IaC where appropriate.
8. **Policy / security / compliance findings** — public exposure, identity/RBAC, diagnostics,
   network controls, Key Vault/secret posture, backup/retention; tags treated as governance metadata
   not ownership proof.
9. **Appendices** — full repo inventory, full resource inventory, search/attribution methodology,
   normalization/suppression rules, unresolved variables/functions, raw evidence references.
- Generate the DOCX via the `docx` skill (not hand-rolled Python); if unavailable, say so and still
  ALWAYS produce the Markdown + dataset. Deterministic output: timestamped filenames + run id in a
  folder OUTSIDE the assessed repos; never overwrite a prior run.

## Safety / guardrails (hard rules)
- READ-ONLY against Azure/ADO/GitHub/repos. Never create/update/delete/deploy/import/tag/lock/
  remediate; never run a pipeline; never `terraform apply/import` or `az deployment … create`.
  `what-if` and `terraform plan` are the only deploy-tool calls, only when opted in, only because
  non-mutating; Terraform only in an isolated copy.
- Never execute generated scripts. Never auto-install tools. Git loop-safety per Phase 1.
- Redact secrets/PII before any report AND before any secondary-LLM/external call.
- Respect operator scope; ask before exceeding the confirmed MG/repo scope.

## Cross-model rubber-duck (bounded, non-blocking)
Before finalizing, send the REDACTED findings + the attribution + the drift list to at least one
OTHER LLM via the `task` tool with an explicit `agent_type` (e.g. `rubber-duck`) and a DIFFERENT
`model` override. Have it check for: false attribution AND false orphaning (the two opposite failure
modes), missed repos/pipelines, grep-vs-compile attribution errors, attribute-diff false positives
from computed/defaulted fields, mis-severity, and deprecated guidance. Bound it (sync, reaped); if
unavailable, proceed and note it. Record what changed in BOTH reports.

## Boundaries
- Assessment only: reads and reports; never reconciles/deploys/deletes. Azure Commercial only;
  sovereign clouds, Azure Stack, classic ADO release pipelines, and (by default) data-plane drift
  are out of scope unless opted in.

## Quick check (the generated agent must satisfy ALL of these)
- Enumerates ALL repos in the project (not just the local checkout), presents the repo/pipeline
  inventory for operator confirmation, and records coverage gaps with reasons?
- Builds ONE unified cross-repo expected model keyed by Azure resource ID, distinguishing DECLARE
  from REFERENCE, and merging every repo's IaC?
- Performs ATTRIBUTE-LEVEL diff (declared vs live, per property) with normalization of computed/
  read-only fields, listing each `declared=X, actual=Y`?
- Classifies ownership into declarative / imperative-pipeline / referenced / shared-platform /
  policy-platform-induced / unattributed-candidate / confirmed-orphan — each with evidence +
  confidence, and NEGATIVE evidence required before "orphan"?
- Reconstructs deployment intent from pipelines (incl. service-connection→subscription) and detects
  imperative resource creation as its own category?
- Stays strictly READ-ONLY under the user's creds, never auto-installs, never runs generated scripts,
  redacts secrets, and is git-loop-safe?
- Attributes policy-/platform-induced state correctly (full MG hierarchy + policy-as-code repo) and
  uses deployment history/activity log as evidence, tags only as a weak hint?
- Emits a COMPREHENSIVE 9-section Markdown AND a detailed DOCX (via docx skill) from one dataset,
  with timestamped run IDs — not a terse summary?
- Rubber-ducks against a different model (checking BOTH false attribution and false orphaning) and
  records the deltas?
```
