---
title: Help desk Zscaler triage dashboard and KBA notebook
type: design
scope: tenant
status: draft
updated: 2026-10-20
sources:
  - https://help.zscaler.com/zpa/understanding-user-activity-log-fields
  - https://help.zscaler.com/zpa/about-user-status-log-fields
  - https://help.zscaler.com/zpa/zpa-session-status-codes
  - https://support.microsoft.com/en-US/Microsoft-365-Copilot/provide-custom-instructions-for-your-microsoft-365-copilot-notebook
---

# Help desk Zscaler triage dashboard and KBA notebook

The service desk has no Zscaler visibility today, so every ambiguous Zscaler Internet Access (ZIA) or Zscaler Private Access (ZPA) incident escalates to network security. A read-only Splunk dashboard scoped to one user and one time window gives the desk enough evidence to tell "blocked by policy" from "broken" and to route app faults to the app team. The KBAs drafted in the Copilot notebook depend on this dashboard for their triage steps; without it, most KBAs reduce to "collect details and escalate".

This document covers the dashboard design, the reason-code lookup that connects it to the KBAs, the checks to run before building, and the full Copilot notebook instructions.

## Current state and open assumptions

**Observed:** the service desk has no Zscaler admin portal or Splunk access.

Nothing else below is confirmed yet. Each assumption needs an owner and a test before the design is treated as decided.

| Assumption | Owner | How to test |
| :--- | :--- | :--- |
| ZPA User Activity and User Status logs are streamed to Splunk via the Log Streaming Service (LSS) | Splunk admin | Feed inventory search below |
| ZIA web logs are in Splunk | Splunk admin | Feed inventory search below. If absent, panel 5 is dropped. |
| Feed latency is low enough for live triage (p95 under ~5 minutes) | Splunk admin | Latency search below |
| Timestamps are parsed in the correct timezone | Splunk admin | Latency search below: lag near 36,000 or 39,600 s means a timezone fault |
| The ZPA `Username` value matches the user's UPN | Network security | Search ten known users by UPN and compare |
| Help desk access to ZIA block records is covered by the workplace monitoring notice | Privacy / HR | Written confirmation before go-live |
| A summary index can restrict the desk to minimised fields | Splunk admin | Build in non-production, confirm the desk role cannot search raw indexes |

## Design

### Shape: single-user lookup, not a monitoring wall

The dashboard answers five questions about one user in one time window. Fleet-wide charts are out of scope: the desk cannot act on them, and an always-red panel teaches people to ignore the dashboard.

Inputs:

- **User UPN** (required).
- **Time window**: default last 60 minutes, capped at 24 hours.
- **App or host** (optional).

All times display in AEST/AEDT, with the timezone shown on the dashboard.

### Panels

| Panel | Question it answers | Source | What the desk does with it |
| :--- | :--- | :--- | :--- |
| 1. Feed freshness | Is the data recent enough to trust an empty result? | Latest event time per feed | If stale, nothing below is evidence. Wait or escalate. |
| 2. ZPA sign-in | Is this user authenticated to ZPA, and on which Zscaler Client Connector (ZCC) version? | ZPA User Status log | Auth failed or no recent status → ZCC/identity KBA, not an app KBA. |
| 3. Private app attempts | Did ZPA see the request? Was it blocked, errored or fine? | ZPA User Activity log + reason-code lookup | Policy block → access request process. App Connector error → network escalation. Normal session → fault is past ZPA, route to app team. |
| 4. Who else is affected | Is it only this user? | ZPA User Activity, same app, last 15 minutes, distinct users succeeding vs failing | Many failing → attach to the major incident and stop single-user triage. |
| 5. Internet blocks | Did ZIA block or caution this destination, and under which category or rule? | ZIA web log, blocked and cautioned transactions only, domain not full URL | Category block → exception request. No block → not ZIA policy. |

Panels 3 and 4 do most of the deflection. A normal ZPA session with a named App Connector reaching the server is strong evidence the fault sits past ZPA; today those tickets land in the network security queue by default. Panel 4 stops one outage becoming dozens of parallel single-user investigations.

**[unverified]** ZIA web log field names for action, URL category and rule are not confirmed for this Splunk deployment. Take them from the feed inventory, not from memory.

### Session outcome comes from InternalReason, not ConnectionStatus

Zscaler's User Activity log reference lists `ConnectionStatus` values as Open, Close and Active. That field records connection state, not whether access succeeded. A denied session ends as an ordinary "Close". The outcome is carried in `InternalReason`.

A panel built on `ConnectionStatus` shows policy denies as normal sessions, and the desk tells the user ZPA is fine. This mistake has shipped in at least one third-party SIEM parser, which later switched its block/allow logic to `InternalReason`.

### Reason-code lookup

The lookup translates `InternalReason` codes into desk language and is the bridge between the dashboard and the KBAs. Zscaler groups session status codes into Error, Info and Policy Block types, prefixed by component (App Connector, Client Connector, Service Edge, Browser Access). The lookup mirrors that grouping.

| Column | Holds |
| :--- | :--- |
| `InternalReason` | The raw reason code |
| `category` | Policy block, Error or Info |
| `component` | Which component the code is about |
| `desk_message` | One sentence in plain English |
| `kba_link` | The KBA the desk should follow |
| `escalation_group` | Who gets the ticket |

Rules:

- Any code not in the lookup displays as **"Unmapped, escalate"**. It never displays blank.
- The lookup is authoritative. KBAs cite it rather than restating codes, so the two cannot drift.
- Network security owns the lookup. Re-run the reason-code inventory monthly and add any new codes.

**[unverified]** `BRK_MT_SETUP_FAIL_REJECTED_BY_POLICY` (an explicit deny matched) and `BRK_MT_SETUP_FAIL_NO_POLICY_FOUND` (no matching allow rule) are described this way in third-party material only. Confirm against the Zscaler session status codes page before writing desk wording. The distinction matters once segmentation is enforced: the first is a rule working as designed, the second may be a gap in the design.

## What breaks if you get this wrong

**Empty panels read as "all fine".** No rows can mean the user never reached Zscaler, the feed is stale, or the search used the wrong username. Panel 1 exists for this reason. Every other panel states "no events found for this user in this window" rather than rendering blank.

**The username may not match.** Zscaler describes `Username` as the name entered into ZCC. If that is an email alias rather than the UPN, a desk search by UPN returns nothing, which looks identical to "never reached ZPA". Normalise case and test with real accounts.

**Timezone parsing.** The User Activity field reference does not state the timezone of the log timestamp. If the Splunk add-on parses it wrongly, every result is offset by 10 or 11 hours and the desk searches the wrong window.

**ZIA web logs are browsing history.** Read access to the raw ZIA index lets the desk search any user's browsing.

**Blast-radius false comfort.** Panel 4 is weak for low-use apps. Three users failing out of three is an outage; the panel must show the counts, not a red/green verdict.

## Access and privacy

1. A scheduled search runs every five minutes and writes minimised fields to a summary index: user, time, app or domain, action, category, rule, ZPA reason code. ZIA rows are limited to blocked and cautioned transactions.
2. A dedicated desk role has read access to that summary index only.
3. The dashboard reads only from the summary index.

Trade-off: up to five minutes of extra delay on top of feed latency. Panel 1 must show the summary index freshness, not the raw feed freshness.

**[unverified]** Role design and summary indexing approach to be confirmed with the Splunk admin.

## Before building: confirm current state

Run these three searches before designing any panel.

### Feed inventory

```spl
// Question:  Which Zscaler feeds are in Splunk, and how recent is the newest event?
// Expects:   One row per index/sourcetype. A missing ZPA User Status or ZIA web
//            feed removes panel 2 or 5 from the design.
// Caveats:   Sourcetype wildcards are guesses; widen if nothing returns.
// Updated:   2026-10-20
| tstats count max(_time) as last_event
    where index=* (sourcetype=*zscaler* OR sourcetype=*zpa* OR sourcetype=*nss*)
    by index sourcetype
| eval minutes_since_last=round((now() - last_event) / 60, 1)
```

### Feed latency

```spl
// Question:  How long after an event can the desk see it?
// Expects:   p95 under a few minutes. ~36000 or ~39600 s = timezone parse fault.
// Index:     <zpa_index> (placeholder, take from feed inventory)
// Updated:   2026-10-20
index=<zpa_index> earliest=-4h
| eval lag_s = _indextime - _time
| stats count p50(lag_s) as p50_lag_s p95(lag_s) as p95_lag_s max(lag_s) as max_lag_s
    by sourcetype
```

### Reason-code inventory

```spl
// Question:    Which InternalReason codes occur here, and how often?
// Answers for: Seeding the reason-code lookup; ranking the KBA backlog;
//              segmentation validation after enforcement.
// Index:       <zpa_index>, <user_activity_sourcetype> (placeholders)
// Updated:     2026-10-20
index=<zpa_index> sourcetype=<user_activity_sourcetype> earliest=-30d
| stats count dc(Username) as users by InternalReason
| sort - count
```

The reason-code inventory is a better basis for the KBA backlog than help desk ticket categories, which over-count Zscaler incidents. It is also the validation search for segmentation: counts of policy-block codes per segment after enforcement. File it under `splunk/queries/`.

## Acceptance test

Replay 20 resolved Zscaler-related tickets through the dashboard before handing it to the desk. For each, record whether the dashboard would have sent the desk to the correct KBA and team. Any wrong routing blocks go-live.

## Copilot notebook instructions

Paste into the notebook's Copilot instructions dialog (expand arrow next to the notebook name → Copilot instructions). **[unverified]** The instruction limit is reported as 8,000 characters by a third party; this text is kept well under it. Anyone with edit access to a shared notebook can change these instructions.

Add as notebook references: this document, the reason-code lookup, the dashboard user guide, a resolved-incident export (state its window), the escalation and assignment-group process, the app inventory (private vs SaaS), and the Zscaler help pages matching the deployed ZCC version.

```text
ROLE
You draft knowledge base articles (KBAs) that let service desk analysts triage Zscaler Internet Access (ZIA) and Zscaler Private Access (ZPA) incidents, resolve what is within their permissions, and escalate the rest with a complete evidence package. The goal is fewer, better tickets reaching the network security queue, not zero tickets. A KBA that has the desk mask a fault is worse than no KBA.

AUDIENCE
Service desk analysts: competent with Windows, M365 and remote support; no Zscaler admin access or training. Write for someone on a call with a frustrated user. Australian English, plain words, short sentences, no vendor marketing.

ACCURACY
1. Ground every technical claim in the notebook references. Priority: our internal docs and resolved incident records, then Zscaler help portal pages, then Microsoft docs for Entra ID and Windows.
2. Never invent menu paths, error codes, log fields, Zscaler Client Connector (ZCC) settings, ports or URLs. If a step needs one and the references don't contain it, write [VERIFY: what needs confirming] in its place.
3. When explaining causes, label each as Observed (in a ticket or log), Inferred, Assumed, or Confirmed (vendor or internal doc). Never present an inferred cause as the cause.
4. Tenant-specific details (hostnames, IdP groups, connector names, IP ranges, cloud name) come only from references, never general knowledge.
5. Strip personal data from incident examples: no names, emails, device names or ticket numbers in the KBA body.

CLASSIFY FIRST
Before drafting, state:
- Product path: ZIA (internet/SaaS), ZPA (private app), ZCC itself, identity (Entra ID/SAML), or not Zscaler. State how the desk tells these apart from what the user sees. Never assume the user knows.
- Whether the symptom can be intended policy working as designed. A ZPA deny may present as a connection failure rather than a block page. Every KBA must give the desk a way to tell "blocked by policy" from "broken", or tell them to escalate when they can't.
If the references don't support a confident classification, list what's missing instead of drafting.

SERVICE DESK BOUNDARY
The desk CAN: view ZCC status on the user's device; use the "Zscaler Triage" Splunk dashboard (read-only, single user lookup) as described in the dashboard reference document.
The desk CANNOT: access Zscaler admin portals, change policy, add bypasses, disable ZCC.
Only write steps inside CAN; anything else becomes an escalation step. Never write a step that disables, uninstalls, bypasses or logs out of ZCC, or has the user switch networks or VPNs to "test", unless a reference shows it is approved. If it is, include the revert step.
When a triage step uses the dashboard, name the panel and state what each possible result means and where it leads. Use the reason-code lookup for ZPA outcomes; never interpret reason codes yourself. An empty panel is never evidence of success: always check the Feed freshness panel first and treat stale data as "unknown".

KBA STRUCTURE (these headings, in order)
Title: the user's symptom in their words plus product, e.g. "Internal app 'can't reach this page' from home (ZPA)".
Metadata: Product | Type: Troubleshooting | Status: Draft | Owner: {{owner}} | Last reviewed: DD/MM/YYYY | Review trigger | Sources
Applies to / Does not apply to: scope, plus look-alike symptoms this KBA does not cover and where they go.
Symptoms: exact error text, block page wording and ZCC status, as the user sees them.
Quick checks (under 2 minutes): rule out common non-Zscaler causes first.
Triage: numbered steps, one action each. Every step states the expected result and what to do otherwise (go to step N, resolve, or escalate).
Resolution: only fixes inside the desk boundary, each with how to confirm it worked.
Escalate when: explicit conditions. Never "if the issue persists".
Escalation package: fields the desk must capture. Time of failure with timezone stated; user UPN; device name; network (office, home, hotspot); ZCC version and per-service status; app URL or FQDN and port if known; full error text including any reference or transaction ID shown; dashboard panel results; steps tried and results; ZCC logs if applicable. Drop fields that don't apply to the product path.
Do not: actions that worsen or hide the problem.
Sources: references used.
Gaps: every [VERIFY] item and who should confirm it.

STYLE
- Verb first in steps. No hedging: if uncertain, mark [VERIFY] and keep the sentence direct.
- One KBA per symptom. If a draft needs two triage paths, split it and say so.
- Explain "why" in one sentence only where it changes what the analyst does.
- Dates DD/MM/YYYY. Times AEST/AEDT with offset, or UTC stated explicitly.

AFTER EVERY DRAFT
Add "Reviewer notes" (not part of the KBA): the weakest step and why, assumptions made, any step that could hide a real fault, and what change (policy, ZCC version, ZPA segmentation, dashboard or lookup change) would make this KBA wrong.
When asked for a KBA backlog, rank candidates by incident volume in the references and state the basis (source, period, count). If there's no volume data, say the ranking is a guess.
```

## Gaps

| Item | Who confirms |
| :--- | :--- |
| Meaning of `REJECTED_BY_POLICY` vs `NO_POLICY_FOUND` codes | Network security, against Zscaler session status codes page |
| ZIA web log field names in this Splunk deployment | Splunk admin, from feed inventory |
| Timezone of LSS timestamps and Splunk parsing | Splunk admin, from latency search |
| Summary index and role restriction approach | Splunk admin |
| Copilot notebook instruction character limit | Notebook owner, by testing in the dialog |
| Privacy cover for desk access to block records | Privacy / HR |

## See also

- Zscaler: Understanding User Activity Log Fields — https://help.zscaler.com/zpa/understanding-user-activity-log-fields
- Zscaler: About User Status Log Fields — https://help.zscaler.com/zpa/about-user-status-log-fields
- Zscaler: Understanding ZPA Session Status Codes — https://help.zscaler.com/zpa/zpa-session-status-codes
- Microsoft: Provide custom instructions for your Microsoft 365 Copilot Notebook — https://support.microsoft.com/en-US/Microsoft-365-Copilot/provide-custom-instructions-for-your-microsoft-365-copilot-notebook
