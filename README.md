<img width="1439" height="848" alt="Screenshot 2026-04-14 at 2 48 31 PM" src="https://github.com/user-attachments/assets/7c282c84-a477-4f2c-b629-c7e4ba50f251" />
# servicenow-incident-api

A custom REST API I built on ServiceNow to manage incidents — plus an outbound webhook that fires automatically when a new incident gets created.

Built this on my Personal Developer Instance (PDI) while learning ServiceNow development. The goal was to go beyond clicking around the UI and actually write scripts that talk to the platform programmatically.

---

## What it does

**Inbound API (3 endpoints):**

- `GET /api/1908869/incident_api/incidents` — returns all incidents from the table
- `GET /api/1908869/incident_api/incidents/{number}` — returns one specific incident by number (like INC0010005)
- `POST /api/1908869/incident_api/incidents` — creates a new incident with short description, urgency, and impact

**Outbound webhook:**

Every time a new incident is created, a Business Rule triggers automatically and sends a POST request with the incident details to an external URL. I tested this with webhook.site and confirmed the payload came through with the real incident data.

---

## The scripts

### GET all incidents

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {

    var gr = new GlideRecord('incident');
    gr.query();

    var results = [];

    while (gr.next()) {
        results.push({
            number: gr.getValue('number'),
            short_description: gr.getValue('short_description'),
            state: gr.getValue('state'),
            urgency: gr.getValue('urgency')
        });
    }

    response.setStatus(200);
    response.setBody({ incidents: results });

})(request, response);
```

### GET incident by number

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {

    var number = request.pathParams.number;

    var gr = new GlideRecord('incident');
    gr.addQuery('number', number);
    gr.query();

    if (gr.next()) {
        response.setStatus(200);
        response.setBody({
            number: gr.getValue('number'),
            short_description: gr.getValue('short_description'),
            state: gr.getValue('state'),
            urgency: gr.getValue('urgency'),
            assigned_to: gr.getDisplayValue('assigned_to')
        });
    } else {
        response.setStatus(404);
        response.setBody({ error: 'Incident not found' });
    }

})(request, response);
```

### POST create incident

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {

    var body = request.body.data;

    if (!body.short_description) {
        response.setStatus(400);
        response.setBody({ error: 'short_description is required' });
        return;
    }

    var gr = new GlideRecord('incident');
    gr.initialize();
    gr.setValue('short_description', body.short_description);
    gr.setValue('urgency', body.urgency || '3');
    gr.setValue('impact', body.impact || '3');
    var sysId = gr.insert();

    response.setStatus(201);
    response.setBody({
        message: 'Incident created',
        sys_id: sysId
    });

})(request, response);
```

### Business Rule — outbound webhook on incident create

```javascript
(function executeRule(current, previous /*null when async*/) {

    var rm = new sn_ws.RESTMessageV2('Notify Incident Created', 'POST Incident');
    rm.setStringParameterNoEscape('number', current.getValue('number'));
    rm.setStringParameterNoEscape('short_description', current.getValue('short_description'));
    rm.setStringParameterNoEscape('urgency', current.getValue('urgency'));
    rm.setStringParameterNoEscape('state', current.getValue('state'));

    var response = rm.execute();

})(current, previous);
```

---

## Proof it works

Created incident INC0010005 through the POST endpoint. The Business Rule fired automatically and the webhook received the payload with the real incident data:

![webhook showing INC0010005](screenshot.png)

---

## Stack

- ServiceNow (PDI)
- GlideRecord API
- Scripted REST API
- RESTMessageV2
- Business Rules
- JavaScript

---

## Why I built this

I'm learning ServiceNow development and wanted to understand how the platform handles API design under the hood — not just use the built-in Table API, but actually build custom endpoints and wire up automation. This was Week 1. Next up is integrating Python (FastAPI) as the consuming layer on the other side.
