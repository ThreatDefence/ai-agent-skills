# ThreatDefence Alert Interaction Playbook

This file captures the HTTP primitives that proved reliable while clearing the February 2026 backlog. Use it as a quick reference when triaging escalations via API.

## Base URLs

| Purpose | URL | Notes |
| --- | --- | --- |
| API root | `https://portal.threatdefence.io/ac/api/v1` | Default production base URL. Override only if the tenant provides a regional host. |
| Console | `https://console.threatdefence.io/alerts` | Append `/<alertId>` for a deep link. Region-specific consoles follow the same pattern (e.g., `https://eu.console.threatdefence.io/alerts/<alertId>`). |

Define `THREATDEFENCE_API_BASE` in your environment or .env file and reuse it in scripts.

## Authentication

Use the API key header:

```
Authorization: ApiKey <key>
Accept: application/json
Content-Type: application/json
```

Rotate keys regularly and scope them to read/write Alert Centre access.

## Enumerations From Swagger (`threatdefence-openapi.json`)

Pulled from `service.*` definitions so agents don’t have to guess allowed states:

| Field | Values |
| --- | --- |
| `status` (`service.AlertStatus`) | `Closed`, `Open`, `Re-opened`, `Pending`, `Escalated`, `Triaged`, `""` (no change) |
| `category` filters | `open`, `escalated`, `closed` |
| `rating` filters | `unwhitelisted`, `low`, `medium`, `high`, `critical` |
| `module` filters | `o365`, `hids`, `other` |
| `timespan` (`service.Timespan`) | `15m`, `60m`, `8h`, `16h`, `24h`, `48h`, `72h`, `7d`, `30d` |
| `tags` (`service.Tag`) | `SOC247`, `notify_force`, `td_soar` |

> `root_cause` labels: map closures to the ThreatDefence taxonomy below and include the exact label in both the payload and your analyst comment.
>
> | Label |
> | --- |
> | Benign Alert |
> | Expected Behaviour |
> | False Positive |
> | Threat Mitigated |
> | Client Investigating |
> | SOC Team Investigating |
> | Resolved by SOC Team |
> | Escalated to client for action |
> | Client advised no action required |
> | Severity below reporting threshold |
> | Out of SLA terms |
> | Escalated to L2 SOC |
> | Escalated to SOC Manager |
> | Threat Mitigated |
> | Incident |
>
> Keep this table in sync with tenant-provided maps; drop new labels here as they appear.

Fetch the doc anytime via `curl -s https://portal.threatdefence.io/docs/ac/doc.json -o threatdefence-openapi.json` to stay in sync.

## Enumerating Alerts

```
curl -sS \
  -H "Authorization: ApiKey ${THREATDEFENCE_API_KEY}" \
  -H "Accept: application/json" \
  "${THREATDEFENCE_API_BASE}/alerts?status=Open&sort=createdAt&order=asc&limit=100"
```

Common filters align with the enums above. If the list endpoint is degraded (HTTP 500), fall back to specific IDs provided by the customer or export via console CSV.

## Fetching a Single Alert

```
curl -sS \
  -H "Authorization: ApiKey ${THREATDEFENCE_API_KEY}" \
  -H "Accept: application/json" \
  "${THREATDEFENCE_API_BASE}/alerts/${ALERT_ID}"
```

Key response fields:
- `id`, `createdAt`, `updatedAt`
- `title`, `description`, `category`
- `enrichment` (IP, host, user, test flags)
- `status`, `rating`, `tags`
- `metadata.labels[]` for "test" or lab indicators

## Adding Case Notes Without Closing

```
curl -sS -X POST \
  -H "Authorization: ApiKey ${THREATDEFENCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
        "comment": "Documenting lab validation",
        "visibility": "internal"
      }' \
  "${THREATDEFENCE_API_BASE}/alerts/${ALERT_ID}/comments"
```

## Closing / Reclassifying an Alert

Use PUT to `/alerts/{alertId}` with the fields that changed. Minimum payload for benign/test closures:

```
payload=$(cat <<'JSON'
{
  "status": "Closed",
  "classification": "BenignPositive",
  "rootCause": "LoadTest",
  "comment": "Validated as scheduled Keycloak load test. No customer impact."
}
JSON
)

curl -sS -X PUT \
  -H "Authorization: ApiKey ${THREATDEFENCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$payload" \
  "${THREATDEFENCE_API_BASE}/alerts/${ALERT_ID}"
```

`classification` is not enumerated in the current Swagger spec. Default to the ThreatDefence console taxonomy (`TruePositive`, `BenignPositive`, `FalsePositive`, `OperationalTest`) and confirm with the tenant if they publish an updated list. When new values appear in `/cases/selections/*` responses or console metadata, mirror them exactly.

## Console Deep Links

Share a console URL with every update so a human analyst can pivot quickly:

```
https://console.threatdefence.io/alerts/<alertId>
```

When the tenant uses multiple regions, respect their domain (for example `https://au.console.threatdefence.io/alerts/<alertId>`). Always include the direct link in status updates, reports, and customer communications.

## Error Handling

- **401 / 403**: The API key lacks Alert Centre scope or is expired. Rotate and retry.
- **404**: Alert was already closed or belongs to another tenant. Confirm the ID and environment.
- **409**: Concurrent update detected. Re-fetch the alert details, merge your comment, retry.
- **500**: Known issue with the list endpoint. Submit the request ID to ThreatDefence support and continue using targeted GETs.

Log all request IDs returned in the `x-request-id` header for audit trails.
