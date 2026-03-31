# Outreach Workflow — Naturprogramme
> Single pipeline from lead → first email → follow-up → booking → referral.
> CRM source of truth: `leads/OUTREACH_CRM.xlsx` (Pipeline sheet)
> Templates: `outreach/TEMPLATES/` (one file per template type)

---

## Architecture

```
LEADS DB (CRM xlsx)          TEMPLATES/           CALENDAR
   |                             |                    |
   v                             v                    v
[1. SEGMENT] --> [2. PERSONALIZE] --> [3. SEND] --> [4. TRACK] --> [5. FOLLOW-UP] --> [6. CONVERT] --> [7. REFERRAL LOOP]
   ^                                                   |                                  |
   |                                                   v                                  v
   +------------- DAILY REVIEW (5 min) <------------- CRM STATUS CHECK <-------- POST-WORKSHOP FEEDBACK
```

---

## Step 0: Daily Review (5 min — do this EVERY work day)

**Trigger:** Open CRM xlsx, sort by `Naechste_Aktion_Datum`

**Actions:**
1. Filter `Status = Antwort_Erhalten` → respond TODAY (hot leads, 24h rule)
2. Filter any status where `Naechste_Aktion_Datum <= today` → overdue follow-ups
3. Filter `Status = Ausstehend` + `Tier = Tier 1` → candidates for new outreach

**Output:** Pick max 3 actions for today. If one action is "batch send 5 emails," that counts as 1 action.

**Automation hook:**
```
claude "Read OUTREACH_CRM.xlsx, show me: (1) anyone with Status=Antwort_Erhalten, (2) overdue Naechste_Aktion_Datum, (3) top 5 Tier 1 Ausstehend. Max 8 items total."
```

---

## Step 1: Segment — Pick Who to Contact

**Source:** `OUTREACH_CRM.xlsx` → Pipeline sheet

**Priority order:**
1. `Status = Antwort_Erhalten` — ALWAYS handle replies first (warm leads)
2. `Status = Interesse_Spaeter` where `Naechste_Aktion_Datum <= today` — warm re-engagement
3. Any status with overdue `Naechste_Aktion_Datum` — don't let leads go cold
4. `Status = Ausstehend` + `Tier 1` + `Aktion = Outreach` — new cold outreach
5. `Status = Ausstehend` + `Tier 1` + `Aktion = Partnerschaft` — partnership asks
6. `Status = Ausstehend` + `Tier 1` + `Aktion = Registrieren` — platform registrations
7. `Status = Ausstehend` + `Tier 2` — secondary targets

**Batch size:** 5 contacts per session (ADHD-friendly)

**Automation hook:**
```
claude "From OUTREACH_CRM.xlsx, give me the next 5 contacts to action using the priority order in OUTREACH-WORKFLOW.md Step 1. Show: Organisation, Email, Aktion, Kategorie, Info."
```

---

## Step 2: Personalize — Prepare the Message

**Per contact, select template by `Aktion` column:**

| Aktion | Template File | CTA |
|--------|--------------|-----|
| Outreach | `TEMPLATES/cold-outreach.md` | "15-min call?" |
| Partnerschaft | `TEMPLATES/partnership-pitch.md` | "collaborate?" |
| Registrieren | `TEMPLATES/platform-registration.md` | "how to register?" |

**Personalization:** Use the CRM's `Info` and `Warum_Relevant` columns to fill `{personalized_line}`. Each template file includes a personalization guide per contact type.

**Attachments (when ready):**
- Schools/Jugendhaus → Lead magnet PDF
- Partners → One-pager PDF
- Platforms → No attachment

**Automation hook:**
```
claude "For these 5 contacts from the CRM, generate personalized emails. Read the matching template from TEMPLATES/ based on their Aktion column. Use their Info and Warum_Relevant for the personalized line. Output ready-to-send text."
```

---

## Step 3: Send — Execute the Outreach

**Pre-send checklist:**
- [ ] Email sent FROM Elisabeth's account (not Rafael's)
- [ ] `{personalized_line}` references something SPECIFIC about them
- [ ] Correct template variant matches their `Aktion` value
- [ ] Website URL included and working
- [ ] Phone number included
- [ ] DSGVO footer present (in every template)

**After sending, immediately update CRM:**

| Column | Value |
|--------|-------|
| Status | `Gesendet` |
| Gesendet_Am | today (YYYY-MM-DD) |
| Naechste_Aktion | `Follow-Up 1` |
| Naechste_Aktion_Datum | +5 biz days (private sector) or +7 biz days (government/Nationalpark) |
| Kanal | `Email` |

**Batch target:** 5 emails per session, 2 sessions per week = 10/week

**Automation hook:**
```
claude "Update these 5 rows in OUTREACH_CRM.xlsx: set Status=Gesendet, Gesendet_Am=2026-02-XX, Kanal=Email, Naechste_Aktion=Follow-Up 1, Naechste_Aktion_Datum=2026-02-XX"
```

---

## Step 4: Track — Monitor Responses

### Status Values (strict — use only these, must match CRM Legende sheet)

| Status | Meaning | Next Action |
|--------|---------|-------------|
| `Ausstehend` | Not contacted yet | Segment + send |
| `Gesendet` | First email sent, no reply | Wait for Naechste_Aktion_Datum |
| `Follow_Up_1` | First follow-up sent | Wait for follow-up 2 date |
| `Follow_Up_2` | Second follow-up sent | Wait, then Kalt |
| `Antwort_Erhalten` | They replied (any reply) | Respond within 24h |
| `Interesse_Spaeter` | "Interessant, aber nicht jetzt" | Set Naechste_Aktion_Datum to their suggested time or next Saison |
| `Gespraech_Geplant` | Discovery call booked | Prepare + attend |
| `Angebot_Gesendet` | Proposal/offer sent | Follow up in 7 days |
| `Gebucht` | Workshop confirmed | Deliver + get feedback |
| `Abgeschlossen` | Workshop delivered | Testimonial + referral loop |
| `Kalt` | No response after all follow-ups | Revisit next Saison |
| `Nicht_Interessiert` | Explicit rejection | Archive permanently |
| `Partnerschaft_Aktiv` | Ongoing collaboration | Quarterly check-in |

### Edge Cases

**Out-of-office auto-reply:**
Write "OOO bis [Datum]" in `Antwort` column. Set `Naechste_Aktion_Datum` to the OOO return date + 2 days. Keep status as `Gesendet` (not `Antwort_Erhalten`).

**Wrong contact / redirected:**
Update `Kontaktperson` and `Email` in the same CRM row. Add note in `Notizen`: "Weitergeleitet von [original contact]". Do NOT create a duplicate row. Reset follow-up cadence (treat as new first contact).

**Email bounced:**
Try to find alternate email on their website. If found, update `Email` and resend. If not found, add note "Email bounced" and set `Status = Kalt`.

**"Interessant, aber nicht dieses Semester":**
Set `Status = Interesse_Spaeter`. Set `Naechste_Aktion_Datum` to the start of the relevant booking window (see Seasonal Timing below). This does NOT count against your response rate KPIs — it's a warm lead.

### Seasonal Timing (Austrian School Booking Windows)

| Saison | Booking Window | When to Re-Contact |
|--------|---------------|-------------------|
| Herbst (Sep-Nov) | Schools book in June-August | Re-contact June 1 |
| Fruehling (Mar-May) | Schools book in Jan-Feb | Re-contact January 7 |
| Sommer (Ferienpass) | Gemeinden plan in March-April | Re-contact March 1 |
| Ganzjahr | Bildungshaeuser book rolling | +90 days from last contact |

When setting `Naechste_Aktion_Datum` for `Kalt` or `Interesse_Spaeter` contacts, use the next relevant booking window — NOT a flat +90 days.

**Automation hook:**
```
claude "From OUTREACH_CRM.xlsx: count per status, list overdue Naechste_Aktion_Datum, calculate response rate (Antwort_Erhalten / total Gesendet+Follow_Up_1+Follow_Up_2), show next week's due items."
```

---

## Step 5: Follow-Up — The Cadence

### Timeline by Contact Type

**Private sector** (JUFA, Hotels, private Jugendunterkuenfte, Erlebnispaedagogik):
| Day | Action | Template |
|-----|--------|----------|
| 0 | Initial email | See Step 2 |
| +5 biz days | Follow-Up 1 | `TEMPLATES/follow-ups.md` → FU1 |
| +12 biz days | Follow-Up 2 (final) | `TEMPLATES/follow-ups.md` → FU2 |
| +15 biz days | LinkedIn connection request | No message |
| +20 biz days | Mark `Kalt` | Set Naechste_Aktion_Datum to next Saison |

**Government / institutional** (Nationalparks, Landes-Jugendhaus, Bildungsdirektion, Gemeinden):
| Day | Action | Template |
|-----|--------|----------|
| 0 | Initial email | See Step 2 |
| +7 biz days | Follow-Up 1 | `TEMPLATES/follow-ups.md` → FU1 |
| +18 biz days | Follow-Up 2 (final) | `TEMPLATES/follow-ups.md` → FU2 |
| +25 biz days | Mark `Kalt` | Set Naechste_Aktion_Datum to next Saison |

(Government contacts: skip LinkedIn step — Beamte rarely use it for this.)

### CRM Updates Per Follow-Up

| After | Status | Naechste_Aktion | Naechste_Aktion_Datum |
|-------|--------|-----------------|----------------------|
| Follow-Up 1 sent | `Follow_Up_1` | Follow-Up 2 | +7 biz days (private) or +11 biz days (govt) |
| Follow-Up 2 sent | `Follow_Up_2` | LinkedIn/Mark Kalt | +3 days (private) or +7 days (govt) |
| No response | `Kalt` | Revisit | Next booking window (see Seasonal Timing) |

---

## Step 6: Convert — From Reply to Booking

**When someone replies (Status → `Antwort_Erhalten`):**

1. **Within 24 hours:** Send personalized response:
   - Answer their specific question
   - Propose 2-3 call times (or Calendly link)
   - Attach one-pager PDF if not sent yet

2. **After discovery call (Status → `Gespraech_Geplant` → `Angebot_Gesendet`):**
   - Send written offer within 48h
   - Include: dates, format, pricing, what's included
   - Follow up on offer after 7 days

3. **After booking confirmed (Status → `Gebucht`):**
   - Send confirmation email with logistics
   - Add to calendar
   - Prepare workshop materials
   - Send reminder 1 week before

**CRM columns to update at each stage:**

| Column | When |
|--------|------|
| Antwort | Copy key content of their reply |
| Antwort_Datum | Date they replied |
| Kontaktperson | Name of the person who responded |
| Naechste_Aktion | "Call buchen" / "Angebot senden" / "Workshop vorbereiten" |
| Naechste_Aktion_Datum | Deadline for YOUR next action |

---

## Step 7: Referral Loop — Post-Workshop

**After each delivered workshop (Status → `Abgeschlossen`):**

Use templates from `TEMPLATES/post-workshop.md`:

1. **Day +1:** Thank-you email + testimonial request (2 questions)
2. **Day +7:** Referral ask
3. **Day +14:** If testimonial received → add to website + outreach materials
4. **Day +30:** "Next semester" reminder (seasonal rebooking)

**New leads from referrals:** Add to CRM with:
- `Quelle = Empfehlung von {referrer}`
- `Tier = Tier 1` (referrals convert 3-5x better)
- `Naechste_Aktion = Outreach` with high priority

---

## Automation Opportunities (ordered by impact)

### Now (Claude Code, no extra tools)
| What | How | Impact |
|------|-----|--------|
| Daily briefing | Read CRM, filter overdue items | Prevents leads going cold |
| Batch personalization | Read 5 contacts + matching template → output emails | 10x faster than manual |
| CRM status update | Bulk update after send sessions | Keeps data clean |
| Weekly report | Counts per status, conversion rates | Visibility into pipeline |

### Soon (needs Calendly or email tool)
| What | How | Impact |
|------|-----|--------|
| Calendly link in templates | Booking → CRM update | Removes scheduling friction |
| Email draft queue | Generate all week's emails in one batch | Batch work = ADHD friendly |

### Later (needs n8n)
| What | How | Impact |
|------|-----|--------|
| Auto follow-up reminders | n8n reads CRM dates → Telegram alert | Never forget a follow-up |
| Response detection | Email → n8n → flag CRM row | Instant visibility |
| Post-workshop auto-sequence | Status=Abgeschlossen → auto-send testimonial request | Referral loop on autopilot |

---

## File Structure

```
naturprogramme/
  outreach/
    OUTREACH-WORKFLOW.md          ← this file (process, NO inline templates)
    READY-TO-SEND.md              ← DEPRECATED (use CRM + TEMPLATES/ instead)
    TEMPLATES/
      cold-outreach.md            ← Aktion=Outreach
      partnership-pitch.md        ← Aktion=Partnerschaft
      platform-registration.md    ← Aktion=Registrieren
      follow-ups.md               ← Follow-Up 1 + Follow-Up 2
      post-workshop.md            ← Testimonial + Referral + Rebooking
  leads/
    OUTREACH_CRM.xlsx             ← SINGLE SOURCE OF TRUTH (Pipeline + Legende + Dashboard)
    ALL_CONTACTS.csv              ← backup / raw import
    CONTACTS_INDEX.txt            ← reference
    [7 segmented .xlsx files]     ← original research (archived, read-only)
```

---

## KPIs (check weekly)

| Metric | Target | Formula |
|--------|--------|---------|
| Emails sent / week | 10 | Count where Gesendet_Am = this week |
| Response rate | >15% | (Antwort_Erhalten + Interesse_Spaeter) / total sent |
| Call booking rate | >30% of replies | Gespraech_Geplant / Antwort_Erhalten |
| Close rate | >30% of calls | Gebucht / Gespraech_Geplant |
| Follow-up compliance | 100% | No overdue Naechste_Aktion_Datum |

---

## Quick Reference: "What Do I Do Now?"

| Situation | Action |
|-----------|--------|
| Just sat down, 15 min | Run daily briefing → handle 1 reply or send 2 emails |
| Have 1 hour | Batch: personalize 5 + send 5 + update CRM |
| Someone replied | Drop everything, respond within 24h |
| Got OOO auto-reply | Note return date in CRM, set Naechste_Aktion_Datum to return+2 |
| Got "wrong person, try X" | Update CRM row (new contact), resend |
| Got "interesting, not now" | Status=Interesse_Spaeter, set date to next booking window |
| Call went well | Send offer within 48h, update CRM |
| Workshop done | Day +1 thank-you, Day +7 referral ask |
| Nothing to do | Check Tier 2 contacts, or improve templates based on response data |
| 30 emails, 0 replies | STOP. Revise subject line + first line. Test new angle on 10 contacts. |
