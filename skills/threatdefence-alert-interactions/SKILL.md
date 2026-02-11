---
name: threatdefence-alert-interactions
description: Triage and update ThreatDefence Alert Centre escalations via API + console. Use when an agent must enumerate alerts, add analyst notes, close or reclassify cases, or prepare customer-ready status updates (HTML/PDF) for ThreatDefence SIEM tenants.
---

# ThreatDefence Alert Interactions

This skill now lives in three clearly separated tracks—**Alert Triage**, **Alert Escalation**, and **Reporting**—with explicit guardrails on when to run each track and when to stop.

### STOP Signs (read before you run anything)
- **DO NOT proceed with triage** if you cannot explain the trigger or you doubt the telemetry—pause and ask the on-call SOC lead for clarification.
- **DO NOT escalate** until you have console evidence, alert status, and desired customer action documented. Escalations without a call-to-action slow response.
- **DO NOT publish a report** until every alert reference includes a working console hyperlink and root-cause enum pulled from the official list.

---

## Track 1 – Alert Triage Skill

### Purpose
Quickly drive alerts from "Open" to a documented state (Closed, Escalated, Pending, etc.) with console links, API payloads, and whitelist guidance.

### Inputs / Prep
1. Export environment variables:
   - `THREATDEFENCE_API_BASE` (default `https://portal.threatdefence.io/ac/api/v1`).
   - `THREATDEFENCE_API_KEY` (least-privileged key with alert scope).
   - `TENANT_CONSOLE_BASE` (`https://console.threatdefence.io/app/dashboards#/view/...`).
2. Load `references/api-playbook.md` for enum lists, payload templates, and curl snippets.
3. Ensure you can reach both API and console; capture `x-request-id` headers for every API request.

### Workflow
1. **Enumerate backlog** – Prefer `GET /alerts?status=Open`. If the endpoint errors, pivot to user-provided IDs or console CSV exports.
2. **Pull full context** – `GET /alerts/{id}` for each candidate. Cross-check `rating`, `tags`, `worklog`, and `secops.AI` for automation hints. Note any `metadata.labels` that indicate tests.
3. **Validate** – Compare timestamps against change windows, review exchange/sharepoint enrichments, and confirm user/device history. Use console hunt links for pivots.
4. **Document investigation** – Draft a concise note containing: trigger summary, validation steps, console link, next action. When still investigating, `POST /alerts/{id}/comments` with this note.
5. **Decide close vs. keep open** – When closing, `PUT /alerts/{id}` with updated `status`, `root_cause`, `comment`, and optional `assigned_to`. Status/root_cause **must** use the Swagger enums listed in `references/api-playbook.md`.
6. **Console hyperlink discipline** – Every alert reference must be formatted exactly as `[alertId](https://console.threatdefence.io/...%22alertId%22`)) with a single set of brackets/parentheses. Manually eyeball the markdown to ensure each link begins with ``[alertId](`` and ends with `)`—no nested links or code blocks.
7. **Suggest whitelists** – Whenever you close a benign/test alert, recommend a whitelist entry. Cite the specific field match (e.g., ``source.system:"{{WHITELIST_INDICATOR}}"``). Use `references/whitelisting.md` for Lucene syntax and guardrails.
8. **Update tracking artifacts** – Log closed IDs, timestamps, `x-request-id`, and residual backlog for the reporting track.

### Anti-patterns (don’t do this)
- Closing without a narrative that states generator, validation steps, and customer impact.
- Referencing alerts without console links (breaks investigations).
- Leaving `status`/`root_cause` blank or using non-enum strings.

---

## Track 2 – Alert Escalation Skill

### Purpose
Escalate alerts that require customer or executive action via email, Jira, Telegram, or webhooks with consistent messaging.

### Inputs / Prep
1. Completed triage decision that justifies escalation.
2. Alert ID, console link, desired outcome, and supporting evidence.
3. Destination contact list (e.g., `{{PRIMARY_CONTACT_EMAIL}}`, Jira project key, Telegram chat).

### Workflow
1. **Create escalation record** – `PUT /alerts/{alertId}/escalations` with `analyst_notes` summarizing impact plus the console link.
2. **Capture escalation UUID** – Response `msg` includes the ID; store it for downstream notifications.
3. **Notify channels** – Use the same UUID to send emails (`.../emails`), Jira issues (`.../issues`), Telegram/webhooks (`.../messages`, `.../webhooks`). Update the payload subject/body with variables, e.g., `"subject": "{{ALERT_SHORT_SUMMARY}} – {{TENANT_NAME}}"`.
4. **Audit trail** – Add a standard comment inside the alert referencing the escalation ID, destination, and timestamp.
5. **Follow-up loop** – Monitor customer replies. Once they respond, update the alert `status` and `root_cause`, and note any remediation steps.

### Guardrails
- DO NOT escalate without specifying what you expect the recipient to do.
- DO NOT reuse personal names (e.g., "Chris") inside templated payloads—use variables (`{{PRIMARY_CONTACT_NAME}}`) or role labels ("Healius SecOps").
- Include console hyperlinks in every outbound note so humans can pivot instantly.

### References
- `references/escalations.md` – API payload examples with variable placeholders.

---

## Track 3 – Reporting Skill

### Purpose
Produce polished HTML/PDF deliverables summarizing triage/escalation activity, backlog, and next steps without hard-coded personal names.

### Inputs / Prep
1. Latest triage outcomes (including alert IDs, statuses, root causes, console links).
2. Outstanding backlog + action owners.
3. A blank HTML report file to populate (create one if it doesn’t exist yet).

### Workflow
1. **Gather data** – Build resolved/backlog tables from the JSONL/state files you produced during triage.
2. **Draft HTML report** – Create or update your working HTML file with sections for Executive Summary, Resolved Alerts, Outstanding Backlog, and Next Steps. Ensure every alert ID is hyperlinked per the triage rule and replace personal names with variables/roles.
3. **Generate PDF** – Convert that HTML to PDF with `wkhtmltopdf report.html report.pdf` (or equivalent). Store both under `workspace/` for easy retrieval.
4. **Distribute** – Provide the HTML/PDF via workspace paths; if binaries can’t be uploaded, share base64 plus decode instructions.
5. **Document recipients** – Track where the report was delivered (email, workspace share, etc.) in case notes or MEMORY files.

### Reporting Guardrails
- Replace any static names/email addresses with variables (`{{CUSTOMER_CONTACT}}`, `{{MSP_NAME}}`).
- Validate that the "Outstanding" section lists who owns the next action and when you’ll re-check.
- Don’t recycle stale data—refresh data before exporting when backlog changes.

---

## Shared Resources
- [`references/api-playbook.md`](references/api-playbook.md) – Endpoint cheatsheet, enum tables, payload templates, console-link format, and error handling notes.
- [`references/escalations.md`](references/escalations.md) – Escalation workflow with variable-friendly payloads.
- [`references/whitelisting.md`](references/whitelisting.md) – Lucene suppression guide with placeholder-based examples.
- `threatdefence-openapi.json` (workspace root) – Cached Swagger spec; refresh when the platform publishes updates.

Follow the decision matrix, obey the guardrails, and keep console links + enums flawless. That’s how we stay publish-ready.
