# Non-Profit Fundraising CRM

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source donor management and fundraising CRM for nonprofits — covering gift processing, campaign tracking, and planned giving.

The Non-Profit Fundraising CRM is a constituent and gift management platform built for development teams at nonprofit organisations. It unifies donor records, gift and pledge processing, campaign tracking, and planned giving in a single system, with AI-native scoring and automation built in rather than bolted on.

---

## Why Non-Profit Fundraising CRM?

- Incumbent platforms like Blackbaud Raiser's Edge NXT and StratusLIVE deliver enterprise depth but at a cost and implementation complexity that excludes small-to-mid-size nonprofits.
- Affordable mid-market tools such as DonorPerfect carry dated UX and minimal AI/ML capability — no native predictive scoring or automated ask-amount recommendations.
- Salesforce NPSP / Agentforce Nonprofit is extensible but requires Salesforce licensing, certified consultants, and a steep learning curve unsuited to non-technical fundraising staff.
- Planned giving (bequests, charitable remainder trusts, gift annuities) is poorly served across all-in-one platforms; specialist tools do not integrate deeply with CRM workflows.
- Grant management is almost always a separate tool, creating data silos and duplicate entry between donor CRM and grant tracking.

---

## Key Features

### Constituent & Gift Management

- Unified donor record with giving history, contact details, relationships, and full interaction timeline
- Gift and pledge processing with batch entry, soft credit, split gift, tribute, matching gift, and honour/memorial tracking
- Recurring/sustainer giving management with automated payment scheduling
- Automated tax receipt and acknowledgment letter generation tied to gift processing

### Campaigns & Online Fundraising

- Online donation forms with card, ACH, and digital wallet support
- Segmentation and filtering for targeted appeals
- Campaign tracking with retention rate, LYBUNT/SYBUNT, and average gift dashboards
- Email marketing with templates, segmentation, and automated stewardship journeys

### AI-Powered Fundraising Intelligence

- Donor retention risk scoring with automated lapsed-donor outreach triggers
- Personalised ask amount recommendations using ML on giving history and engagement signals
- Planned giving prospect identification from donor tenure, age, and engagement signals
- Wealth screening integration (iWave, DonorSearch) embedded in the constituent record

### Planned Giving & Grants

- Bequest tracking, intent form capture, and estate gift staging
- Grant relationship management with funder records, deadline tracking, and report reminders
- Soft credit architecture for complex attribution (matching gifts, gift-in-kind, influencer credit)

### Integration & Compliance

- REST API with OAuth 2.0 for third-party integration and data export
- PCI DSS compliant payment processing
- GDPR/CCPA consent and data management tools
- NCOA address hygiene support

---

## AI-Native Advantage

Where incumbents have bolted predictive scoring onto legacy data models, an AI-native CRM closes the loop from score to recommended ask to personalised communication without manual handoffs. Churn models trained on RFM and engagement signals trigger stewardship sequences before donors lapse; LLMs draft personalised acknowledgments and stewardship updates at scale; and natural-language query interfaces let gift officers ask complex questions of their data without building filters by hand. Generative AI also transforms structured campaign metrics into executive-ready board reports, eliminating a recurring manual reporting burden.

---

## Tech Stack & Deployment

The project targets a self-hostable, API-first architecture exposing a REST API with OAuth 2.0 for third-party integration. Reference integrations target the standard nonprofit ecosystem (Stripe, PayPal, QuickBooks, Mailchimp, iWave/DonorSearch, Zapier). Webhooks support real-time data push to external systems. The data model follows widely-implemented conventions (household accounts, gift objects, pledge schedules, engagement scoring) that carry no known patent encumbrances.

---

## Market Context

The global nonprofit CRM platform market was valued at approximately $4.76 billion in 2025 and is projected to reach $10.20 billion by 2035 at a 7.9% CAGR (Spherical Insights, 2026). Fundraising and donation management accounts for 44.74% of module revenue, and SaaS subscription commands 81.64% of market share. Typical nonprofit CRM costs run $100–$500/month for small-to-mid-size organisations, with most paying $1,200–$3,600/year for mid-tier subscriptions excluding implementation and training. Primary buyers are development directors, major gift officers, planned giving officers, and annual fund managers at mid-to-large nonprofits, plus smaller nonprofits seeking affordable all-in-one tools.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
