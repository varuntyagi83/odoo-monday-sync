# Build Plan: Odoo → Monday.com Manufacturing Order Sync

**For:** Claude Code, implementing on `n8n.sunday.de` (production)
**Owner:** Varun Tyagi
**Reference:** `PRD_Odoo_Monday_MO_Sync.docx`

---

## Context you need before starting

You are building an n8n automation that synchronizes Manufacturing Orders between Odoo (ERP, system of record) and Monday.com (cross-functional coordination board). The work is scoped for production n8n at `n8n.sunday.de`.

**Scope (confirmed):**
- One-way Odoo → Monday: create AND update on MO changes
- Bidirectional for a whitelisted subset of fields only (Monday → Odoo: status transitions, owner, planned completion date)

**What you must not do:**
- Do not use the n8n MCP. It is bound to one instance and blocks webhook-trigger workflows. Use the n8n REST API with the API key in 1Password (`Sunday Natural` vault, item `n8n.sunday.de API key`).
- Do not create MOs from Monday. Monday is never the system of record.
- Do not delete records in either direction. Archival only.
- Do not hardcode field mappings inside sync logic. Field mapping lives in ONE config node at the top of each main workflow.

---

## Target systems

### Odoo (source of truth)

- **Instance:** `https://odoo.sunday.de`
- **Module:** Manufacturing (`mrp.production` model)
- **Menu path:** Manufacturing → Operations → Manufacturing Orders (menu_id=301, action=443)
- **Sample record URL:** `https://odoo.sunday.de/web#id=&action=443&model=mrp.production&view_type=form&cids=1&menu_id=301`
- **Company ID:** cids=1 (Sunday Natural GmbH)
- **API access:** XML-RPC or JSON-RPC via the integration service user. Do NOT use an admin account.

### Monday.com (coordination surface)

- **Workspace:** `https://sundaynatural.monday.com`
- **Target board ID:** `18375520163`
- **Board URL:** `https://sundaynatural.monday.com/boards/18375520163`
- **API region:** EU (use `https://api.monday.com/v2`, EU tenants route automatically)
- **API version:** Use the current stable API version, confirm headers with `API-Version` on first call

### First thing to do

Before writing any workflow, fetch the live structure of both systems and save the results to `/docs/discovery/`:

1. **Odoo `mrp.production` schema:** Run `execute_kw` with `fields_get` on `mrp.production` to get the full field list. Save as `odoo_mrp_production_fields.json`.
2. **Monday board structure:** Run this GraphQL query against board `18375520163` and save as `monday_board_18375520163.json`:

```graphql
query {
  boards(ids: [18375520163]) {
    id
    name
    columns { id title type settings_str }
    groups { id title }
    items_page(limit: 5) {
      items { id name column_values { id type value text } }
    }
  }
}
```

These two files replace guesswork. Every field mapping decision references them.

---

## Prerequisites (confirm before building)

Run through this checklist before writing any workflow JSON. If anything is missing, stop and flag it in the Jira ticket.

- [ ] n8n API key for `n8n.sunday.de` is available and has write access
- [ ] Monday.com API token is provisioned, with read/write on board `18375520163`
- [ ] Odoo API credentials exist for the integration service user (least-privilege, scoped to `mrp.production` model read and write, on `odoo.sunday.de`)
- [ ] Odoo outgoing webhook capability confirmed by Ciprian (if unavailable, fall back to 2-minute polling on `write_date`)
- [ ] Live Monday board structure fetched and saved to `/docs/discovery/monday_board_18375520163.json`
- [ ] Live Odoo MO schema fetched and saved to `/docs/discovery/odoo_mrp_production_fields.json`
- [ ] Field mapping table from the discovery session has been captured and pasted into this plan below
- [ ] User mapping table (Odoo user → Monday user) exists or has been created
- [ ] BigQuery dataset and table for audit log exists (`sundaybi.ops_automation.mo_sync_events`)
- [ ] `WF-Global-Error-Alert` workflow exists and posts to the Central Data Team Slack channel (if not, build or update it as part of Phase 1)

---

## Field mapping (to be finalized from discovery)

**Paste the final mapping here after the kickoff.** This is the single source of truth for what the workflow transfers.

> **WARNING:** The JSON below is a STRUCTURAL TEMPLATE only. The `monday_column_id` values (e.g., `text_product`, `numbers_qty`, `status`) are placeholders and are NOT the real column IDs on board `18375520163`. Real column IDs come from the Monday API response saved in `/docs/discovery/monday_board_18375520163.json` and must replace every placeholder before the workflow is built. The same rule applies to status label values in the `map` block: they must match the exact label strings configured on the live status column.

```json
{
  "mo_to_monday": {
    "item_name_template": "{odoo.name} | {odoo.product_id.name}",
    "group_selector": "odoo.product_id.category_id.name",
    "columns": {
      "product": { "odoo_field": "product_id.name", "monday_column_id": "text_product", "type": "text" },
      "quantity": { "odoo_field": "product_qty", "monday_column_id": "numbers_qty", "type": "numbers" },
      "planned_start": { "odoo_field": "date_planned_start", "monday_column_id": "date_start", "type": "date" },
      "planned_end": { "odoo_field": "date_planned_finished", "monday_column_id": "date_end", "type": "date", "bidirectional": true },
      "status": { "odoo_field": "state", "monday_column_id": "status", "type": "status", "bidirectional": true, "map": {
        "draft": "Draft",
        "confirmed": "Ready to start",
        "progress": "In production",
        "to_close": "Awaiting QA",
        "done": "Complete",
        "cancel": "Cancelled"
      }},
      "owner": { "odoo_field": "user_id", "monday_column_id": "people", "type": "people", "bidirectional": true, "resolver": "user_mapping" },
      "lots": { "odoo_field": "lot_ids", "monday_column_id": "text_lots", "type": "text" },
      "bom_ref": { "odoo_field": "bom_id.code", "monday_column_id": "text_bom", "type": "text" },
      "entity": { "odoo_field": "company_id.name", "monday_column_id": "dropdown_entity", "type": "dropdown" }
    }
  },
  "monday_to_odoo_whitelist": ["status", "owner", "planned_end"]
}
```

---

## Workflow architecture

Two workflows. One per direction. That's it.

Field mapping lives in a single Code node at the top of each workflow, loaded from the JSON config above. Audit logging is a direct BigQuery insert node. Error alerting is handled by the org-wide `WF-Global-Error-Alert` workflow (n8n's Error Trigger pattern), which is infrastructure, not part of this build.

### 1. `WF-MO-Sync-Odoo-to-Monday`

**Trigger options, in order of preference:**
1. Webhook endpoint receiving Odoo outgoing webhook on `mrp.production` create/write
2. Schedule trigger every 2 minutes, polling `mrp.production` filtered by `write_date > last_run_timestamp`

**Flow:**

```
[Trigger]
    ↓
[Code: load mapping_config]
    ↓
[Fetch full MO from Odoo by ID]  (includes related records: product, BOM, user, lots)
    ↓
[Code: apply field mapping, produce mapped_data + changed_fields]
    ↓
[Check for existing Monday item on board 18375520163 by MO reference]
    ↓
    ├── Not found → CREATE path
    └── Found → UPDATE path

CREATE path:
    ↓
[Resolve target Monday group from mapping_config.group_selector]
    ↓
[Create Monday item via create_item mutation]
    ↓
[Write Monday item ID back to Odoo MO custom field x_monday_item_id]
    ↓
[BigQuery insert: outcome=success]

UPDATE path:
    ↓
[If changed_fields is empty → BigQuery insert outcome=skipped_no_change, end]
    ↓
[Update Monday item via change_multiple_column_values mutation, set hidden column x_last_sync_source=odoo in the same call]
    ↓
[BigQuery insert: outcome=success]
```

**Error handling:** Set n8n node retry to 3 attempts with exponential backoff (5s, 30s, 2m) on the Monday API and Odoo API nodes directly. n8n's built-in retry handles this, no custom logic needed. If all retries fail, the node errors and the Error Trigger workflow posts to Slack.

### 2. `WF-MO-Sync-Monday-to-Odoo`

**Trigger:** Monday webhook subscription on board `18375520163`, event `change_column_value`.

**Flow:**

```
[Monday webhook trigger]
    ↓
[Code: load mapping_config]
    ↓
[If column_id not in monday_to_odoo_whitelist → BigQuery insert skipped_not_whitelisted, end]
    ↓
[Fetch item's x_last_sync_source hidden column]
    ↓
[If value is "odoo" → BigQuery insert skipped_self_triggered, end]
    ↓
[Fetch Odoo MO ID from Monday item back-reference]
    ↓
[Code: apply field mapping for monday_to_odoo direction]
    ↓
[Write to Odoo via mrp.production.write()]
    ↓
[Set x_last_sync_source=monday on the Monday item]
    ↓
[BigQuery insert: outcome=success]
```

**Error handling:** Same as workflow 1. Node-level retry, Error Trigger catches unrecoverable failures.

### Critical implementation details (apply to both workflows)

- Monday GraphQL: always use `change_multiple_column_values`, never one call per column (complexity budget)
- Monday API rate limit: handled by n8n node retry respecting `Retry-After` header
- **Loop prevention:** `x_last_sync_source` is the only thing stopping a ping-pong. Workflow 1 sets it to `odoo` in the same mutation that writes the content. Workflow 2 sets it to `monday` after writing to Odoo. Each workflow checks this marker on entry and exits early if its own change triggered the remote webhook.
- Field mapping is loaded in a Code node at the top of each workflow. Changing the mapping means editing two Code nodes (acceptable: they diverge subtly over time anyway, keeping them local is cleaner than shared indirection).
- Audit log is a single BigQuery insert node per outcome branch. Schema:

```
event_id          STRING (UUID, generated in Code node)
event_timestamp   TIMESTAMP
direction         STRING  (odoo_to_monday | monday_to_odoo)
odoo_mo_id        INTEGER
odoo_mo_reference STRING
monday_item_id    STRING
fields_changed    JSON
outcome           STRING  (success | skipped_no_change | skipped_not_whitelisted | skipped_self_triggered | failed)
error_message     STRING
trigger_source    STRING  (odoo_webhook | odoo_poll | monday_webhook)
workflow_execution_id STRING
```

---

## Build phases (7 steps, each ends with a commit to Bitbucket)

### Phase 1: Setup (0.5 day)
1. Create BigQuery table `sundaybi.ops_automation.mo_sync_events` with the schema above. Create n8n credentials: `monday-api-token`, `odoo-integration-user`, `bigquery-ops-automation`. Confirm `WF-Global-Error-Alert` exists or create it.

### Phase 2: Odoo → Monday (3-4 days)
2. Build `WF-MO-Sync-Odoo-to-Monday` CREATE path. Trigger manually with a real MO ID. Verify a Monday item is created on board `18375520163` and the back-reference is written to Odoo.
3. Add the UPDATE path with field diffing. Trigger with a modified MO. Verify only changed columns are written in Monday (inspect Monday activity log).
4. Wire up the Odoo webhook trigger (or polling fallback if Ciprian confirms webhooks are not available). End-to-end test: create a new MO in Odoo, verify it appears in Monday within 2 minutes.

### Phase 3: Monday → Odoo (2-3 days)
5. Build `WF-MO-Sync-Monday-to-Odoo` with the whitelist check and loop prevention.
6. Wire up the Monday webhook subscription on board `18375520163`. End-to-end test: change a whitelisted column in Monday, verify Odoo reflects the change. Change a non-whitelisted column, verify Odoo does NOT change.

### Phase 4: Hardening (1 day)
7. Loop-prevention stress test: in rapid succession, update the same MO from both sides. Verify no infinite loop, final state is correct (Odoo wins on conflict), all events logged correctly.

---

## Testing checklist before cutover

Run these explicitly with Mariusz present.

- [ ] New MO in Odoo → appears in Monday in correct group with correct columns
- [ ] MO state change in Odoo → Monday status column updates
- [ ] MO quantity change in Odoo → Monday quantity updates, other columns untouched
- [ ] MO cancellation in Odoo → Monday item moves to Cancelled group, not deleted
- [ ] Status change in Monday (whitelisted) → Odoo state updates
- [ ] Owner change in Monday → Odoo user_id updates
- [ ] Random column change in Monday (not whitelisted) → nothing happens in Odoo
- [ ] Simultaneous update both sides → Odoo wins, audit log shows both events
- [ ] Monday API rate limit simulated (fire 50 updates in 10 seconds) → backoff works, no dropped events
- [ ] Odoo down (simulate by revoking credentials) → failure alert fires in Slack after 3 retries
- [ ] One-off backfill script (separate from main workflow) runs against existing open MOs without creating duplicates in Monday

---

## Documentation deliverables (required before cutover)

- [ ] Confluence page in the DTA space: architecture diagram, field mapping, runbook for common failures
- [ ] README inside n8n workflow description field: purpose, owner, dependencies, escalation path
- [ ] Slack post in ops channel when cutover happens: what changed, who to ping if anything looks wrong

---

## Questions to resolve before starting

If any of these are unresolved, stop and flag them. Do not guess.

1. Is the field mapping above final, or is there a newer version from the Michael/Mariusz kickoff?
2. Has Ciprian confirmed Odoo outgoing webhook availability, or do we proceed with polling as primary?
3. Is the BigQuery audit log approach approved, or should the log live elsewhere (e.g., Monday itself as a secondary board)?
4. Has Monday API access been granted for board `18375520163` with read and write scope?
5. Should the one-off backfill script be part of this engagement, or handled separately after cutover?

---

## Acceptance criteria

The automation is considered done when:

- All 11 test cases above pass
- Mariusz has run one full production day with automated sync in parallel with his manual process and confirmed parity
- Cutover has happened: manual process stopped, automation is sole path
- 2 weeks of hypercare has passed with no unresolved incidents
- Varun has signed off in the Jira ticket (project DTA)
