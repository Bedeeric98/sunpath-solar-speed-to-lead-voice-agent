# SunPath Solar — Speed to Lead
## Phase 1: Requirements

---

## Use Case

**Type:** Outbound only — speed-to-lead  
**Trigger:** New lead submits a Facebook lead gen ad form  
**Goal:** AI calls the lead within seconds of form submission, qualifies them for solar, and books an appointment with a solar rep  

---

## Industry Context

**Company:** SunPath Solar  
**Industry:** Residential solar — Facebook lead gen to appointment booking  
**Tone:** Blend of friendly/casual + professional/consultative — warm enough that the homeowner doesn't feel pitched, professional enough that they trust it's a legitimate company, consultative in that the AI asks questions and listens rather than pushes.

### Terminology — Use
- "Solar consultation" or "solar assessment" — what we're booking (not "sales meeting")
- "Solar advisor" — what we call the rep the lead will meet with
- "Going solar" — common homeowner phrase, sounds natural in conversation
- "Utility bill" — industry standard (interchangeable with "electricity bill")
- "Qualify for solar" — correct framing for the qualification call
- "Energy savings" — acceptable in general terms, never with specific dollar amounts

### Terminology — Avoid
- "Free solar," "no cost solar," "government program" — FTC flags all as deceptive
- Technical jargon on the qualification call: kWh, inverters, net metering, arrays — these belong in the sales consultation, not the initial call
- Specific savings amounts ("you'll save $200 a month")
- Specific tax credit percentages as guaranteed (can say "federal incentives may be available")
- Pressure tactics or fake urgency ("only 3 spots left," "offer expires today")

### Compliance Rules — The AI Cannot
1. Guarantee savings amounts or bill reduction percentages
2. Claim government affiliation or say this is a government program
3. Say solar is "free" or "no cost to you"
4. Promise a specific ROI or payback period
5. Quote federal tax credit percentages as guaranteed
6. Use fake urgency or pressure tactics
7. Ignore an opt-out — if the lead says "remove me" or "don't call again," the call ends immediately

### TCPA Context
Lead opted in by submitting the Facebook Lead Ad form. For real deployment, the form must include explicit consent language for automated/AI calls from SunPath Solar specifically.

### Correct Framing
Always frame as "find out if you qualify" — never "you qualify." The AI is a qualifier and scheduler, not a closer. It does not confirm eligibility — it books a consultation so a human solar advisor can do that.

---

## Qualification Criteria (Lead must pass all 3)

1. **Homeowner** — do they own their home (not renting)?
2. **Own roof** — is the roof part of their property (not shared building/HOA-controlled)?
3. **Electricity bill** — monthly electric bill of $100 or more?

If all 3 = yes → book with solar rep  
If any = no → politely disqualify and end call  
If voicemail → no booking attempt, agent hangs up  

---

## Call Flow Scope

**Trigger:** GHL webhook fires when new contact added (simulating Facebook Lead Ad form submission)

### Attempt Sequence

| Attempt | Timing | Hours restriction |
|---|---|---|
| Call 1 | T+2 minutes after form submission | None — fires any time |
| Call 2 (if Call 1 unanswered) | T+7 minutes (5 min after no-answer confirmed) | None — fires any time |
| SMS (if Call 2 also unanswered) | Immediately after Call 2 no-answer confirmed | — |
| Call 3 / 12-hour follow-up | 12 hours after Call 2 fails | Calling window only (8am–9pm local) |

**Call 2 suppression logic:** Before Call 2 fires, n8n reads GHL `lead_status`. If it is no longer `no_answer_call_1` (lead called back, connected, opted out in the interim), the workflow exits without dialing. This prevents a pre-scheduled retry from firing when the lead has already been reached.

**Calling window enforcement:** Call 3 is triggered by a GHL workflow watching for `lead_status = no_answer_call_2`. GHL holds the trigger until the calling window opens (8am–9pm, lead's local time). *(Window hours adjustable per client.)*

### Call Outcomes — Conversation (answered calls)

| Outcome | AI action | GHL update | Notification |
|---|---|---|---|
| `qualified_booked` | Confirmed all 3 criteria, booked solar consultation | Update lead_status, write call note | None |
| `not_qualified` | Lead failed homeowner, roof, or bill criteria | Update lead_status, note includes which criterion failed | Slack + Gmail |
| `callback_requested` | Lead requested callback, AI collected requested time | Update lead_status, note includes requested time | Slack + Gmail |
| `wrong_person` | Someone else answered | Update lead_status, write call note | Slack + Gmail |
| `no_meaningful_conversation` | Connected but lead disengaged, hostile, or hung up | Update lead_status, write call note | Slack + Gmail |
| `do_not_call` | Lead said "remove me" / "don't call again" — AI ends call immediately | Update lead_status = do_not_call | None |

### Call Outcomes — No Answer / Voicemail (fast path, no EOCR needed)

| Event | GHL update | Next action |
|---|---|---|
| Call 1 no-answer or voicemail | `lead_status = no_answer_call_1` | n8n Wait 5 min → check GHL → fire Call 2 if still unreached |
| Call 2 no-answer or voicemail | `lead_status = no_answer_call_2` | Send Telnyx SMS, GHL workflow queues 12-hour Call 3 |
| Call 3 no-answer or voicemail | `lead_status = exhausted` | Slack + Gmail — all attempts used, notify human |

### Inbound Callback

If the lead calls back at any point during the attempt sequence, the inbound handler serves the AI assistant with qualification and booking flow. The inbound connection updates `lead_status` and `call_in_progress`, suppressing any pending outbound retry.

---

## Dynamic Variables (injected per-call)

| Variable | Source | Purpose |
|----------|--------|---------|
| `{{first_name}}` | GHL contact (from form fill) | Used for conversational address throughout the call |
| `{{last_name}}` | GHL contact (from form fill) | Used for identity confirmation at call start |
| `{{lead_phone}}` | GHL contact | Call target (used by Telnyx) |
| `{{ telnyx_current_time \| date: "%A, %B %d, %Y at %I:%M %p %Z", "America/New_York" }}` | Telnyx built-in — Liquid filter applied inline in prompt | Time awareness, Eastern timezone |
| `{{telnyx_end_user_target}}` | Telnyx built-in | The lead's phone number (E.164 format) |
| `{{soonest_availability}}` | Returned by check_availability_sunpath tool | Soonest open consultation slot — AI must use exact phrasing: "We have availability on {{soonest_availability}}" |
| `{{one_week_availability}}` | Returned by check_availability_sunpath tool | Full availability for the next 7 days — used if soonest slot does not work for the lead |

---

## n8n Instance

- **URL:** `https://your-n8n-instance.example.com`
- **API key:** In `.env` as `N8N_API_KEY`

---

## Workflows Required

| Workflow | Trigger | Function |
|---|---|---|
| Outbound Dialer | GHL webhook (new lead) | Receives lead data, schedules Call 1 at T+2min via Telnyx Scheduled Events API |
| Call Status Handler + EOCR | Telnyx call status webhook | Handles in-progress (sets call_in_progress); handles no-answer (5-min Wait → GHL check → Call 2 gate → SMS if Call 2 also fails); routes completed calls through EOCR → GHL update + notes + notifications |
| TeXML Router + Inbound Handler | Telnyx TeXML webhook | Routes inbound vs outbound; detects voicemail (AnsweredBy = machine_start); serves inbound AI assistant for callbacks |
| Dynamic Variable Webhook | Telnyx per-call webhook | Returns lead_name and contact context to both outbound and inbound assistants |
| Availability & Book | AI Tool webhook | Checks GHL calendar and books solar consultation |
| Call 3 Dialer | GHL webhook (12-hour delay trigger) | Fires scheduled Call 3 after GHL enforces calling window |

---

## Client Values — To Be Collected

| Placeholder | Value | Status |
|------------|-------|--------|
| `{{telnyx_phone_number}}` | `+12125550105` | ✅ |
| `{{telnyx_account_sid}}` | `00000000-0000-4000-8000-000000000001` | ✅ |
| `{{outbound_assistant_id}}` | `YOUR_TELNYX_OUTBOUND_ASSISTANT_ID` | ✅ |
| `{{inbound_assistant_id}}` | `YOUR_TELNYX_INBOUND_ASSISTANT_ID` | ✅ |
| `{{texml_app_sid}}` | `YOUR_TEXML_APP_ID` | ✅ |
| `{{ghl_location_id}}` | `YOUR_GHL_LOCATION_ID` | ✅ |
| `{{calendar_id}}` | `YOUR_GHL_CALENDAR_ID` | ✅ |
| `{{staff_user_id_1}}` | `YOUR_GHL_USER_ID_1` (Alex Monroe — Solar Advisor) | ✅ |
| `{{staff_user_id_2}}` | `YOUR_GHL_USER_ID_2` (Jordan Hayes — Solar Advisor) | ✅ |
| `{{staff_user_id_3}}` | `YOUR_GHL_USER_ID_3` (Marcus Webb — Senior Solar Advisor) | ✅ |
| GHL Auth | n8n credential (already configured) | ✅ — use n8n credential on all GHL HTTP Request nodes, no raw API key needed |
| Slack | n8n credential (already configured) | ✅ — use n8n credential on Slack nodes |
| Gmail | n8n credential (already configured) | ✅ — use n8n credential on Gmail nodes |
| `{{field_id_call_in_progress}}` | `YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS` | ✅ |
| `{{field_id_lead_status}}` | `YOUR_GHL_FIELD_ID_LEAD_STATUS` | ✅ |
| `{{field_id_solar_qualified}}` | `YOUR_GHL_FIELD_ID_SOLAR_QUALIFIED` | ✅ |
| `{{n8n_instance_url}}` | `https://your-n8n-instance.example.com` | ✅ |

---

## GHL Custom Fields Required (Phase 3)

| Field Name | Type | Values | Purpose |
|-----------|------|--------|---------|
| `call_in_progress` | Single line / yes-no | yes / no | Bridges gap between webhook executions |
| `lead_status` | Single line | pending / no_answer_call_1 / no_answer_call_2 / qualified_booked / not_qualified / callback_requested / wrong_person / no_meaningful_conversation / do_not_call / exhausted | Tracks call outcome across all attempts |
| `solar_qualified` | Single line | yes / no | Flags leads who passed all 3 criteria |

---

## GHL Trigger

- **Event:** New contact added to sub-account (simulating Facebook Lead Ad form submission)  
- **Action:** Trigger → fire outbound dialer webhook in n8n

---

## Notifications

| Trigger | Channel |
|---|---|
| not_qualified | Slack + Gmail |
| callback_requested | Slack + Gmail |
| wrong_person | Slack + Gmail |
| no_meaningful_conversation | Slack + Gmail |
| Call 3 no-answer / exhausted | Slack + Gmail |

Both channels use n8n credentials already configured in the instance — no raw URLs or email addresses needed in workflow config.

---

## Timezone

- **Timezone:** US/Eastern (EST/EDT) — company based in the US
