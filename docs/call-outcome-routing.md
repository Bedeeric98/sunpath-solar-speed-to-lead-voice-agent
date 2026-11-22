# SunPath Solar — Call Outcome Routing Reference

GHL field IDs:
- `call_in_progress` → `YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS`
- `lead_status` → `YOUR_GHL_FIELD_ID_LEAD_STATUS`
- `solar_qualified` → `YOUR_GHL_FIELD_ID_SOLAR_QUALIFIED`

---

## EOCR Outcomes → GHL Fields

**Updated 2026-09-02:** added `qualified_not_booked` (a lead qualified but the call ended/cut off before a specific appointment time was confirmed and booked — distinct from a genuine `qualified_booked`, where `book_appointment_sunpath` actually fired). Telnyx's own AI-generated call summary can otherwise hallucinate "booked" language from a truncated conversation — don't trust the outcome label alone for a state-changing outcome; this new value exists specifically so the workflow doesn't have to. Admin notifications (Slack + Gmail) were also added to `qualified_booked` and `qualified_not_booked` — previously `qualified_booked` was the only outcome with no admin notification at all, despite representing real revenue-adjacent activity.

| call_outcome | lead_status value | solar_qualified | call_in_progress | Notifications |
|---|---|---|---|---|
| `qualified_booked` | qualified_booked | yes | no | Slack + Gmail |
| `qualified_not_booked` | qualified_not_booked | yes | no | Slack + Gmail |
| `not_qualified` | not_qualified | no | no | Slack + Gmail |
| `callback_requested` | callback_requested | — | no | Slack + Gmail |
| `wrong_person` | wrong_person | — | no | Slack + Gmail |
| `no_meaningful_conversation` | no_meaningful_conversation | — | no | Slack + Gmail |
| `do_not_call` | do_not_call | — | no | Slack + Gmail |

`do_not_call` also picked up admin notifications on 2026-07-20 — it was the highest compliance-risk outcome and had been the only one with none at all until then.

**Inbound EOCR recovery layer (added 2026-09-02):** inbound now has the same Redis-backed recovery path as outbound — if Telnyx's `analyzed` webhook never arrives (confirmed real failure mode), a 7-minute wait + Telnyx-API pull rejoins the same `Search Contact on GHL For Inbound EOCR` → outcome-routing flow above, so every row in this table applies identically regardless of which path got there.

---

## No-Answer / Voicemail → GHL Fields

| Situation | lead_status value | call_in_progress | Notifications |
|---|---|---|---|
| Call 1 no-answer or voicemail | no_answer_call_1 | no | None — n8n Wait 5min → GHL check → fire Call 2 |
| Call 2 no-answer or voicemail | no_answer_call_2 | no | Send Telnyx SMS; GHL workflow queues 12hr Call 3 |
| Call 3 no-answer or voicemail | exhausted | no | Slack + Gmail — all attempts used |

Voicemail detection (AnsweredBy = machine_start) is handled in TeXML Router section, not Call Status Handler.

---

## GHL Update Bodies

### EOCR — qualified_booked
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "qualified_booked" },
    { "id": "YOUR_GHL_FIELD_ID_SOLAR_QUALIFIED", "fieldValue": "yes" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

### EOCR — qualified_not_booked
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "qualified_not_booked" },
    { "id": "YOUR_GHL_FIELD_ID_SOLAR_QUALIFIED", "fieldValue": "yes" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

### EOCR — not_qualified
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "not_qualified" },
    { "id": "YOUR_GHL_FIELD_ID_SOLAR_QUALIFIED", "fieldValue": "no" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

### EOCR — callback_requested / wrong_person / no_meaningful_conversation
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "{{ OUTCOME }}" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

### EOCR — do_not_call
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "do_not_call" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

**Inbound now has the same 7 outcomes as outbound (added 2026-09-02):** `Route Inbound Call Outcome` was a 6-way switch missing `do_not_call` entirely — an inbound caller opting out had no way to be recorded. Added as a 7th rule appended at the end of the switch (never insert mid-array — positional connection indices would silently misroute the existing 6 rules). Inbound's `Update GHL For Do Not Call Route Inbound` body matches the outbound one above, using `key`/`fieldValue` (camelCase) instead of `id`/`fieldValue` — both forms are live and correct in this workflow.

### Call Status — in-progress (write flag)
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "yes" }
  ]
}
```

### Call Status — no-answer Call 1 / Voicemail Call 1
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "no_answer_call_1" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

### Call Status — no-answer Call 2 / Voicemail Call 2
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "no_answer_call_2" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

### Call Status — no-answer Call 3 / exhausted
```json
{
  "customFields": [
    { "id": "YOUR_GHL_FIELD_ID_LEAD_STATUS", "fieldValue": "exhausted" },
    { "id": "YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS", "fieldValue": "no" }
  ]
}
```

---

## GHL Note Headers by Outcome

| Outcome | Header |
|---|---|
| qualified_booked | Lead qualified and booked for solar consultation. |
| qualified_not_booked | Lead qualified but the call ended before a specific appointment time was confirmed and booked. Human follow-up required. |
| not_qualified | Lead did not qualify — failed homeowner, roof, or bill criteria. |
| callback_requested | Lead requested a callback — preferred time noted. |
| wrong_person | Wrong person answered the call. |
| no_meaningful_conversation | Call connected but no useful conversation took place. Human follow-up required. |
| do_not_call | Lead opted out — do not contact again. |
| no_answer_call_1 | No answer — Call 1. Call 2 queued. |
| no_answer_call_2 | No answer — Call 2. SMS sent. 12-hour Call 3 queued. |
| exhausted | All three call attempts failed. No answer. Human follow-up required. |
| inbound qualified_booked | Inbound call — lead called back and qualified. Consultation booked. |
| inbound qualified_not_booked | Inbound call — lead called back and qualified, but the call ended before booking could be completed. |
| inbound not_qualified | Inbound call — lead called back but did not qualify. |
| inbound callback_requested | Inbound call — lead called back but requested a callback. |
| inbound wrong_person | Inbound call — wrong person answered. |
| inbound no_meaningful_conversation | Inbound call — connected but no useful conversation took place. |
| inbound do_not_call | Inbound call — lead opted out — do not contact again. |

---

## GHL Note Body Template (SunPath)

```
[HEADER LINE]

Call Report — {{ $now.setZone('America/New_York').toFormat('cccc d LLLL yyyy, h:mm a') }} ET

Outcome: {{ call_outcome }}
Lead Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}

Summary:
{{ call_summary }}

Transcript:
{{ transcript_text }}

Recording: {{ recording_url }}
```
