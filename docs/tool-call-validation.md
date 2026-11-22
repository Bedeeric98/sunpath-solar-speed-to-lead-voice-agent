# Tool-Call Parameter Validation — SunPath Solar Instance

> Tool-call validation for SunPath Solar's outbound and inbound speed-to-lead qualifier. This file contains what's specific to this build: the Part 2 worksheet answers, the actual field-by-field validation rules, and the planned node implementation.

---

## Part 2 worksheet, answered for SunPath's tools

**1. Every caller-triggerable tool, and every LLM-originated field in its schema** (pulled directly from live Telnyx config via `retrieve_ai_assistants`, 2026-09-04, not from the prompt's prose):

| Tool | Field | Source | Caller-influenced? |
|---|---|---|---|
| `check_availability_sunpath` | *(none — `soonest_availability`/`full_week_availability` are the tool's RESPONSE fields, not request fields the LLM sends)* | — | No — this tool has no request-side fields at all. `Set Calendar Search Parameters` is fully hardcoded (calendarId, date window, timezone) with zero caller input. **Not in scope for Step 4** — confirmed by reading the live node config, not assumed. |
| `book_appointment_sunpath` | `first_name` | LLM, echoing the lead's name | Yes |
| `book_appointment_sunpath` | `last_name` | LLM, echoing the lead's name (marked optional in schema) | Yes |
| `book_appointment_sunpath` | `phone_number` | LLM, "full international or local format" per schema — no format enforced by Telnyx | Yes |
| `book_appointment_sunpath` | `absolute_time` | LLM, "ISO timestamp of the chosen appointment slot" per schema — no format enforced by Telnyx | Yes |

**Confirmed via the live n8n workflow (`n8n_get_workflow`, `Set Book Parameters`/`Search Contact For Booking`/`Create Appointment` nodes) that none of these four fields are currently validated anywhere in the chain** — `phone_number` goes straight into a GHL contact search `contains` filter, `absolute_time` goes straight into `startTime`/`endTime` on the appointment create call, with no format or window check at any point.

**2. What does "structurally valid" mean for SunPath's business?**
- `first_name`/`last_name`: a real human name — short (say, 1-50 characters), no digits, not sentence-shaped.
- `phone_number`: parses as a valid phone number (10-11 digits after stripping formatting, matching US/NANP shape — SunPath serves US callers).
- `absolute_time`: parses as a real ISO timestamp, AND falls within a sane booking window — not in the past (even a few minutes' grace for clock skew), and not more than the calendar's own actual search window would ever return (the calendar search itself only looks 7 days out — `Set Calendar Search Parameters`' `end_date` is `+7 days` — so any `absolute_time` more than, generously, 30 days out is already structurally impossible given what the tool could have offered; the wide margin over the 7-day search window is deliberate, since a caller could name their own specific time per the prompt's `check_availability_sunpath` tool-usage rule, not just pick from what was read aloud).

**3. Shared worksheet question 4's answer, reused from `transcript-red-flag-scanning.md`:** the highest-harm category for this agent type is a Compliance-rule breach (a spoken guarantee) — that's what Steps 1-2 already guard. Step 4's fields (name, phone, appointment time) carry a *different*, lower-drama harm: a broken/garbage GHL record (an appointment with an invalid start time, a contact record with a corrupted phone number) — an operational failure, not a legal-exposure one. This means Step 4's fields warrant reject-and-recover (let the LLM try again on the call), not an admin Slack alert on every rejection — alerting on every malformed value would be noise, not signal, given STT mishears are a completely ordinary, expected source of a bad first attempt.

**4. What happens on validation failure, per field:**
- All four fields: **hard-reject the tool call**, return a structured error `book_appointment_sunpath` can react to conversationally (the prompt's existing "don't hallucinate... tell the lead a solar advisor will reach out" pattern already establishes this tool has a defined failure-handling path to extend).
- **No admin notification on a single rejection** — an isolated bad value is expected (STT mishears a phone number digit, a date computation drifts). Reserve escalation for a pattern: see "Future consideration" below.

---

## Final validation rules — as planned for the Code node

```js
// Validate Book Appointment Parameters (Code node — JavaScript, "Run Once for Each Item" mode)
// Runs after Set Book Parameters, before Search Contact For Booking.
// Structural validation only — not a semantic/intent check.

const errors = [];

// first_name / last_name — sane human-name shape
function isSaneName(value, fieldName, required) {
  if (!value || typeof value !== 'string' || value.trim().length === 0) {
    if (required) errors.push({ field: fieldName, reason: 'missing or empty' });
    return;
  }
  const trimmed = value.trim();
  // Guard against the literal string "null"/"undefined" — a known LLM/webhook failure
  // shape (an omitted optional field sometimes arrives as the word itself rather than
  // truly empty).
  if (/^(null|undefined|n\/a|none)$/i.test(trimmed)) {
    if (required) errors.push({ field: fieldName, reason: 'literal "null"/"n/a" string, not a real value' });
    return;
  }
  if (trimmed.length > 50) {
    errors.push({ field: fieldName, reason: 'exceeds 50 characters — not a plausible name' });
  }
  if (/\d/.test(trimmed)) {
    errors.push({ field: fieldName, reason: 'contains digits — not a plausible name' });
  }
  // A real name is not sentence-shaped — reject anything containing multiple spaces in a
  // pattern suggesting a full sentence (3+ words AND a terminal punctuation mark) rather
  // than "First Last" or "First Middle Last". Deliberately generous — real names do
  // sometimes have 3+ words (e.g. hyphenated or multi-part surnames handled as spaces).
  const wordCount = trimmed.split(/\s+/).length;
  if (wordCount > 4 || /[.!?]$/.test(trimmed)) {
    errors.push({ field: fieldName, reason: 'sentence-shaped value, not a plausible name' });
  }
}
isSaneName($json.first_name, 'first_name', /* required */ true);
isSaneName($json.last_name, 'last_name', /* required */ false); // schema marks this optional

// phone_number — must parse as a plausible NANP (US/Canada) number, this build's locked
// geography (US callers)
function isSanePhone(value) {
  if (!value || typeof value !== 'string') {
    errors.push({ field: 'phone_number', reason: 'missing or empty' });
    return;
  }
  const digitsOnly = value.replace(/\D/g, '');
  // 10 digits (no country code) or 11 digits starting with 1 (with country code)
  const validNanp = /^1?\d{10}$/.test(digitsOnly) && digitsOnly.length >= 10 && digitsOnly.length <= 11;
  if (!validNanp) {
    errors.push({ field: 'phone_number', reason: 'does not parse as a valid US/Canada phone number' });
  }
}
isSanePhone($json.phone_number);

// absolute_time — must parse as a real ISO timestamp, must not be in the past (with a small
// grace window for clock skew / in-call latency), must not be further out than the calendar
// tool could plausibly have offered (7-day search window + generous margin for a
// caller-named specific time, per the prompt's own check_availability_sunpath tool-usage rule)
function isSaneAppointmentTime(value) {
  if (!value || typeof value !== 'string') {
    errors.push({ field: 'absolute_time', reason: 'missing or empty' });
    return;
  }
  const parsed = new Date(value);
  if (isNaN(parsed.getTime())) {
    errors.push({ field: 'absolute_time', reason: 'not a valid ISO timestamp' });
    return;
  }
  const now = new Date();
  const graceMs = 5 * 60 * 1000; // 5 minutes grace for clock skew / call latency
  if (parsed.getTime() < now.getTime() - graceMs) {
    errors.push({ field: 'absolute_time', reason: 'appointment time is in the past' });
  }
  const maxOutMs = 30 * 24 * 60 * 60 * 1000; // 30 days — generous margin over the 7-day
  // calendar search window, since a caller can name their own specific time
  if (parsed.getTime() > now.getTime() + maxOutMs) {
    errors.push({ field: 'absolute_time', reason: 'appointment time is more than 30 days out — outside any plausible offered window' });
  }
}
isSaneAppointmentTime($json.absolute_time);

const valid = errors.length === 0;

return {
  json: {
    ...$json,
    valid,
    errors,
  }
};
```

**IF node condition:** `{{ $json.valid }}` boolean-true → TRUE continues to `Search Contact For Booking` (existing chain, unchanged); FALSE → `Return Book Appointment Validation Error`.

**Error response shape** (matches the structured-error pattern from the framework doc, gives the LLM something to react to):
```json
{
  "error": "validation_failed",
  "message": "One or more appointment details could not be confirmed. Please ask the caller to repeat the affected detail(s).",
  "fields": [ /* the errors array from above */ ]
}
```

---

## Node implementation — LIVE as of 2026-09-04, verified via independent `n8n_get_workflow` re-fetch (both filtered node config and full structure/connection topology, not just the write response)

```
Set Book Parameters
  → Validate Book Appointment Parameters      [NEW — Code node]
      → If Book Parameters Valid               [NEW — IF node, condition: $json.valid]
           TRUE  → Search Contact For Booking   (existing, unchanged)
                 → Create Appointment           (existing, unchanged)
                 → Return Book Confirmation     (existing, unchanged)
           FALSE → Return Book Appointment Validation Error   [NEW — respondToWebhook, structured error]
```

**`check_availability_sunpath` needs no equivalent gate** — confirmed via live node config that it has zero caller-influenced request fields (see worksheet question 1 table above). Building a validation node for it would have nothing real to check.

**No admin notification node added** — per worksheet question 4's answer, an isolated rejection is expected/routine (STT mishears), not alert-worthy. See "Future consideration" below for the one case that might warrant revisiting this.

**Field source:** `$json.first_name`, `$json.last_name`, `$json.phone_number`, `$json.absolute_time` — all already present on `Set Book Parameters`' output (the existing node reads them straight from `$json.body.*`), so the validation node needs no new upstream data, just inserts one step after where these fields already land.

---

## Status

**LIVE as of 2026-09-04.** Node count 224→227 (+3: Code node, IF node, error-response node), connection count 163→173. Confirmed via two independent post-write re-fetches: a filtered fetch of all 5 relevant nodes (matching config/jsCode exactly) and a full structure-mode fetch confirming the connection topology matches the plan exactly (`Set Book Parameters` → `Validate Book Appointment Parameters` → `If Book Parameters Valid` → TRUE into the existing chain unchanged, FALSE into the new error-response node). `check_availability_sunpath`'s chain confirmed untouched.


