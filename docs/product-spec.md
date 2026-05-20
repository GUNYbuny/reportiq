# ReportIQ — Technical Product Specification

## Overview
ReportIQ automates client reporting for digital marketing agencies. Agencies connect client ad/analytics accounts via OAuth. ReportIQ pulls data, runs AI narrative generation, and delivers white-labeled PDF reports by email automatically.

**Revenue target:** $10,000/month recurring
**Path:** 34 customers × $299/mo (Growth plan) = $10,166/mo
**Current phase:** Pre-build validation

---

## Technical Architecture

### Frontend
**Stack:** Next.js 14 (App Router) + Tailwind CSS
**Hosting:** Vercel (free tier: 100GB bandwidth, unlimited deploys)
**Why:** Zero config deployment, built-in API routes, server components reduce client JS

### Backend
**Stack:** Next.js API routes + Supabase (Postgres + Auth + Storage)
**Hosting:** Vercel (compute) + Supabase (free tier: 500MB DB, 1GB storage, 50MB file uploads)
**Job Queue:** Inngest (free tier: 50k runs/month) — handles async report generation
**Overflow hosting if needed:** Railway ($5/mo) or Render (free tier)

### Data Pipeline
```
Agency connects client account (OAuth)
  → Credentials stored encrypted in Supabase (connections table)
  → Scheduled job fires (monthly/weekly via Inngest cron)
  → Data fetcher pulls from platform APIs (GA4, Google Ads, Meta, GSC)
  → Raw data normalized + stored (reports table, raw_data JSONB column)
  → AI prompt assembled with client context + data
  → Claude API call → narrative text returned
  → PDF generator assembles white-labeled PDF (agency branding)
  → PDF stored in Supabase Storage
  → Email sent via Resend API to client + agency notification
  → Delivery logged (deliveries table)
```

### Authentication
- Agency user auth: Supabase Auth (email/password + magic link)
- Client platform OAuth: Standard OAuth 2.0 flows
  - Google (GA4 + Google Ads + GSC): Google OAuth — single consent screen covers all three
  - Meta Ads: Meta OAuth — requires Meta App Review for ads_read permission (1-2 weeks)
- Tokens stored encrypted in Supabase (connections table, encrypted_tokens column)
- Token refresh handled automatically on each data fetch

### Database Schema
```sql
-- Core tables
users (id, email, created_at, stripe_customer_id, plan)
agencies (id, user_id, name, logo_url, brand_color, created_at)
clients (id, agency_id, name, email, industry, notes, created_at)
connections (id, client_id, platform, encrypted_tokens, scopes, connected_at, last_synced)
reports (id, client_id, period_start, period_end, status, raw_data JSONB, ai_narrative, created_at)
deliveries (id, report_id, sent_to, sent_at, opened_at, pdf_url)
templates (id, agency_id, name, tone, focus_kpis, custom_instructions)
```

### AI Prompt Architecture
```
System: You are a senior digital marketing analyst writing a client report for [agency_name].
        Tone: [professional/conversational/detailed — set per client template]
        Client: [client_name], [industry], [business_goal]
        
User:   Period: [date_range]
        Previous period for comparison: [prior_period_data]
        
        GA4 Data: [sessions, users, conversions, top pages, traffic sources]
        Google Ads Data: [spend, impressions, clicks, CTR, CPC, conversions, ROAS]
        Meta Ads Data: [spend, reach, impressions, CTR, CPC, conversions, ROAS]
        GSC Data: [top queries, impressions, clicks, avg position]
        
        Write a complete monthly report with:
        1. Executive Summary (3-4 sentences, key wins + concerns)
        2. Channel-by-channel performance (what changed, why, what it means)
        3. Top 3 wins this period
        4. Top 3 areas for improvement
        5. Recommended actions for next period
        
        Write in first person plural as the agency ("We saw...", "We recommend...").
        Do not mention AI. Sound like a human analyst.
```

### PDF Generation
**Library:** Puppeteer (headless Chrome) via Vercel serverless function
**Approach:** Render HTML template with agency branding + AI narrative → capture as PDF
**Template:** Handlebars HTML template with agency logo, colors, client name injected
**Storage:** Supabase Storage (PDF stored, signed URL generated for email delivery)
**Alternative:** WeasyPrint if Puppeteer proves too heavy for serverless (Python-based)

### Email Delivery
**Service:** Resend (free tier: 3,000 emails/month — sufficient for MVP)
**White-labeling:** Emails sent from agency's domain via Resend custom domain (DNS setup, 15 min)
**Templates:** React Email templates — agency logo in header, professional design

---

## MVP Feature Scope

### MUST HAVE
- Agency signup + auth (Supabase Auth)
- Add clients + set basic info (name, industry, goal)
- Connect GA4 + Google Ads via OAuth (covers ~80% of agency clients)
- Manual "Generate Report" trigger (not scheduled yet — reduce complexity)
- AI narrative generation (Claude API)
- Basic PDF output with agency name/logo
- Email delivery to one client address
- Waitlist → paid conversion (Stripe, 1 plan only: $299/mo)

**Why these only:** Validates the core value prop (AI writes the report) with minimum build. Meta, GSC, scheduling, and templates are all additive.

### NICE TO HAVE (v2)
- Scheduled automatic delivery
- Meta Ads integration
- Google Search Console integration
- Review-before-send workflow
- Client-specific templates
- Slack notifications
- Multiple plan tiers
- White-label custom domain email

### EXPLICITLY OUT OF SCOPE for MVP
- Client-facing portal/login
- Team accounts (multi-seat)
- API access
- LinkedIn Ads, TikTok Ads, Shopify, HubSpot
- Report analytics (open rates, etc.)
- Custom report sections/drag-and-drop
- Mobile app

---

## 6-Week MVP Build Roadmap

### Week 1 — Auth + Data Model
- Set up Next.js + Supabase + Vercel project
- Implement agency signup/login (Supabase Auth)
- Build database schema (all tables)
- Create agency dashboard shell (add clients, view clients)
- **Test:** Can create account, add client, see it in DB

### Week 2 — Google OAuth + Data Fetching
- Implement Google OAuth flow (GA4 + Google Ads in one consent)
- Store encrypted tokens in connections table
- Build GA4 data fetcher (sessions, users, conversions, top pages)
- Build Google Ads data fetcher (spend, clicks, CTR, conversions, ROAS)
- **Test:** Connect a real test GA4 account, fetch real data, log to console
- **Critical path:** This is the riskiest step. Google OAuth approval + data shape validation can surface issues.

### Week 3 — AI Narrative Generation
- Build prompt assembly function (takes raw data → structured prompt)
- Integrate Claude API (Anthropic)
- Test narrative output quality with real data (iterate prompt until output is client-ready)
- Store narrative in reports table
- **Test:** Generate 3 sample reports manually. Would you send these to a real client?

### Week 4 — PDF Generation + Email Delivery
- Build HTML report template (agency-branded, responsive)
- Implement Puppeteer PDF generation (agency logo + colors injected)
- Store PDF in Supabase Storage
- Integrate Resend for email delivery
- Build "Generate + Send Report" button in dashboard
- **Test:** Full end-to-end: connect accounts → generate → receive PDF in email

### Week 5 — Billing + Polish
- Integrate Stripe (single $299/mo plan)
- Add paywall: must subscribe to generate reports
- Polish dashboard UI
- Add basic error handling + loading states
- **Test:** Full purchase flow. Someone who doesn't know you built it can sign up and use it.

### Week 6 — First Paying Customer
- Soft launch to waitlist
- Onboard first 3 agencies manually (white-glove — be on calls, fix everything)
- Fix all bugs found in live use
- **Goal:** 1 paying customer by end of week 6

---

## Unit Economics

### Cost Per Customer Per Month
| Item | Cost @ 10 customers | Cost @ 50 customers | Cost @ 100 customers |
|------|--------------------|--------------------|---------------------|
| Claude API (10 reports/customer, ~4k tokens each) | ~$0.80/customer | ~$0.70/customer | ~$0.60/customer |
| Supabase (scales with storage/compute) | ~$0 (free tier) | ~$2/customer | ~$0.50/customer |
| Vercel hosting | ~$0 (free tier) | ~$0.40/customer | ~$0.20/customer |
| Resend email | ~$0 (free tier) | ~$0.10/customer | ~$0.05/customer |
| PDF generation (compute) | ~$0.20/customer | ~$0.20/customer | ~$0.15/customer |
| **Total COGS** | **~$1/customer/mo** | **~$3.40/customer/mo** | **~$1.50/customer/mo** |
| **Revenue (Growth plan)** | **$299** | **$299** | **$299** |
| **Gross margin** | **~99.7%** | **~98.9%** | **~99.5%** |

**Break-even (including $50/mo fixed costs — Stripe fees, domain, misc):** 1 customer covers fixed costs.
**Profitable from customer 1.**

### Revenue at Scale
- 10 customers → $2,990/mo
- 34 customers → $10,166/mo (target)
- 50 customers → $14,950/mo
- 100 customers → $29,900/mo

---

## Risk Analysis

### Technical Risks
1. **Google OAuth approval delay** — Google sometimes flags new OAuth apps for review. Mitigation: register OAuth app immediately, use real agency email, complete all required fields, don't use "test" in app name.
2. **Meta Ads API requires App Review** — ads_read permission needs Meta to approve. Takes 1-2 weeks. Mitigation: Start Meta v1 with just GA4 + Google Ads. Meta is v2.
3. **Puppeteer too heavy for Vercel serverless** — 50MB function limit. Mitigation: Use @sparticuz/chromium (optimized for serverless). Fallback: dedicated Railway container for PDF generation.
4. **AI output quality inconsistency** — Reports may be generic or hallucinate specifics. Mitigation: Strong system prompt, few-shot examples in prompt, manual QA of first 20 reports, let agency review before send.
5. **Token refresh failures** — Stale OAuth tokens break scheduled reports. Mitigation: Proactive token refresh before each run, error alerts to agency if connection breaks.

### Business Risks
1. **Not enough people have the problem badly enough** — Mitigation: 5 demo calls before writing code. If nobody will commit to joining a paid beta, don't build.
2. **Competition from established players** — AgencyAnalytics ($12-15/client/mo), Databox, NinjaCat. Mitigation: They don't write the narrative. That's the wedge. Own "AI-written reports" before they copy it.
3. **Agencies won't pay until they see output quality** — Mitigation: Offer first report free. Let the product sell itself. One sample report > 10 cold emails.

### Biggest Assumption to Validate First
**"Agencies will pay $299/mo for AI-written reports, not just AI-assisted dashboards."**
Test this before building: In demo calls, show a sample AI-written report (generated manually for demo purposes). Ask: "Would you pay $299/month for this, delivered automatically?" If < 3 of 5 say yes, either the price is wrong or the output quality isn't there yet.

---

## Validation Checklist (Before Writing Code)

- [ ] 5 demo calls completed with real agency owners
- [ ] At least 3 of 5 say "yes" to paying $299/mo for automated AI-written reports
- [ ] At least 1 person says "take my money now" (strongest signal)
- [ ] Sample report generated manually for demo — reviewed by someone who receives agency reports
- [ ] Sample report rated "I'd send this to a client" by at least 3 agency owners
- [ ] Competitor research confirms no one is doing narrative AI reports at this price
- [ ] Google OAuth test app approved (can take a week — start immediately)
- [ ] At least 20 people on waitlist (demand signal, not just curiosity)

**If all boxes checked: build. If not: identify which assumption failed and pivot or reposition.**
