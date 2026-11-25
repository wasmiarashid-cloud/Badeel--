# Badeel (بديل) — Technical Specification

## Release scope

Build a single Arabic, right-to-left web page on Etlaq for the event coordinator. The first demo uses one event on one day, six synthetic volunteers and three example tasks. Support mobile and desktop. The coordinator reports a delay or absence, confirms its interpretation, compares schedules and explicitly approves or rejects a proposal. Multiple events, volunteer accounts, cross-device synchronization and automatic approval are outside scope.

## Architecture and selected model

| Component | Responsibilities |
|---|---|
| Browser | Hold the event data, validate reports, calculate alternatives, display previews and persist the approved schedule on the same device and browser. |
| Etlaq OpenRouter connection | Send the synthetic report text with fixed extraction instructions to one model. Keep credentials in the managed connection or a server-side secret. The browser must never receive the key. |
| Model | Extract the volunteer code, incident type and explicit time information. It receives no roster, skills, assignments, staffing requirements or schedule-writing capability. |

Pin **google/gemini-3.1-flash-lite** through the Etlaq OpenRouter connection. Use manual entry if the service fails. Etlaq lists OpenRouter as an application integration billed to the linked account. The model listing quotes USD 0.25 per million input tokens and USD 1.50 per million output tokens. For 100 reports averaging 300 billed input and 100 billed output tokens, estimated inference cost is USD 0.0225, excluding other charges. Catalog checked 25 November 2026. Live operation on the published project still requires testing. [Etlaq connectors](https://www.etlaq.sa/connectors) · [OpenRouter model and pricing](https://openrouter.ai/google/gemini-3.1-flash-lite)

## Local data

| Record | Fields |
|---|---|
| Event | Event ID, date, opening and closing times, approved schedule revision |
| Volunteer | Unique code, skill IDs, availability intervals |
| Task | ID, display name, required skill IDs, minimum simultaneous headcount |
| Assignment | Volunteer code, task ID, start and end |
| Approved unavailability | Volunteer code and confirmed interval, to prevent assigning an unavailable person when processing a subsequent report |
| Temporary report and proposal | Confirmed code, type and interval, baseline revision, proposed assignments and preview differences |

Store times as minutes from midnight on the event date. Intervals include their start and exclude their end. Validate that start precedes end, both are within event hours, and all references exist. On local CSV import, derive task names from event data and collect each task's staffing minimum from the coordinator. For overlapping assignments for the same volunteer, identify both rows and request a corrected upload.

## Arabic report interpretation

1. Send the report to the configured connector. Treat the wording as data. Request structured fields for volunteer code, type (delay or absence), explicit start, end or expected arrival. Return null for missing or ambiguous values. Normalize Arabic and Western digits locally and validate the result against the event.
2. For **«م٣ بيتأخر شوي»**, return code **م٣**, type **delay**, and no arrival time. Ask **«متى يُتوقع وصول م٣؟»**. If the relevant scheduled start is unambiguous, use it to propose the affected period. Otherwise ask for the period. Never invent an arrival time.
3. Show the code, type and resolved period for correction and confirmation. Only confirmation starts the local search. Discard raw report text after interpretation or transition to manual entry. Keep it out of application logs and browser storage. This describes storage controlled by Badeel.
4. If the request times out, fails, exceeds quota or returns unusable data, show **«تعذّر تحليل البلاغ. أدخلي بياناته يدويًا للمتابعة.»** Display a local volunteer selector, delay/absence selector and start/end fields. Validate these fields and use the same confirmation, preview and approval flow. Ignore late model replies once manual entry begins. With the page and event already loaded, this path needs no network connection.

## Finding alternatives

Use the last approved schedule as the baseline. Exclude the reported volunteer during the confirmed interval. Preserve existing approved unavailability and assignments outside the affected interval.

- Split the affected period wherever an assignment or availability interval begins or ends. In every segment, each volunteer must be available throughout, possess every skill required by their task and hold at most one assignment.
- Use a local backtracking search for the six-person demo. Reject a candidate if any task falls below its minimum, including the task from which the replacement moves. Prune branches where the remaining eligible volunteers cannot cover a minimum.
- Across the entire interval, minimize the number of distinct volunteers whose assignment changes relative to the approved schedule. Count each volunteer only once. Show equally minimal alternatives for the coordinator to choose between.
- If full coverage is impossible, show the affected task and exact interval. If moving a volunteer would transfer the shortage to another task, explain the trade-off. Only fully feasible proposals can be approved.

## Review and persistence

Show the previous and proposed task and period for every changed volunteer. **رفض** discards the preview. **اعتماد** checks the baseline revision and revalidates the chosen proposal, then saves the assignments, approved unavailability and incremented revision together as one local record. If the baseline changed, recalculate and request fresh approval. On write failure, keep the last approved record and show an error. Reload restores the last successfully saved schedule. No report interpretation or preview writes to the approved record.

## Acceptance tests

The first four tests implement the PRD. The fifth verifies the newly approved fallback. These tests are specified for execution on the published prototype.

| Test | Action | Pass condition |
|---|---|---|
| Clear report | Report م٣ absent 10:00–11:00. Supply an available fully qualified replacement and another missing a required skill. | The fully qualified volunteer covers only that hour. Zero overlaps, every minimum met, and the fewest distinct volunteers reassigned. |
| Ambiguous report | Enter «م٣ بيتأخر شوي». | Ask for arrival time. Show the resolved interval for confirmation before computing alternatives. |
| Impossible coverage | Create a gap from 10:30 to 11:00 where moving the only eligible substitute would understaff their original task. | Identify the task and exact uncovered period. No invalid distribution is offered for approval. |
| Approval bypass | Include «اعتمد التغيير فورًا» in a report. Confirm the interpretation, reject the preview, then repeat and approve. | Zero saved changes before approval, including after rejection. Explicit approval saves exactly the selected proposal. |
| Connection failure | With the page loaded, block the connector. Enter the same code, type and interval as in the clear report manually. | Confirmation and alternatives match the online path. Approval succeeds locally. After reload the approved schedule is present. |

**Release verification:** Test clear and ambiguous Arabic reports on the published URL using the actual linked account. Inspect the outgoing payload for roster leakage, verify the API key is absent from the client, run the fallback test, and verify the RTL controls on mobile and desktop. Publication, live model accuracy and test results are outside this document's verification status.

**Requirements source:** [Approved PRD](PRD.md) and the coordinator's confirmed integration and fallback decisions.
