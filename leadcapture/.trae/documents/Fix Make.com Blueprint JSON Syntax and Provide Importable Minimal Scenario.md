## Why Import Failed
- Uses non-standard top-level key `modules` instead of `flow`.
- Uses `type` for module identifiers; Make’s blueprint expects `module` (e.g., `json:ParseJSON`, `http:ActionSendData`, `gateway:CustomWebHook`).
- Includes `links` and an `errorHandler` block that aren’t part of the importable blueprint schema.
- Missing the required `metadata` wrapper with scenario and designer info.

## Correct Blueprint Structure
- Top-level keys:
  - `name`: Scenario name
  - `flow`: Array of module objects
  - `metadata`: `{ version, scenario, designer }`
- Module object keys:
  - `id`: Numeric unique ID
  - `module`: Valid module identifier (e.g., `gateway:CustomWebHook`)
  - `version`: Usually `1`
  - `config`: Module configuration (minimal defaults so import succeeds)
  - `metadata.designer`: `{ x, y }` coordinates for layout

## Minimal, Import-Friendly Scenario
- Modules:
  1. `gateway:CustomWebHook` — receive payload
  2. `json:ParseJSON` — parse `{{1.body}}`
  3. `json:CreateJSON` — prepare HTTP bodies (GHL contact, SMS, Airtable fields)
  4. `http:ActionSendData` — upsert GHL contact (placeholder URL and headers)
  5. `http:ActionSendData` — send SMS (placeholder URL and headers)
  6. `http:ActionSendData` — log to Airtable (placeholder URL and headers)
  7. `gateway:WebhookRespond` — return 200 OK
- Placeholders to be set post-import in UI: `{{GHL_API_TOKEN}}`, `{{AIRTABLE_API_TOKEN}}`, `{{AIRTABLE_BASE_ID}}`, `{{GHL_NUMBER_ID}}`.
- Intentionally minimal `config` per module to avoid schema validation errors; after import, connect real accounts or replace HTTP with native app modules.

## Deliverables After Approval
- Replace the existing JSON with a corrected blueprint that follows the schema:
```
{
  "name": "Lead Capture via Webhook → HTTP",
  "flow": [
    {"id":1,"module":"gateway:CustomWebHook","version":1,
      "config":{"hook":"lead_capture","maxResults":1},
      "metadata":{"designer":{"x":0,"y":0}}},
    {"id":2,"module":"json:ParseJSON","version":1,
      "config":{"json":"{{1.body}}"},
      "metadata":{"designer":{"x":220,"y":0}}},
    {"id":3,"module":"json:CreateJSON","version":1,
      "config":{"structure":{/* contact, sms, airtable fields as discussed */}},
      "metadata":{"designer":{"x":440,"y":0}}},
    {"id":4,"module":"http:ActionSendData","version":1,
      "config":{"url":"https://services.leadconnectorhq.com/contacts/","method":"POST",
        "headers":{"Authorization":"Bearer {{GHL_API_TOKEN}}","Content-Type":"application/json"},
        "body":"{{3.output.contact}}"},
      "metadata":{"designer":{"x":660,"y":0}}},
    {"id":5,"module":"http:ActionSendData","version":1,
      "config":{"url":"https://services.leadconnectorhq.com/conversations/messages/","method":"POST",
        "headers":{"Authorization":"Bearer {{GHL_API_TOKEN}}","Content-Type":"application/json"},
        "body":"{{3.output.sms}}"},
      "metadata":{"designer":{"x":880,"y":0}}},
    {"id":6,"module":"http:ActionSendData","version":1,
      "config":{"url":"https://api.airtable.com/v0/{{AIRTABLE_BASE_ID}}/Leads%20Log","method":"POST",
        "headers":{"Authorization":"Bearer {{AIRTABLE_API_TOKEN}}","Content-Type":"application/json"},
        "body":"{\"fields\": {{3.output.airtable.fields}} }"},
      "metadata":{"designer":{"x":1100,"y":0}}},
    {"id":7,"module":"gateway:WebhookRespond","version":1,
      "config":{"status":200,"body":"{\"status\":\"ok\"}"},
      "metadata":{"designer":{"x":1320,"y":0}}}
  ],
  "metadata": {
    "version": 1,
    "scenario": {
      "roundtrips": 1,
      "maxErrors": 3,
      "autoCommit": true,
      "autoCommitTriggerLast": true,
      "sequential": false,
      "confidential": false,
      "dataloss": false,
      "dlq": false,
      "freshVariables": false
    },
    "designer": {"orphans": []}
  }
}
```
- This structure mirrors the format accepted by Make’s scenario creation endpoints and is safe for import via the UI.

## Post-Import Steps
- Open each HTTP module and paste real tokens and IDs.
- If desired, replace HTTP modules with native Airtable and GoHighLevel modules and remap fields.
- Run once with a sample webhook payload; then activate.

## Optional Next Step
- If import errors persist, share the exact error message; I will adjust module configs (e.g., required config keys) accordingly. I can also produce a variant that uses only Webhooks and JSON modules to confirm the blueprint parser accepts the file before layering HTTP calls.