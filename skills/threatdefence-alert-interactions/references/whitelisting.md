# Whitelisting & Noise Suppression

Use whitelist entries to silence recurring benign alerts by matching known fields with Lucene queries. ThreatDefence exposes the whitelist API under `/msps/{msp}/tenants/{tenant}/whitelist`.

## 1. Review Existing Entries

```
curl -s \
  -H "Authorization: ApiKey ${THREATDEFENCE_API_KEY}" \
  "${THREATDEFENCE_API_BASE}/msps/${MSP}/tenants/${TENANT}/whitelist" | jq '.whitelist[]'
```

Look for duplicates before adding a new rule. Each entry contains:
- `name`
- `whitelist_notes`
- `whitelist_query` (Lucene expression)
- `whitelisted_by`

## 2. Identify Stable Fields

Common fields worth filtering:
- `source.system` (e.g., `source.system:"{{WHITELIST_INDICATOR}}"`)
- `td.alert.name`
- `user.email`
- `event.module`
- `destination.ip`

Always verify the field exists in the alert payload (`GET /alerts/{id}`) before building the query.

## 3. Add a New Whitelist Entry

```
payload=$(cat <<'JSON'
{
  "name": "Ignore darkweb hits for {{WHITELIST_DISPLAY_NAME}}",
  "notes": "{{JUSTIFICATION}}",
  "query": "source.system:\"{{WHITELIST_INDICATOR}}\""
}
JSON
)

curl -s -X PUT \
  -H "Authorization: ApiKey ${THREATDEFENCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$payload" \
  "${THREATDEFENCE_API_BASE}/msps/${MSP}/tenants/${TENANT}/whitelist"
```

**Guardrails:**
- Keep queries as specific as possible to avoid suppressing legitimate incidents.
- Document why the alert is safe to ignore in `notes`.
- Review whitelists quarterly to ensure they still make sense.

## 4. Testing a Query

Before committing, run the Lucene expression against alert data via the console or an ad-hoc search endpoint (if available). A quick sanity check avoids silencing too much.

## 5. Removal / Updates

When a whitelist is no longer valid, delete or edit it using the entry ID pulled from the GET response. Always log the change in case notes or MEMORY.md for audit trails.
