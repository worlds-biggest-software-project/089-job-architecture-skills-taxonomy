# Job Architecture & Skills Taxonomy — Feature & Functionality Survey

> Candidate #89 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Lightcast (Open Skills) | Labor market data + open taxonomy | Open taxonomy (CC); data access commercial | https://lightcast.io/open-skills |
| Gloat | Internal talent marketplace + skills ontology | Commercial SaaS | https://gloat.com |
| Beamery | Talent lifecycle platform + skills graph | Commercial SaaS | https://beamery.com |
| iMocha | Skills intelligence + assessment platform | Commercial SaaS | https://imocha.io |
| Fuel50 | Career pathing + skills mapping | Commercial SaaS | https://fuel50.com |
| Lexonis | AI-assisted skills framework management | Commercial SaaS | https://lexonis.com |
| 365Talents | AI skills inference + job architecture | Commercial SaaS | https://365talents.com |
| O*NET (US DOL) | Occupational database | Open / Free (US government) | https://www.onetonline.org |

## Feature Analysis by Solution

### Lightcast Open Skills Taxonomy

**Core features**
- Library of 32,000–34,000 real-world skills generated from 500 million+ job postings, updated fortnightly
- Skill classification into three types: specialised skills, common skills, and software skills
- 31 top-level skill categories with hierarchical sub-categories
- Bi-weekly refresh pulling data from 40,000+ sources daily
- Open-source taxonomy available via API under Creative Commons licence

**Differentiating features**
- Largest real-world-signal-derived skills dataset publicly available
- Creative Commons licence enables downstream product embedding at no licensing cost
- Crosswalk mappings to O*NET and ESCO for interoperability
- Used as the embedded skills language in dozens of commercial HR tech products

**UX patterns**
- Primarily a data/API product; no end-user interface — consumed programmatically by downstream applications
- World Bank and enterprise analytics teams use the taxonomy layer directly via API

**Integration points**
- REST API for skills classification and job-posting parsing
- Compatible with HRIS, ATS, LMS, and talent marketplace systems
- Official crosswalk to O*NET and ESCO taxonomies

**Known gaps**
- Raw taxonomy only; no built-in job architecture builder, grade/level hierarchy, or gap analysis UX
- Full data access requires significant enterprise contract spend ($50K–$300K+/year)
- Skills at the leading edge of emerging technology may lag the fortnightly update cycle

**Licence / IP notes**
- Open Skills taxonomy: Creative Commons Attribution 4.0 — free to embed and redistribute with attribution
- Underlying job-posting database: proprietary; API access under commercial licence

---

### Gloat

**Core features**
- Dynamic Skills Ontology that updates automatically from internal employee data and external labour market signals
- Internal talent marketplace matching employees to projects, gigs, mentorships, and open roles by inferred skills
- Skill gap insights and organisational capability heat maps
- Skills inference from employee profiles, CVs, and work history without manual tagging

**Differentiating features**
- Continuous ontology refresh driven by market data — taxonomy stays current without manual curation
- Internal mobility engine powered by skills matching rather than title/seniority matching
- Workforce planning analytics that connect skills supply to strategic business objectives

**UX patterns**
- Employee-facing career hub showing inferred skills profile, recommended opportunities, and learning suggestions
- Manager-facing skills coverage dashboards by team and business unit
- HR admin console for taxonomy management and gap analysis configuration

**Integration points**
- HRIS connectors (Workday, SAP SuccessFactors, Oracle HCM)
- LMS integration for learning recommendations against skills gaps
- ATS integration for internal mobility vs. external hiring decision support

**Known gaps**
- High cost makes it inaccessible to mid-market and SMB organisations ($150K–$500K+/year)
- Primarily large-enterprise focused; implementation complexity is significant
- Skills ontology is proprietary and locked to the platform

**Licence / IP notes**
- Fully proprietary; no open-source components exposed
- Skills ontology data is Gloat's commercial asset; no external export rights

---

### Beamery

**Core features**
- Talent graph unifying candidate, employee, and alumni data with skills inference
- Job architecture and career pathway builder with configurable grade/level hierarchies
- Skills inference engine that automatically enriches profiles from CV text, job title, and project data
- Talent pipeline analytics connecting skills supply to workforce demand

**Differentiating features**
- End-to-end talent lifecycle coverage — from candidate attraction through internal mobility to alumni
- Strongest skills inference from profile data among commercial platforms (no manual tagging required)
- Talent graph approach enables cross-functional skills visibility at organisational scale

**UX patterns**
- Recruiter and HR dashboards with skills-filtered talent pools
- Employee career hub showing recommended internal roles and skills development pathways
- Executive reporting on capability strengths and strategic gaps

**Integration points**
- Deep CRM-style integration with ATS systems (Workday, Greenhouse, Lever)
- HRIS connectors for employee data sync
- API-first architecture supports custom integrations

**Known gaps**
- Very expensive for most organisations ($200K–$600K+/year)
- Complex implementation; typically requires dedicated consultants
- Less suited to smaller organisations without dedicated HR transformation teams

**Licence / IP notes**
- Fully proprietary; all skills inference models and talent graph data are Beamery commercial IP
- Raised $138M Series C (2022, ~$800M valuation)

---

### iMocha

**Core features**
- Skills library of 30,000+ skills across 3,000+ ontologies spanning multiple industries
- Integrated skills assessments for validating self-reported or inferred skills
- Skills taxonomy management with gap analysis against role requirements
- Validated by industrial-organisational psychology methodology

**Differentiating features**
- Only platform that combines skills taxonomy with validated psychometric skills assessments in a single product
- Ontology depth across technical, functional, and behavioural skill domains
- Skills scoring backed by assessment data — not just inferred from text

**UX patterns**
- HR admin-facing framework builder for defining role skill requirements
- Candidate/employee-facing assessment portal for skills validation
- Dashboard reporting on organisational skills coverage and gap severity

**Integration points**
- ATS and HRIS integrations for triggering assessments at hire and development stages
- LMS integrations for gap-to-learning pathway automation
- API access for embedding assessments in custom workflows

**Known gaps**
- US and India centric; international taxonomy coverage is thinner for non-English markets
- Assessment-heavy approach may create friction in cultures with low assessment acceptance
- Less strong on real-time labour market signal integration compared to Lightcast

**Licence / IP notes**
- Fully proprietary; assessment content and taxonomy are commercial IP
- Custom enterprise pricing; no published list price

---

### O*NET (US Department of Labor)

**Core features**
- Occupational database covering 1,000+ occupations with detailed descriptors for knowledge, skills, abilities, tasks, work activities, and work context
- Free public access to all data via O*NET Web Services API
- Crosswalk to ESCO (European Skills framework) maintained by O*NET
- Regularly updated by DOL occupational analysts (though less frequently than market-signal-driven taxonomies)

**Differentiating features**
- Authoritative US government source; widely adopted as a reference standard for job architecture work
- Zero cost for all data and API access
- Strong cross-referencing capability for occupational comparisons

**UX patterns**
- Public-facing search interface at onetonline.org
- Primarily consumed via API by downstream applications

**Integration points**
- O*NET Web Services REST API (free)
- Official crosswalk to ESCO for international mapping
- Embedded in many commercial HR products as the foundational occupation reference

**Known gaps**
- Updated slowly relative to market pace; emerging roles (AI Engineer, Prompt Engineer) take 2–3 years to appear
- US-centric; non-US occupational conventions not covered
- Not designed for internal job architecture tooling — requires custom application layer to build grade/level hierarchies and skills frameworks on top

**Licence / IP notes**
- US government public domain; unrestricted use, modification, and redistribution permitted
- No patent or IP restrictions on the data or taxonomy

## Cross-Cutting Feature Themes

### Table-Stakes Features
- A structured skills taxonomy (either built, licensed, or sourced from open datasets) as the foundation
- Role-to-skills mapping enabling job architecture definition by level and function
- Skills gap analysis comparing employee profiles to role requirements
- Integration with at least one HRIS system for employee data sync
- Reporting dashboards showing coverage, gaps, and capability heat maps by team or business unit

### Differentiating Features
- Real-time labour market signal integration (job posting analysis) to keep the taxonomy current
- Skills inference from unstructured data (CVs, job titles, project history) without manual tagging
- Internal mobility matching using skills rather than job titles
- Organisational-level capability risk identification ("the company has no quantum computing expertise")
- Grade/level hierarchy builder with pay-band linkage for compensation benchmarking

### Underserved Areas / Opportunities
- Mid-market and SMB organisations ($10M–$500M revenue) have no affordable job architecture tooling; Korn Ferry and Mercer charge $200K–$1M+ for consulting engagements; software solutions all target large enterprises
- Open-source job architecture tools are essentially absent; O*NET provides data but no tooling; Lightcast's open taxonomy has no accompanying builder
- Multilingual, non-US-centric skills frameworks are poorly served by the dominant US/Anglo taxonomies
- Skills taxonomy maintenance automation — most organisations build a taxonomy once and watch it go stale within 18 months

### AI-Augmentation Candidates
- Automatic role profiling: inferring skill requirements from job postings, incumbent performance data, and project outputs to generate job architecture first drafts without manual curation
- Self-maintaining taxonomy: continuous parsing of job postings, LinkedIn profiles, and course catalogues to identify emerging skills and deprecate obsolete ones
- Organisational capability risk alerting: aggregating skills gaps across the organisation to surface systemic risks and recommend build/buy/partner responses
- Natural-language job architecture builder: a practitioner describes a role in plain English; AI generates the skill profile, grade level, and career pathway suggestions

## Legal & IP Summary

- Lightcast Open Skills taxonomy is Creative Commons Attribution 4.0 — freely embeddable in OSS products without restriction; the underlying job-posting database is proprietary and must be licensed separately
- O*NET data is US government public domain — no restrictions whatsoever; safe foundation for any OSS project
- ESCO is published by the European Commission under CC-BY 4.0 — freely reusable
- Commercial platforms (Gloat, Beamery, iMocha) hold proprietary rights over their skills ontologies, taxonomy structures, and inference models; these cannot be reverse-engineered or incorporated into OSS products
- No known blocking patents on skills taxonomy methodology; patent risk is low for an OSS project building on open taxonomies
- ABA/bar-equivalent professional conduct rules do not apply to HR taxonomy tools

## Recommended Feature Scope

**Must-have (MVP)**:
- Open skills taxonomy foundation sourced from Lightcast Open Skills (CC-BY) and O*NET (public domain), importable and searchable
- Job profile builder: define roles with skill requirements, proficiency levels, and seniority/grade tiers
- Skills gap analysis: compare employee profiles (uploaded or HRIS-synced) against role requirements with visual gap heat maps
- Grade/level hierarchy builder: configure job families, levels, and banding — the core of any job architecture project
- Export to standard formats (CSV, JSON, HR-XML) for downstream HRIS import
- Basic HRIS integration (at minimum CSV import/export; ideally one API connector for Workday or BambooHR)

**Should-have (v1.1)**:
- AI-assisted role profiling: given a job title and description, auto-suggest skills and proficiency levels drawn from the open taxonomy
- Labour market signal integration: optional Lightcast API connection to show market demand and salary benchmarks alongside internally-defined roles
- Career pathway visualisation: show employees possible progression paths through the job architecture
- Multi-language support for skills taxonomy display (leveraging ESCO multilingual data)
- Organisational skills heat map: aggregate view of capability coverage and gaps across teams

**Nice-to-have (backlog)**:
- Self-maintaining taxonomy agent: AI monitors external sources to propose emerging skills for addition and obsolete skills for deprecation
- Succession planning integration: identify internal candidates with the closest skills match to critical senior roles
- Pay equity analysis: overlay skills-based job architecture with compensation data to surface pay equity risks
- LMS integration for gap-to-learning pathway automation (recommend specific courses for identified gaps)
- Plug-in architecture for connecting additional open taxonomies (SFIA for IT roles, WEF Future of Jobs taxonomy)
