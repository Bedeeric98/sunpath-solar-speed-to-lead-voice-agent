# SunPath Solar — Notification Templates

Slack credential: your n8n Slack credential
Slack user __rl: `{"__rl": true, "value": "YOUR_SLACK_USER_ID", "mode": "list"}`
Gmail field: `sendTo` — use n8n Gmail credential, typeVersion 2.2
Admin email: `user@example.com`

Node name references below use the SunPath node naming convention.
Update if node names change during build.

---

**Updated 2026-09-02:** the correct field is `call_summary`, not `telnyx_summary` — the field name in this file was wrong for every outcome from the start; fixed throughout below. `qualified_booked` and `qualified_not_booked` templates added — both branches previously had no admin notification at all (note-only), which was a real gap for the highest-value outcomes. `do_not_call` also has a Slack/Gmail template as of 2026-07-20 (previously the only branch with none, despite being the highest compliance-risk outcome).

**Updated again 2026-09-02 (inbound EOCR recovery-layer build + Test C bugfix pass):** all 7 Inbound sections below now correctly reference `Set Inbound Contact & Insights Data` instead of the stale `Get a conversation Inbound`/`Parse Call Insights Inbound EOCR` references — those two nodes don't run on the recovery path (when the `analyzed` webhook never arrives and the flow rejoins via `Parse Recovered Inbound Conversation Insights` instead), so the old direct references would throw `hasn't been executed` on any recovery-path execution. This was a real, live bug found and fixed via Test C (execution 17321). Email templates for `not_qualified`, `callback_requested`, `wrong_person`, and `no_meaningful_conversation` were also missing from this doc entirely despite being live in the workflow — added.

**Decision — no `call_no_conversation_sunpath` equivalent needed on inbound:** that flag exists on outbound to stop a voicemail-hit's bogus EOCR classification from corrupting `lead_status_sunpath` mid-retry-sequence (Call 1→2→3). Inbound has no such retry sequence to protect and no AMD (AMD detects whether the *far end* picked up as a machine on an outbound dial — meaningless when SunPath's own line is the one being called). Decision made explicit 2026-09-02 rather than left as a silent gap.

---

## Outbound — qualified_booked

**Slack:**
```
✅ *Lead Qualified & Booked — SunPath Solar*

*Name:* {{ $('Set Outbound Contact & Insights Data').item.json.first_name }} {{ $('Set Outbound Contact & Insights Data').item.json.last_name }}
*Phone:* {{ $('Get a Conversation Outbound').item.json.metadata.to }}
*Outcome:* Lead qualified and booked a solar consultation
*Summary:* {{ $('Parse Conversation Insights Outbound EOCR').item.json.call_summary }}
```

**Email Subject:** `Lead Qualified & Booked — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead qualified for a SunPath Solar consultation and booked an appointment.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead qualified and booked
Summary: {{ call_summary }}
Recording: {{ recording_url }}
```

---

## Outbound — qualified_not_booked

**Slack:**
```
⚠️ *Lead Qualified But Not Booked — SunPath Solar*

*Name:* {{ $('Set Outbound Contact & Insights Data').item.json.first_name }} {{ $('Set Outbound Contact & Insights Data').item.json.last_name }}
*Phone:* {{ $('Get a Conversation Outbound').item.json.metadata.to }}
*Outcome:* Lead qualified but the call ended before a specific appointment time was confirmed and booked
*Summary:* {{ $('Parse Conversation Insights Outbound EOCR').item.json.call_summary }}
*Action Required:* Human should follow up to complete the booking
```

**Email Subject:** `Lead Qualified But Not Booked — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead qualified for a SunPath Solar consultation, but the call ended before a booking was completed.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead qualified, booking not completed
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: A human should follow up to complete the booking.
```

---

## Outbound — not_qualified

**Slack:**
```
⚠️ *Lead Not Qualified — SunPath Solar*

*Name:* {{ $('Set Outbound Contact & Insights Data').item.json.first_name }} {{ $('Set Outbound Contact & Insights Data').item.json.last_name }}
*Phone:* {{ $('Get a Conversation Outbound').item.json.metadata.to }}
*Outcome:* Lead did not qualify for solar consultation
*Summary:* {{ $('Parse Call Insights Outbound EOCR').item.json.call_summary }}
```

**Email Subject:** `Lead Not Qualified — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead did not pass qualification for a SunPath Solar consultation.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead did not qualify — failed homeowner, roof, or bill criteria
Summary: {{ call_summary }}
Recording: {{ recording_url }}
```

---

## Outbound — callback_requested

**Slack:**
```
📞 *Callback Requested — SunPath Solar*

*Name:* {{ $('Set Outbound Contact & Insights Data').item.json.first_name }} {{ $('Set Outbound Contact & Insights Data').item.json.last_name }}
*Phone:* {{ $('Get a Conversation Outbound').item.json.metadata.to }}
*Outcome:* Lead answered but requested a callback
*Summary:* {{ $('Parse Call Insights Outbound EOCR').item.json.call_summary }}
*Action Required:* Schedule a follow-up call at the time noted in the summary
```

**Email Subject:** `Callback Requested — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead requested a callback during our outbound call.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead requested a callback — preferred time noted in summary
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: Schedule a follow-up call at the preferred time.
```

---

## Outbound — wrong_person

**Slack:**
```
⚠️ *Wrong Person Answered — SunPath Solar*

*Phone:* {{ $('Get a Conversation Outbound').item.json.metadata.to }}
*Outcome:* Someone other than the intended lead answered the call
*Summary:* {{ $('Parse Call Insights Outbound EOCR').item.json.call_summary }}
*Action Required:* Verify contact details and re-attempt
```

**Email Subject:** `Wrong Person Answered — SunPath Solar Lead`

**Email Body:**
```
Someone other than the intended lead answered our outbound call.

Phone: {{ phone }}
Outcome: Wrong person answered — intended lead was not available
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: Verify contact details and re-attempt manually if needed.
```

---

## Outbound — no_meaningful_conversation

**Slack:**
```
⚠️ *No Meaningful Conversation — SunPath Solar*

*Name:* {{ $('Set Outbound Contact & Insights Data').item.json.first_name }} {{ $('Set Outbound Contact & Insights Data').item.json.last_name }}
*Phone:* {{ $('Get a Conversation Outbound').item.json.metadata.to }}
*Outcome:* Call connected but no useful conversation took place
*Summary:* {{ $('Parse Call Insights Outbound EOCR').item.json.call_summary }}
*Action Required:* Human should reach out directly
```

**Email Subject:** `No Meaningful Conversation — {{ first_name }} {{ last_name }}`

**Email Body:**
```
Our AI agent connected with the lead but was unable to have a meaningful conversation.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Call connected but no useful conversation took place
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: A human should reach out directly to this lead.
```

---

## Outbound — do_not_call

**Slack:**
```
🚫 *Do Not Call Requested — SunPath Solar*

*Name:* {{ $('Set Outbound Contact & Insights Data').item.json.first_name }} {{ $('Set Outbound Contact & Insights Data').item.json.last_name }}
*Phone:* {{ $('Get a Conversation Outbound').item.json.metadata.to }}
*Outcome:* Lead opted out — do not contact again
*Summary:* {{ $('Parse Conversation Insights Outbound EOCR').item.json.call_summary }}
```

**Email Subject:** `Do Not Call Requested — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead requested not to be contacted again during our outbound call.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead opted out — do not contact again
Summary: {{ call_summary }}
Recording: {{ recording_url }}
```

---

## Call Status — exhausted (all 3 attempts failed)

**Slack:**
```
🔴 *All Attempts Exhausted — SunPath Solar Lead*

*Name:* {{ $('Decipher GHL Custom Fields For No Answer').item.json.first_name }} {{ $('Decipher GHL Custom Fields For No Answer').item.json.last_name }}
*Phone:* {{ $('Decipher GHL Custom Fields For No Answer').item.json.phone }}
*Status:* Called 3 times — no answer on any attempt
*Action Required:* Manual follow-up needed
```

**Email Subject:** `All Attempts Exhausted — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A SunPath Solar lead did not answer any of our three call attempts.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Status: Called 3 times — no answer on any attempt
Action Required: Manual follow-up needed — lead may need to be contacted through a different channel.
```

---

## Inbound — qualified_booked

**Slack:**
```
🎉 *Inbound Call — Qualified & Booked*

*Phone:* {{ $('Set Inbound Contact & Insights Data').item.json.phone }}
*Outcome:* Lead called back, qualified for solar consultation, and a time slot was booked
*Summary:* {{ $('Set Inbound Contact & Insights Data').item.json.call_summary }}
*Action Required:* None - appointment is on the calendar. FYI only.
```

**Email Subject:** `Inbound Call — Qualified & Booked — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead called back, was qualified, and booked a solar consultation.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead qualified and a time slot was booked
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: None - appointment is on the calendar. FYI only.
```

---

## Inbound — qualified_not_booked

**Slack:**
```
🟡 *Inbound Call — Qualified But Not Booked*

*Phone:* {{ $('Set Inbound Contact & Insights Data').item.json.phone }}
*Outcome:* Lead called back and passed all 3 qualification criteria, but the call ended before a time slot was confirmed
*Summary:* {{ $('Set Inbound Contact & Insights Data').item.json.call_summary }}
*Action Required:* Call back promptly to complete the booking - do not let a qualified lead go cold
```

**Email Subject:** `Inbound Call — Qualified But Not Booked — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead called back, was qualified, but the call ended before booking was completed.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead qualified on all 3 criteria, but the call ended before a time slot was confirmed
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: Call back promptly to complete the booking - do not let a qualified lead go cold.
```

---

## Inbound — not_qualified

**Slack:**
```
⚠️ *Inbound Call — Lead Not Qualified*

*Phone:* {{ $('Set Inbound Contact & Insights Data').item.json.phone }}
*Outcome:* Lead called back but did not qualify for solar consultation
*Summary:* {{ $('Set Inbound Contact & Insights Data').item.json.call_summary }}
```

**Email Subject:** `Inbound Call — Lead Not Qualified — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead called back but did not pass qualification for a SunPath Solar consultation.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead did not qualify — failed homeowner, roof, or bill criteria
Summary: {{ call_summary }}
Recording: {{ recording_url }}
```

---

## Inbound — callback_requested

**Slack:**
```
📞 *Inbound Call — Callback Requested*

*Phone:* {{ $('Set Inbound Contact & Insights Data').item.json.phone }}
*Outcome:* Lead called back but was busy and requested a callback
*Summary:* {{ $('Set Inbound Contact & Insights Data').item.json.call_summary }}
*Action Required:* Schedule follow-up at the time noted in the summary
```

**Email Subject:** `Inbound Call — Callback Requested — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead called back and requested a callback.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead requested a callback — preferred time noted in summary
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: Schedule a follow-up call at the preferred time.
```

---

## Inbound — wrong_person

**Slack:**
```
⚠️ *Inbound Call — Wrong Person Answered*

*Phone:* {{ $('Set Inbound Contact & Insights Data').item.json.phone }}
*Outcome:* Someone other than the intended lead answered the inbound call
*Summary:* {{ $('Set Inbound Contact & Insights Data').item.json.call_summary }}
*Action Required:* Verify contact details before re-attempting
```

**Email Subject:** `Wrong Person Answered — Inbound SunPath Solar Call`

**Email Body:**
```
Someone other than the intended lead answered an inbound call.

Phone: {{ phone }}
Outcome: Wrong person answered — intended lead was not available
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: Verify contact details and re-attempt manually if needed.
```

---

## Inbound — no_meaningful_conversation

**Slack:**
```
⚠️ *Inbound Call — No Meaningful Conversation*

*Phone:* {{ $('Set Inbound Contact & Insights Data').item.json.phone }}
*Outcome:* Lead called back but no useful conversation took place
*Summary:* {{ $('Set Inbound Contact & Insights Data').item.json.call_summary }}
*Action Required:* Human should reach out directly
```

**Email Subject:** `No Meaningful Conversation — Inbound — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead called back but no useful conversation took place.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Call connected but no useful conversation took place
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: A human should reach out directly to this lead.
```

---

## Inbound — do_not_call

**Slack:**
```
🚫 *Inbound Call — Do Not Call — SunPath Solar*

*Phone:* {{ $('Set Inbound Contact & Insights Data').item.json.phone }}
*Outcome:* Lead called back and asked to be removed from contact list — do not call again
*Summary:* {{ $('Set Inbound Contact & Insights Data').item.json.call_summary }}
*Action Required:* Confirm lead is suppressed from all future outreach
```

**Email Subject:** `Inbound Call — Do Not Call — {{ first_name }} {{ last_name }}`

**Email Body:**
```
A lead called back and asked to be removed from our contact list and should not be called again.

Name: {{ first_name }} {{ last_name }}
Phone: {{ phone }}
Outcome: Lead requested no further contact — do not call again
Summary: {{ call_summary }}
Recording: {{ recording_url }}
Action Required: Confirm this lead is suppressed from all future outreach immediately.
```
