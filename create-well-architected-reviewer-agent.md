# Prompt: Create Well-Architected Reviewer Agent

Use this prompt in GitHub Copilot CLI to draft `well-architected-reviewer.agent.md` — an
agent that assesses Azure Infrastructure-as-Code and/or live resources against the five
Azure Well-Architected Framework pillars, grounds every finding in the docs via the
Microsoft Learn MCP Server, and emits a machine-actionable Markdown remediation backlog
plus a human-readable DOCX report, cross-checked against a second LLM.

## Copy/paste prompt

```text
# Lab Task: Create Well-Architected Reviewer Agent

## Source docs
- GitHub Copilot custom agents: https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-custom-agents
- Microsoft Learn MCP Server: https://learn.microsoft.com/api/mcp
- Azure Well-Architected Framework: https://learn.microsoft.com/en-us/azure/well-architected/

## Goal
Construct a "Well-Architected Reviewer" agent for GitHub Copilot CLI that audits Azure
workloads — supplied as Infrastructure-as-Code and/or read live from Azure — against all
five pillars of the Azure Well-Architected Framework (WAF), and produces two consistent
reports: a machine-actionable Markdown remediation backlog and a human-readable DOCX.
Ground every recommendation in the live WAF docs via the Learn MCP Server, and cross-check
findings against a second LLM before finalizing.

## Output
Write as a GitHub Copilot custom agent file, using YAML frontmatter and Markdown
instructions, ready to copy and paste to:

$HOME\.copilot\agents\well-architected-reviewer.agent.md

## Structure requirement
- Follow GitHub Copilot custom agent structure (YAML frontmatter + Markdown instructions).
- Include a clear `name` ("Well-Architected Reviewer") and `description`.
- Include `mcp-servers` configuration for the Learn MCP Server at
  https://learn.microsoft.com/api/mcp. Optionally also configure the Azure MCP Server for
  live resource reads.
- Keep MCP usage scoped to the reviewer's needs.

## The agent you write must — Inputs & scope
- Accept Infrastructure-as-Code: Bicep (+ .bicepparam), ARM/JSON templates (+ parameter
  files), Terraform (incl. modules, variables, .tfvars, plan, state) under a given path;
  AND/OR live Azure resources via Azure Resource Graph, ARM REST, Azure CLI, Az PowerShell,
  Microsoft Graph (or the Azure MCP Server).
- Work with either or both; auto-detect mode (IaC-only / live-only / combined).
- Take an explicit scope (subscription / resource group / specific resource IDs / IaC
  path). Default live reads to a single resource group; project only the properties needed
  (never `select *`) to protect the context window.
- Take workload context (business criticality, environment, RTO/RPO, target availability,
  regions, data sensitivity, regulatory constraints, owner). Where unknown, state
  assumptions explicitly and proceed — WAF is workload-oriented.

## Pre-flight & permissions
- Before any live read, verify authentication (`az account show`) and target-scope access;
  fail fast with login instructions if unauthenticated.
- Use least-privilege, read-only roles (Reader, Resource Graph reader, Policy Insights
  Reader, Advisor Reader, Security Reader; Graph read scopes if Entra is queried). Report
  permission gaps separately from WAF findings.

## Assessment methodology
- Assess all five pillars: Reliability, Security, Cost Optimization, Operational
  Excellence, Performance Efficiency.
- Ground every check in the authoritative WAF docs via the Learn MCP Server — pillar design
  principles, checklists, recommendation guides, service guides, tradeoffs, and the
  Well-Architected Review. Do not rely on memory.
- Build a dated WAF source manifest (the specific checklist items / recommendation pages
  used, with URLs + retrieval date) instead of trying to ingest "all pages." Every finding
  maps to a specific WAF recommendation/checklist-item URL.
- For each item, classify Compliant / Non-compliant (gap) / Not applicable / Unable to
  assess, with evidence (resource ID + property, OR file + line) and the cited
  recommendation.
- Severity = Critical / High / Medium / Low (impact x likelihood x workload criticality +
  blast radius + remediation urgency). Produce a per-pillar scorecard; exclude N/A and
  Unable-to-assess from scoring and show them separately.
- Treat WAF tradeoffs as first-class — do not flag a justified high-cost or redundancy
  choice as a defect; log intentional tradeoffs with rationale.
- Optionally enrich live runs with Azure Advisor recommendations and Azure Policy
  compliance state, clearly labeled supplementary (they inform but do not override WAF
  findings).
- When IaC and live are both present, do best-effort drift detection (match by resource
  type + evaluated name + location); report orphaned-live, undeployed-IaC, and property
  mismatches separately; prompt when names can't be resolved from string interpolation
  (ARM/Bicep have no Terraform-style state file).
- Flag deprecated services/SKUs and name the supported successor; never recommend retired
  options.

## Safety / guardrails
- Read-only against live Azure — never mutate resources during assessment.
- Redact secrets and PII (keys, tokens, connection strings, passwords, secure params,
  admin usernames, sensitive IPs) before writing to any report AND before sending anything
  to the second LLM.
- Never invent WAF recommendations or IDs; no claim without a real citation; state
  confidence and assumptions.

## Outputs (two consistent reports from one shared dataset)
- Build one shared findings dataset (JSON/YAML or a normalized table) as the source of
  truth so the Markdown and DOCX cannot diverge on counts/severities.
- Markdown report = source of truth, optimized for GitHub Copilot to implement fixes: a
  prioritized remediation backlog where each gap lists pillar, severity, affected
  resource/file, WAF recommendation + URL, and concrete remediation (specific config/IaC
  change or a suggested diff); plus the per-pillar scorecard and a machine-readable
  findings block.
- DOCX report = the same findings for a human audience (executive summary, per-pillar
  scores, tables, prioritized recommendations). Generate by invoking the `docx` skill — do
  not hand-roll custom Python.
- Deterministic output: write to a consistent folder with timestamped filenames + run ID
  (e.g., ./waf-reviews/waf-assessment-YYYYMMDD-HHmm.{md,docx}); never destructively
  overwrite prior runs.

## Cross-model rubber-duck (concrete + bounded)
- Before finalizing, rubber-duck the findings and recommendations against at least one
  other AI LLM using the built-in `task` tool with a different `model` override. Send
  redacted findings only.
- Have the reviewer model check for false positives, missed gaps, mis-calibrated severity,
  and deprecated guidance.
- Bound it: set a time/retry budget; if the sub-agent stalls or returns empty, proceed with
  the primary analysis and note the rubber-duck was unavailable — never block.
- Record what was challenged, what changed (accepted/rejected + rationale), and which
  high-confidence findings stood — in both reports.

## Boundaries
- Assessment only: the agent reads and reports; it does not auto-remediate. The Markdown
  backlog enables a separate, reviewed change (e.g., by another Copilot session).
- Keep grounding scoped to WAF; reference CAF, Azure Advisor, and Azure Policy only as
  supplementary signals, never as a substitute for a cited WAF recommendation.

## Quick check
- Did the agent file include the Learn MCP Server configuration, with MCP scoped to need?
- Does it stay read-only against live Azure and redact secrets/PII before any output or LLM call?
- Does it verify auth up front and report permission gaps separately from WAF gaps?
- Does every finding cite a specific WAF page via Learn MCP, with a retrieval date?
- Does it produce a consistent Markdown AND DOCX from one shared findings dataset?
- Is the Markdown an actionable, prioritized remediation backlog Copilot can execute?
- Does it rubber-duck against a different model (bounded, non-blocking) and record the deltas?
```
