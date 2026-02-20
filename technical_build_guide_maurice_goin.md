# Technical Build Guide: Maurice Goin — Both Projects

## Internal Reference for Anthony Pinto / Veteran Vectors

**Client:** Maurice Goin — GFS Building Maintenance / Neat & Sweet Cleaning  
**Date:** February 19, 2026

---

# PROJECT 1: Hiring Pipeline & Workforce Management CRM

---

## Architecture Overview

**Central hub:** Google Sheet (or Airtable) as the database  
**Automation engine:** n8n (self-hosted or cloud)  
**Integrations:** Indeed, DocuSign, background check provider, payroll system (switching from Paychecks — confirm new provider at audit call), email/SMS notifications

---

## Google Sheet Structure

### Tab 1: Candidates

| Column | Purpose |
|--------|---------|
| Candidate ID | Auto-generated (timestamp-based) |
| Name | Full name |
| Email | For DocuSign + comms |
| Phone | For SMS notifications |
| Source | Indeed / Referral / Other |
| Region | Which service area |
| Date Applied | Auto-populated |
| Screening Score | Auto-scored from questions |
| Status | Applied → Screened → Interview Scheduled → Background Check → Approved → Onboarding → Active → Terminated |
| Interviewer | Director of Ops or assigned |
| Interview Date | Scheduled date |
| Interview Notes | Free text |
| Background Check Status | Pending / Passed / Failed |
| Background Check Date | When initiated |
| DocuSign Status | Not Sent / Sent / Signed |
| Payroll Status | Not Added / Added |
| Start Date | First day on the job |
| Assigned Locations | Which sites they clean |
| Notes | General notes |

### Tab 2: Locations

| Column | Purpose |
|--------|---------|
| Location ID | Unique identifier |
| Location Name | Building/client name |
| Address | Physical address |
| Region | Service area grouping |
| Inspection Frequency | How often (weekly, biweekly, monthly) |
| Last Inspection Date | Auto-updated |
| Last Inspection Score | Pass/Fail + score |
| Assigned Workers | Who cleans this location |
| Contract Value | Monthly revenue |
| Client Contact | POC at the client |

### Tab 3: Inspections

| Column | Purpose |
|--------|---------|
| Inspection ID | Auto-generated |
| Location ID | Links to Locations tab |
| Inspector | Who did the inspection |
| Date | When |
| Overall Score | Numeric or Pass/Fail |
| Areas Failed | Bathrooms, dusting, floors, etc. |
| Work Orders Generated | Yes/No |
| Worker Assigned | Who gets the work order |
| Work Order Status | Pending / Completed |
| Notes | Inspector comments |

### Tab 4: Performance Tracker

| Column | Purpose |
|--------|---------|
| Worker ID | Links to Candidates tab |
| Worker Name | |
| Total Inspections | Count of inspections at their locations |
| Failed Inspections | Count of failures attributed to them |
| Failure Rate | Auto-calculated percentage |
| Status Flag | Green / Yellow / Red |
| Transition Plan | Yes/No — triggers hiring for their region |
| Last Review Date | |

---

## n8n Workflows

### Workflow 1: Candidate Intake from Indeed

**Trigger:** Schedule (every 15–30 min) or webhook if Indeed supports it  
**Purpose:** Pull new applicants into the tracker automatically

**Nodes:**
1. **Schedule Trigger** — runs every 30 min
2. **HTTP Request** — hit Indeed API or scrape Indeed inbox (email parsing as fallback)
   - Alternative: Use an email trigger that monitors a dedicated hiring inbox where Indeed sends notifications
3. **Extract Data** — parse candidate name, email, phone, screening responses
4. **Score Screening** — Code node: assign points based on screening question answers (availability, transportation, experience, willingness to clean)
5. **Check Duplicates** — Google Sheets lookup: does this email already exist?
6. **IF node** — New candidate? Continue. Duplicate? Skip.
7. **Google Sheets (Append)** — add new row to Candidates tab with Status = "Applied"
8. **Send Notification** — email or Slack to VA: "New applicant: [Name] — Score: [X/10]"

**Fallback approach if Indeed API isn't available:**
- Set up a dedicated email (hiring@gfsbuilding.com)
- Indeed sends application notifications there
- n8n Email Trigger monitors that inbox
- Parse the email body for candidate info

---

### Workflow 2: Screening → Interview Scheduling

**Trigger:** Google Sheets trigger (row updated) or webhook from VA's action  
**Purpose:** When VA marks candidate as "Screened," notify Director of Ops to schedule interview

**Nodes:**
1. **Google Sheets Trigger** — watches Candidates tab for Status change to "Screened"
2. **IF node** — Screening Score >= threshold (e.g., 7/10)? 
   - Yes → continue
   - No → auto-update Status to "Rejected," send rejection email
3. **Send Email/SMS to Director of Ops** — "New candidate ready for interview: [Name], Score: [X], Region: [Y]. Schedule interview?"
4. **Wait node** — optional, wait for Director to respond or update sheet
5. **Google Sheets (Update)** — set Status to "Interview Scheduled" when date is entered

---

### Workflow 3: Background Check Automation

**Trigger:** Google Sheets trigger — Status changes to "Background Check"  
**Purpose:** Auto-initiate background check, no more photo-of-form workflow

**Nodes:**
1. **Google Sheets Trigger** — Status = "Background Check"
2. **HTTP Request** — hit background check provider API (e.g., Checkr, GoodHire, Sterling)
   - Pass: candidate name, email, SSN (if collected), DOB
   - Note: Need to confirm which provider Maurice uses at audit call
3. **Google Sheets (Update)** — set Background Check Status = "Pending"
4. **Wait node** — poll for results (or use webhook if provider supports it)
5. **IF node** — Passed?
   - Yes → update Status to "Approved," trigger Workflow 4
   - No → update Status to "Failed Background," notify Director of Ops

**If no API available on background check provider:**
- Use DocuSign or a form to collect candidate consent
- VA still submits manually, but the RESULT triggers the next step
- n8n monitors for background check result (email parser or manual sheet update)

---

### Workflow 4: Automated Onboarding

**Trigger:** Google Sheets trigger — Status changes to "Approved"  
**Purpose:** Fire off DocuSign, welcome comms, payroll entry — zero manual steps

**Nodes:**
1. **Google Sheets Trigger** — Status = "Approved"
2. **DocuSign API** — create envelope from template, pre-fill with candidate name + email
   - Template: all standard onboarding paperwork
   - Send to candidate's email
3. **Google Sheets (Update)** — DocuSign Status = "Sent"
4. **DocuSign Webhook** — listen for "completed" event
5. **On completion:**
   - Google Sheets (Update) — DocuSign Status = "Signed"
   - HTTP Request — push new hire data to payroll system API
   - Google Sheets (Update) — Payroll Status = "Added"
   - Send Email — welcome email to new hire with first-day instructions, location assignment, point of contact
   - Send Notification — to Director of Ops: "[Name] is fully onboarded and ready to start on [Date]"
6. **Google Sheets (Update)** — Status = "Active"

---

### Workflow 5: Inspection Logging & Work Orders

**Trigger:** Form submission (Google Form, Typeform, or custom n8n form)  
**Purpose:** Inspector submits results → auto-generate work orders for failed items

**Nodes:**
1. **Webhook/Form Trigger** — inspector submits inspection via mobile form
   - Fields: Location, date, pass/fail per area (bathrooms, dusting, floors, trash, etc.), notes, photos (optional)
2. **Google Sheets (Append)** — add row to Inspections tab
3. **Google Sheets (Update)** — update Location tab: Last Inspection Date, Last Inspection Score
4. **IF node** — any areas failed?
   - Yes → continue to work order generation
   - No → done, send "all clear" notification
5. **Create Work Order** — for each failed area:
   - Look up assigned worker for that location
   - Send SMS/email to worker: "Work order for [Location]: [Failed Areas]. Complete by next shift on [Date]."
   - Add work order to Inspections tab with status "Pending"
6. **Update Performance Tracker** — increment inspection count + failure count for that worker
7. **IF node** — failure rate > threshold (e.g., 30%)?
   - Yes → flag worker as "Red," send alert to Director of Ops: "Worker [Name] has [X%] failure rate. Consider transition plan for [Region]."
   - This is the KEY piece: inspection data drives hiring decisions

---

### Workflow 6: Transition Plan → Targeted Hiring Alert

**Trigger:** Performance Tracker update — worker flagged as "Red"  
**Purpose:** When a worker is consistently underperforming, auto-recommend targeted hiring for their region

**Nodes:**
1. **Google Sheets Trigger** — Status Flag changes to "Red"
2. **Lookup** — which locations/regions does this worker cover?
3. **IF node** — is there already an active job posting for this region?
   - Yes → notify Director of Ops: "Active posting already exists for [Region]"
   - No → continue
4. **Send Alert** — to Maurice + Director of Ops: "Recommend posting for [Region] based on inspection performance. Worker [Name] has [X%] failure rate across [Y] inspections. Transition plan recommended."
5. **Optional:** Auto-draft Indeed posting for that region (pre-filled template)

---

## Audit Call Checklist (What to Confirm)

- [ ] Which payroll system are they switching to? API availability?
- [ ] Which background check provider? API or manual?
- [ ] DocuSign account details — template setup
- [ ] Indeed account type — email notifications or API access?
- [ ] How inspections currently happen — paper? App? Who submits?
- [ ] What inspection software are they evaluating? Can we integrate?
- [ ] How many regions/locations total?
- [ ] Who needs access to the dashboard? (Maurice, Director of Ops, VA)
- [ ] EOS metrics — do they want any KPIs surfaced in this system?
- [ ] Communication preferences — email, SMS, Slack, other?

---

## Tool Stack & Costs

| Tool | Purpose | Monthly Cost |
|------|---------|-------------|
| n8n Cloud (Starter) | Workflow engine | ~$20/mo |
| Google Sheets | Database (free) or Airtable ($20/mo for Pro) | $0–20/mo |
| DocuSign API | Onboarding paperwork | Depends on plan (may already have) |
| Background check API | Checkr/GoodHire | Per-check pricing (already paying) |
| Twilio (optional) | SMS notifications | ~$10/mo |
| **Total new costs** | | **~$30–50/mo** |

---
---

# PROJECT 2: Lead Generation — Trigger-Based Detection System

---

## Architecture Overview

**Central hub:** Google Sheet (or Airtable) as the lead pipeline  
**Automation engine:** n8n (same instance as Project 1)  
**Data sources:** SAM.gov API, Apify scrapers, BLDUP, job board monitoring, Google Alerts  
**Outreach:** Email (SalesHandy or SMTP), LinkedIn (manual or semi-automated)  
**AI layer:** OpenAI API for lead scoring and outreach personalization

---

## Google Sheet Structure

### Tab 1: Lead Pipeline

| Column | Purpose |
|--------|---------|
| Lead ID | Auto-generated |
| Company Name | |
| Contact Name | Decision-maker |
| Title | Facility Manager, Director of Ops, etc. |
| Email | |
| Phone | |
| LinkedIn URL | |
| Region/State | |
| Lead Source | RFP / New Construction / Job Change / Property Sale / Contract Expiration / Competitor Issue |
| Trigger Detail | Specific event that flagged this lead |
| Trigger Date | When the signal was detected |
| Estimated Value | Contract size estimate |
| Status | Detected → Qualified → Contacted → Replied → Meeting → Site Visit → Proposal → Won/Lost |
| Outreach Date | When first contacted |
| Follow-up Count | How many touches |
| Next Action | What's next + date |
| Notes | |

### Tab 2: RFP Tracker

| Column | Purpose |
|--------|---------|
| RFP ID | SAM.gov notice ID or platform ID |
| Source | SAM.gov / FindRFP / BidNet / State Portal |
| Title | Opportunity title |
| Agency/Org | Who posted it |
| NAICS | Should be 561720 |
| Location | State/city |
| Deadline | Submission deadline |
| Estimated Value | If listed |
| Set-Aside | Small business, veteran-owned, etc. |
| AI Score | How well it matches GFS (location, size, type) |
| Status | New / Reviewing / Bidding / Submitted / Won / Lost / Passed |
| Link | Direct URL to opportunity |
| Notes | |

### Tab 3: Signal Monitor Log

| Column | Purpose |
|--------|---------|
| Signal ID | Auto-generated |
| Signal Type | RFP / Construction Permit / Job Posting / Property Sale / Contract Expiration / Review |
| Source | Which platform detected it |
| Raw Data | Summary of what was detected |
| Date Detected | |
| Processed | Yes/No — has it been turned into a lead? |
| Lead ID | Links to Lead Pipeline if converted |

---

## n8n Workflows

### Workflow 1: SAM.gov RFP Monitor

**Trigger:** Schedule — runs daily at 6 AM  
**Purpose:** Find new janitorial RFPs in target states

**Nodes:**
1. **Schedule Trigger** — daily at 6:00 AM
2. **HTTP Request** — SAM.gov Opportunities API
   - Endpoint: `https://api.sam.gov/opportunities/v2/search`
   - Parameters:
     - `naics=561720` (Janitorial Services)
     - `postedFrom=[yesterday's date]`
     - `postedTo=[today's date]`
     - `state=NH,MA,ME,CT,NY,VT,NJ,PA,MD,VA` (East Coast coverage)
     - `api_key=[free SAM.gov API key]`
3. **Split In Batches** — process each opportunity
4. **Check Duplicates** — Google Sheets lookup by RFP ID
5. **IF node** — new opportunity?
6. **AI Agent (OpenAI)** — score the opportunity:
   - Prompt: "Score this RFP for a commercial janitorial company based in NH that targets $50K+ contracts. Consider: location match, contract size, service type, set-aside eligibility (veteran-owned). Score 1-10 and explain."
   - Input: title, description, location, NAICS, set-aside info
7. **IF node** — AI Score >= 6?
   - Yes → add to RFP Tracker + send notification
   - No → log but don't alert
8. **Google Sheets (Append)** — add to RFP Tracker tab
9. **Google Sheets (Append)** — add to Signal Monitor Log
10. **Send Notification** — email to Maurice + business partner:
    - Subject: "New RFP Match: [Title] — Score [X/10]"
    - Body: summary, deadline, link, AI reasoning

**Getting the SAM.gov API key:**
- Free: Register at https://sam.gov → go to https://open.gsa.gov/api/sam-opportunities/ → request API key
- Takes ~24 hours to receive

**Alternative if API is tricky:**
- Use Apify's SAM.gov Opportunities Monitor (~$5/mo)
- Configure it to post to an n8n webhook when new NAICS 561720 opportunities appear
- Much simpler setup, handles pagination and deduplication

---

### Workflow 2: State & Local RFP Scraping

**Trigger:** Schedule — runs daily  
**Purpose:** Catch bids from state/local platforms that SAM.gov doesn't cover

**Nodes:**
1. **Schedule Trigger** — daily
2. **HTTP Request (multiple)** — scrape or poll these sources:
   - **FindRFP.com** — if they have RSS/email alerts, use Email Trigger instead
   - **BidNet Direct** — same approach, monitor email alerts
   - **CleanWork (getcleanwork.com)** — scrape NH/MA pages for new bids
   - **State procurement portals** — NH: das.nh.gov, MA: commbuys.com
3. **For each source:** parse results, extract title, deadline, location, value
4. **Check Duplicates** — against RFP Tracker
5. **AI Score** — same scoring as Workflow 1
6. **Google Sheets (Append)** — to RFP Tracker + Signal Log
7. **Send Notification** — if score >= 6

**Practical note:** Some of these may need Apify or Browserless for scraping. Budget ~$10-15/mo for scraping credits.

---

### Workflow 3: New Construction Permit Monitor

**Trigger:** Schedule — runs weekly (permits don't change daily)  
**Purpose:** Spot new commercial buildings that will need cleaning contracts

**Nodes:**
1. **Schedule Trigger** — weekly, Monday mornings
2. **HTTP Request** — BLDUP API or scrape BLDUP.com for NH + MA
   - Filter: commercial projects, new construction, status = "permitted" or "under construction"
   - Alternative: Shovels.ai API — filter by property type = commercial, permit status = active
   - Alternative: Boston Open Data API (data.boston.gov) for Boston-area permits
3. **Filter** — only commercial buildings above a minimum square footage (e.g., 10,000+ sq ft)
4. **Check Duplicates** — have we already logged this project?
5. **Enrich Contact** — for each new project:
   - Look up the developer/owner name
   - HTTP Request to Apollo or Clay API → find email + phone for the owner or property manager
   - If no enrichment available, flag for manual research
6. **AI Score** — "Score this construction project as a cleaning lead: [details]. Consider building type, size, location proximity to NH, estimated completion date."
7. **Google Sheets (Append)** — to Lead Pipeline with Source = "New Construction"
8. **Google Sheets (Append)** — to Signal Monitor Log
9. **Send Notification** — "New construction lead: [Building Name] in [City] — [X sq ft] — Est. completion: [Date]"

**Contact enrichment options (pick one at audit call):**
- Apollo.io API — $49/mo for 10,000 credits
- Clay.com — $149/mo but very powerful enrichment
- Manual: just log the project and Maurice's team finds the contact

---

### Workflow 4: Facility Manager Job Change Monitor

**Trigger:** Schedule — runs daily  
**Purpose:** Detect when companies hire new facility managers (buying signal)

**Nodes:**
1. **Schedule Trigger** — daily
2. **HTTP Request** — scrape job boards for relevant postings:
   - **Google Jobs API** (via SerpAPI ~$50/mo) — search: "Facility Manager" OR "Director of Facilities" OR "Building Operations Manager" in NH, MA, ME, CT, NY
   - **LinkedIn Jobs** (via Apify LinkedIn Jobs Scraper) — same search
   - **Indeed** (via Apify Indeed Scraper) — same search
3. **Filter** — remove cleaning company job postings (we want the CLIENTS hiring FMs, not competitors)
4. **For each posting:**
   - Extract: company name, job title, location, company size (if available)
   - Check: is this company already in our Lead Pipeline?
5. **IF node** — new company?
6. **Enrich** — look up company details:
   - What industry? (filter for: offices, medical, retail, industrial, property management)
   - Company size? (100+ employees preferred)
   - Do they have multiple locations? (higher contract value)
7. **AI Score** — "This company just posted for a Facility Manager. Score as a cleaning service lead: [company details]"
8. **Google Sheets (Append)** — to Lead Pipeline with Source = "Job Change"
9. **Set Timer** — DON'T reach out immediately. Wait 30-60 days (new FM needs to settle in)
10. **Wait node** → then trigger outreach workflow

**Why this works:** New facility managers review all vendor contracts within their first 90 days. Reaching out at day 30-60 puts Maurice in front of them exactly when they're evaluating.

---

### Workflow 5: Contract Expiration Tracker (Government)

**Trigger:** Schedule — runs weekly  
**Purpose:** Track when competitor government contracts expire so Maurice can bid the re-solicitation

**Nodes:**
1. **Schedule Trigger** — weekly
2. **HTTP Request** — FPDS (Federal Procurement Data System) or USASpending.gov API
   - Search: NAICS 561720, states = target states
   - Pull: contractor name, contract start date, contract end date, contracting agency, value
3. **Calculate** — Code node: for each contract, calculate months until expiration
4. **IF node** — expiring within 6 months?
   - Yes → add to tracking list
5. **Google Sheets (Append)** — to RFP Tracker with Status = "Upcoming Expiration"
6. **Set Reminders:**
   - 6 months out: "Heads up — [Agency] contract with [Competitor] expires [Date]"
   - 3 months out: "Watch SAM.gov for re-solicitation of [Agency] janitorial contract"
   - 1 month out: "Urgent — should be posted by now. Check SAM.gov."
7. **Cross-reference** — when a new SAM.gov posting appears (Workflow 1), check if it matches a tracked expiration

**Data source note:** USASpending.gov has a free API with contract award data. FPDS is the underlying source. Both are free and comprehensive for federal contracts.

---

### Workflow 6: Competitor Review Monitor

**Trigger:** Schedule — runs weekly  
**Purpose:** Spot when competitor cleaning companies get bad reviews

**Nodes:**
1. **Schedule Trigger** — weekly
2. **Build competitor list** — maintain a Google Sheet tab with local competitor names
3. **For each competitor:**
   - **HTTP Request** — Google Places API (or SerpAPI) — fetch recent reviews
   - Filter for reviews from the past week
   - Filter for 1-2 star reviews
4. **IF node** — new negative reviews found?
5. **AI Analysis** — "Analyze this review of a cleaning company. Does it suggest the reviewer's company might be looking for a new cleaning provider? Extract: reviewer company name if mentioned, specific complaints, severity."
6. **IF node** — actionable lead?
7. **Google Sheets (Append)** — to Lead Pipeline with Source = "Competitor Issue"
8. **Send Notification** — "[Competitor] got a bad review from what appears to be [Company/Location]. Potential lead for GFS."

**Low effort, low volume, but high conversion when it hits.**

---

### Workflow 7: Outreach Sequence (For Qualified Leads)

**Trigger:** Google Sheets trigger — Status changes to "Qualified" in Lead Pipeline  
**Purpose:** Personalized multi-channel outreach based on the trigger type

**Nodes:**
1. **Google Sheets Trigger** — Status = "Qualified"
2. **AI Agent (OpenAI)** — draft personalized email based on trigger type:
   - **RFP lead:** "We noticed [Agency] posted a janitorial services RFP. GFS Building Maintenance has X years of experience in this region..."
   - **New Construction:** "Congratulations on the [Building Name] project. As you approach completion, we'd love to discuss your cleaning and maintenance needs..."
   - **Job Change:** "Welcome to [Company]. As you settle into your new role, we'd love to introduce GFS Building Maintenance and discuss how we can support your facilities..."
   - **Property Sale:** "We noticed [Property] recently changed ownership. We specialize in commercial cleaning for properties like yours..."
   - **Contract Expiration:** "As your current cleaning contract approaches renewal, we'd like the opportunity to present our services..."
   - **Competitor Issue:** "We understand maintaining consistent cleaning quality can be challenging. GFS Building Maintenance prides itself on..."
3. **Human Review Step** — send draft to Maurice via email/Slack for approval before sending
   - This is important: Maurice said relationships matter. Don't auto-send cold outreach for $50K+ contracts.
4. **On Approval:**
   - Send email via SMTP or SalesHandy
   - Update Lead Pipeline: Outreach Date, Status = "Contacted"
5. **Follow-up Sequence** — Wait 5 days → send follow-up email (different angle)
   - Wait 7 more days → send final follow-up
   - After 3 touches with no response → Status = "No Response"
   - If reply detected → Status = "Replied," send notification to Maurice

**Key principle:** For RFPs, there's no outreach needed — it's a bid response. The outreach sequence is for the non-RFP triggers (construction, job changes, property sales, etc.)

---

### Workflow 8: Weekly Summary Report

**Trigger:** Schedule — every Friday at 4 PM  
**Purpose:** Give Maurice a snapshot of the lead pipeline

**Nodes:**
1. **Schedule Trigger** — Friday 4 PM
2. **Google Sheets (Read)** — pull all data from Lead Pipeline + RFP Tracker
3. **Code node** — calculate:
   - New leads detected this week (by source)
   - RFPs with upcoming deadlines
   - Leads contacted this week
   - Responses received
   - Meetings/site visits scheduled
   - Pipeline value (sum of estimated contract values by stage)
4. **AI Agent** — format into a clean summary email
5. **Send Email** — to Maurice + business partner

---

## Shared Infrastructure (Both Projects)

| Component | Purpose | Cost |
|-----------|---------|------|
| n8n Cloud (Starter) | Runs all workflows for both projects | ~$20/mo |
| Google Sheets | Database for both systems | Free |
| OpenAI API | Lead scoring, email drafting, inspection analysis | ~$10-15/mo |
| Apify (Starter) | SAM.gov monitor + web scraping | ~$30/mo |
| BLDUP (Free tier or Pro) | Construction permit data | $0-50/mo |
| Twilio | SMS notifications (hiring workflows) | ~$10/mo |
| SalesHandy or SMTP | Outreach emails (lead gen) | ~$25/mo |
| **Total shared stack** | | **~$75-130/mo** |

If Maurice does both projects, the shared tools (n8n, OpenAI, Apify) serve both systems. That's the bundle savings angle.

---

## Build Order & Timeline

### Week 1: Hiring Pipeline (Phase 1-2)
- Set up Google Sheets structure (both projects)
- Build Workflow 1: Candidate Intake
- Build Workflow 2: Screening → Interview
- Build Workflow 3: Background Check
- Build Workflow 4: Onboarding Automation
- Test end-to-end: fake candidate through full pipeline

### Week 2: Hiring Pipeline (Phase 3) + Lead Gen (Phase 1)
- Build Workflow 5: Inspection Logging & Work Orders
- Build Workflow 6: Transition Plan alerts
- Set up SAM.gov API key
- Build Lead Gen Workflow 1: SAM.gov Monitor
- Build Lead Gen Workflow 2: State/Local RFP Scraping
- Configure Apify actors

### Week 3: Lead Gen (Phase 2-3)
- Build Lead Gen Workflow 3: Construction Permits
- Build Lead Gen Workflow 4: Job Change Monitor
- Build Lead Gen Workflow 5: Contract Expiration Tracker
- Build Lead Gen Workflow 6: Competitor Reviews
- Build Lead Gen Workflow 7: Outreach Sequence
- Build Lead Gen Workflow 8: Weekly Report

### Week 4: Testing, Training, Handoff
- End-to-end testing of all workflows
- Error handling and edge cases
- Training sessions with Maurice, Director of Ops, VA
- Documentation: how to use dashboards, what to do when alerts come in
- Handoff + 60-day support period begins

---

## Risk Factors & Mitigations

| Risk | Mitigation |
|------|-----------|
| Indeed doesn't have a clean API | Fall back to email parsing (Indeed sends notifications) |
| Background check provider has no API | Manual entry triggers the rest of the automation |
| New payroll system unknown | Build payroll push as a separate module, plug in once confirmed |
| SAM.gov API rate limits | Use Apify actor as buffer, caches and deduplicates |
| BLDUP doesn't cover all target regions | Supplement with Shovels.ai or manual permit database checks |
| Maurice wants human review on outreach | Built into Workflow 7 — drafts go to him for approval first |
| Low volume on some signal types | Expected — competitor reviews and property sales are low volume but high quality. RFPs and job changes will be the primary volume drivers. |

---

## Questions for Audit Call

### Hiring Pipeline
1. What payroll system are you switching to? When?
2. Which background check provider do you use? Online portal or paper?
3. How does the Director of Ops currently submit inspection results? Phone? Paper? App?
4. What inspection software are you evaluating? Can we skip it and build inspection submission into this system?
5. How many locations/regions total?
6. How many workers total (approx)?
7. What does your current DocuSign template look like? How many documents?
8. Does the VA have a Google account we can share the dashboard with?

### Lead Generation
1. What states/regions do you want to target for new business?
2. Are you registered on SAM.gov? Do you have a D-U-N-S number?
3. Are you certified as veteran-owned business (SDVOSB)?
4. What's the minimum contract size worth pursuing? ($25K? $50K?)
5. What property types are you best at? (Office, medical, industrial, retail?)
6. Who reviews and approves outreach before it goes out?
7. Do you have a business email domain set up for cold outreach? (separate from main domain)
8. Any competitors you specifically want to monitor?
