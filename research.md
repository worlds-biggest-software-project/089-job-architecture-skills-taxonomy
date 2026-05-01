# Job Architecture & Skills Taxonomy

> Candidate #89 · Researched: 2026-05-01

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|---|---|---|---|---|
| **Lightcast (fmr. Emsi Burning Glass)** | Largest integrated labor market data provider; Open Skills taxonomy of 34,000+ skills updated fortnightly from 500M+ job postings | Commercial / Open taxonomy | Open Skills API: free tier; full data access custom pricing ($50K–$300K+/year for enterprise) | Strength: most comprehensive real-world skills dataset, bi-weekly refresh, open-source taxonomy. Weakness: raw data; requires integration work to build job architecture |
| **Gloat** | AI-powered internal talent marketplace with a dynamic, market-responsive skills ontology | Commercial | Custom enterprise pricing; typically $150K–$500K/year | Strength: continuous ontology refresh from market signals, internal mobility engine. Weakness: high cost, primarily large-enterprise focus |
| **Beamery** | Talent lifecycle platform with skills-based talent graph; job architecture and career pathways | Commercial | Custom enterprise; typically $200K–$600K+/year | Strength: end-to-end talent lifecycle, strong skills inference from profile data. Weakness: expensive, complex implementation |
| **iMocha** | Skills intelligence platform; 30,000+ skills, 3,000+ ontologies; skills assessments + taxonomy | Commercial | Custom pricing; mid-to-large enterprise | Strength: combines skills assessment with taxonomy; strong industrial-organizational psychology foundation. Weakness: US/India centric |
| **Fuel50** | Career pathing and skills mapping platform; taxonomy built by I-O psychologists | Commercial | ~$5–$10 PEPM | Strength: psychometrically grounded taxonomy, good UX. Weakness: lighter on market-data signals vs. Lightcast |
| **Lexonis** | Skills framework management tool; AI-assisted framework development reducing build time from months to weeks | Commercial | Custom SME/enterprise pricing | Strength: fastest time-to-framework for custom job architectures. Weakness: less known, smaller client base |
| **365Talents** | AI skills intelligence platform; infers skills from employee profiles, aligns to job architecture | Commercial | Custom; European mid-market focus | Strength: multilingual, EU-native compliance. Weakness: smaller taxonomy than Lightcast/Gloat |
| **O*NET (US DOL)** | US government occupational database; 1,000+ occupations with detailed skills, abilities, and tasks | Open / Free | Free | Strength: authoritative, free, widely adopted. Weakness: updated slowly, US-centric, not designed for internal job architecture tooling |

## Relevant Industry Standards or Protocols

- **O*NET (Occupational Information Network)** — US DOL's primary occupational taxonomy; 1,000+ occupations with knowledge, skills, abilities, tasks, and work activity attributes; industry-standard reference for US job architecture.
- **ESCO (European Skills, Competences, Qualifications and Occupations)** — European Commission multilingual classification system; 27 official languages; increasingly adopted globally including Latin America; official crosswalk to O*NET maintained.
- **Lightcast Open Skills Taxonomy** — open-source library of 34,000+ real-world skills; Creative Commons licensed; widely embedded in HR tech products as a common skills language.
- **WEF Global Skills Taxonomy** — World Economic Forum framework for categorising skills relevant to the Future of Jobs; used in global workforce planning discussions.
- **HR Open Standards (HR-XML / JSON)** — data interchange schemas for exchanging job profiles and competency data between systems.
- **IEEE P2933 (Clinical IoT) / ISO 30414** — ISO standard on human capital reporting increasingly references skills measurement; skills taxonomy data feeds into these disclosures.
- **SFIA (Skills Framework for the Information Age)** — technology-specific skills framework widely used in IT job architecture; version 9 current as of 2024.

## Available Research Materials

1. Lightcast (2023). *The Lightcast Open Skills Taxonomy*. lightcast.io. https://lightcast.io/resources/blog/open-skills-taxonomy — Technical overview of methodology; vendor-produced.
2. Pexelle (2025). *How Frameworks like ESCO and O*NET Help Organize Skills for the Modern Workforce*. pexelle.com. https://pexelle.com/how-frameworks-like-esco-and-onet-help-organize-skills-for-the-modern-workforce/ — Independent practitioner analysis.
3. World Economic Forum (2025). *Future of Jobs Report 2025*. WEF. https://www.weforum.org/reports/the-future-of-jobs-report-2025 — Peer-quality global research; 63% of employers cite skill gaps as top barrier.
4. Deloitte Insights (2025). *From Jobs to Skills to Outcomes: Rethinking How Work Gets Done*. deloitte.com. https://www.deloitte.com/us/en/insights/topics/talent/future-of-workforce-planning/planning-work-outcomes.html — Thought leadership from major consulting firm.
5. iMocha (2025). *5 Steps to Building a Skills Architecture in 2026*. imocha.io. https://www.imocha.io/blog/skills-architecture — Practitioner guide; some vendor perspective.
6. Korn Ferry (2025). *Designing a Future-Ready Job Architecture Framework*. kornferry.com. https://www.kornferry.com/insights/featured-topics/future-of-work/designing-a-future-ready-job-architecture-framework — Consulting firm guidance.
7. Jobspikr (2025). *Open Skills and Talent Graphs: Guide to Skills-Based Hiring in 2025*. jobspikr.com. https://www.jobspikr.com/blog/open-skills-and-talent-graphs-2025/ — Technical overview of skills graph infrastructure.

## Market Research

**Market Size:**
- Global skills management / talent intelligence market: ~$3.5B in 2024, projected $7.5B by 2030 (CAGR ~13%).
- WEF Future of Jobs 2025: 63% of employers identify skill gaps as the #1 barrier to business transformation; 39% of existing skill sets expected to be outdated by 2030 — creating urgent demand.
- Skills-based hiring adoption: 45% of large enterprises report moving to skills-based job architecture vs. title-based in 2025 (iMocha survey).

**Pricing Table:**

| Tier | Example | Price |
|---|---|---|
| Open / Free | O*NET, ESCO, Lightcast Open Skills | $0 |
| Lightcast data access | Lightcast API | $50K–$300K+/year (custom) |
| SMB career pathing | Fuel50 | $5–$10 PEPM |
| Mid-market skills platform | 365Talents, Lexonis | Custom ~$50K–$150K/year |
| Enterprise talent intelligence | Gloat, Beamery | $150K–$600K+/year |

**Buyer Personas:**
- **Compensation & Total Rewards team** — needs a consistent job architecture (levels, grades, bands) to underpin pay equity and benchmarking.
- **HR Transformation Lead** — implementing skills-based organisation; wants a taxonomy that integrates with LMS, ATS, and succession tools.
- **L&D Director** — maps skills gaps to learning content; needs a live taxonomy that reflects what roles actually require.
- **CHRO / CPO** — driving skills-based workforce strategy; needs executive reporting on capability strengths and gaps.

**Notable M&A / Funding:**
- Lightcast (merger of Emsi + Burning Glass, 2021) — private equity backed; dominant market position in labor market data.
- Beamery raised $138M Series C (2022, valuation ~$800M).
- Gloat raised $90M Series D (2022).
- Eightfold AI (skills inference) raised $220M Series E (2021, valuation $2.1B).
- SAP acquired WalkMe (digital adoption) in 2024 for $1.5B partly for skills adoption analytics.

## AI-Native Opportunity

- **Continuous, self-maintaining skills taxonomy**: static taxonomies (even O*NET) go stale as new technologies emerge; AI can continuously parse job postings, LinkedIn profiles, course descriptions, and industry publications to identify emerging skills and deprecate obsolete ones — keeping the taxonomy current without manual curation effort.
- **Automatic role-to-skills inference**: organisations spend months building job architecture manually; AI can infer what skills are genuinely required for each role by analysing job postings, incumbent performance data, and project outputs — replacing the expensive consulting engagement with an automated first draft.
- **Skills gap visualisation at organisational scale**: current tools show individual skills gaps; AI can aggregate across the organisation to identify systemic capability risks (e.g., "the company has zero internal quantum computing expertise") and recommend build/buy/partner strategies.
- **Underserved segment — mid-market HR teams without consultants**: Korn Ferry and Mercer charge $200K–$1M+ for job architecture consulting engagements; SMBs and mid-market firms cannot afford this; an AI-native OSS tool that automates the job architecture process would democratise a capability currently reserved for large enterprises.
- **OSS differentiation**: the Lightcast Open Skills taxonomy is Creative Commons licensed and available as an API — an OSS job architecture tool that starts with this open taxonomy, adds AI-driven role profiling, a grade/level hierarchy builder, and integration with major HRIS systems would be genuinely novel and solve a real market need at zero licensing cost.
