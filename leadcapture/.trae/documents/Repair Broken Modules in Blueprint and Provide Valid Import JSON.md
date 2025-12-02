## Findings
- The file `Lead Capture via Webhook → HTTP.blueprint.json` defines HTTP modules as `http:ActionSendData` (ids 8, 9, 10), which renders as “Module Not Found”.
- Webhooks and JSON modules are present but lack `config`, making the flow non-functional.
- Make’s HTTP module identifier is `http:Request` and works across tenants. Airtable can be hit via HTTP safely using the REST API.

## Fix Approach
- Replace `http:ActionSendData` → `http:Request` for all HTTP steps.
- Add minimal `config` to all modules:
  - Webhook: `hook` and `maxResults`.
  - Parse JSON: `json` = `{{1.body}}`.
  - Create JSON: structured payloads for GHL contact, SMS, and Airtable fields.
  - HTTP requests: `url`, `method`, `headers`, `body`.
  - Webhook respond: `status` and `body`.
- Keep the existing `metadata.scenario` and `designer` info.

## Corrected Blueprint JSON (to replace file content)
```
{
  "name": "Lead Capture via Webhook → HTTP",
  "flow": [
    {
      "id": 1,
      "module": "gateway:CustomWebHook",
      "version": 1,
      "config": { "hook": "lead_capture", "maxResults": 1 },
      "metadata": { "designer": { "x": 0, "y": 0 } }
    },
    {
      "id": 2,
      "module": "json:ParseJSON",
      "version": 1,
      "config": { "json": "{{1.body}}" },
      "metadata": { "designer": { "x": 220, "y": 0 } }
    },
    {
      "id": 3,
      "module": "json:CreateJSON",
      "version": 1,
      "config": {
        "structure": {
          "contact": {
            "firstName": "{{2.output.first_name}}",
            "lastName": "{{2.output.last_name}}",
            "email": "{{2.output.email}}",
            "phone": "{{2.output.phone}}",
            "source": "Facebook Lead Ads",
            "tags": ["Facebook Lead", "Form:{{2.output.form_id}}", "Campaign:{{2.output.campaign_id}}"]
          },
          "sms": {
            "to": "{{2.output.phone}}",
            "from": "{{GHL_NUMBER_ID}}",
            "message": "Hi {{2.output.first_name}}, thanks for your interest! Reply STOP to opt out."
          },
          "airtable": {
            "fields": {
              "fb_lead_id": "{{2.output.leadgen_id}}",
              "first_name": "{{2.output.first_name}}",
              "last_name": "{{2.output.last_name}}",
              "email": "{{2.output.email}}",
              "phone": "{{2.output.phone}}",
              "form_id": "{{2.output.form_id}}",
              "campaign_id": "{{2.output.campaign_id}}",
              "adset_id": "{{2.output.adset_id}}",
              "ad_id": "{{2.output.ad_id}}",
              "raw_payload": "{{1.body}}",
              "created_at": "{{now}}",
              "updated_at": "{{now}}"
            }
          }
        }
      },
      "metadata": { "designer": { "x": 440, "y": 0 } }
    },
    {
      "id": 8,
      "module": "http:Request",
      "version": 1,
      "config": {
        "url": "https://services.leadconnectorhq.com/contacts/",
        "method": "POST",
        "headers": { "Authorization": "Bearer {{GHL_API_TOKEN}}", "Content-Type": "application/json" },
        "body": "{{3.output.contact}}"
      },
      "metadata": { "designer": { "x": 660, "y": 0 } }
    },
    {
      "id": 9,
      "module": "http:Request",
      "version": 1,
      "config": {
        "url": "https://services.leadconnectorhq.com/conversations/messages/",
        "method": "POST",
        "headers": { "Authorization": "Bearer {{GHL_API_TOKEN}}", "Content-Type": "application/json" },
        "body": "{{3.output.sms}}"
      },
      "metadata": { "designer": { "x": 880, "y": 0 } }
    },
    {
      "id": 10,
      "module": "http:Request",
      "version": 1,
      "config": {
        "url": "https://api.airtable.com/v0/{{AIRTABLE_BASE_ID}}/Leads%20Log",
        "method": "POST",
        "headers": { "Authorization": "Bearer {{AIRTABLE_API_TOKEN}}", "Content-Type": "application/json" },
        "body": "{\"fields\": {{3.output.airtable.fields}} }"
      },
      "metadata": { "designer": { "x": 1100, "y": 0 } }
    },
    {
      "id": 7,
      "module": "gateway:WebhookRespond",
      "version": 1,
      "config": { "status": 200, "body": "{\"status\":\"ok\"}" },
      "metadata": { "designer": { "x": 1320, "y": 0 } }
    }
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
    "designer": { "orphans": [] }
  }
}
```

## After Replacement
- Open each HTTP module and fill the placeholders:
  - `{{GHL_API_TOKEN}}`, `{{AIRTABLE_API_TOKEN}}`, `{{AIRTABLE_BASE_ID}}`, `{{GHL_NUMBER_ID}}`
- Send a test payload to the webhook; verify contact upsert, SMS send, and Airtable log.

## Optional Variant (Native Airtable)
- Swap the Airtable HTTP step with the native module “Create a Record” after import; identifiers differ per app version, so replacing in the UI is safer than hand-coding keys.

## Approval
- On approval, I will replace the file content with the corrected JSON and verify the import in Make’s editor view.