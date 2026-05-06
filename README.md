# Job Architecture & Skills Taxonomy

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source toolkit for defining roles, levels, and skills frameworks company-wide — built on open taxonomies so any organisation can adopt skills-based workforce planning without enterprise consulting fees.

Job Architecture & Skills Taxonomy gives HR transformation, total rewards, and L&D teams a structured way to model job families, grade/level hierarchies, and skill requirements, then continuously compare them against employee profiles. It is aimed at mid-market and SMB organisations that today have no affordable path to skills-based job architecture, and grounds its data in the Lightcast Open Skills taxonomy (CC-BY) and O*NET (US public domain) rather than proprietary ontologies.

---

## Why Job Architecture & Skills Taxonomy?

- **Consulting-grade work is locked behind enterprise budgets.** Korn Ferry and Mercer charge $200K–$1M+ per engagement to build a job architecture; most firms below the Fortune 1000 simply go without.
- **Enterprise software is no cheaper.** Gloat ($150K–$500K+/year), Beamery ($200K–$600K+/year), and Lightcast data access ($50K–$300K+/year) all target large enterprises and bundle proprietary ontologies that cannot be exported.
- **Open data exists, but no open tooling.** O*NET is public domain and the Lightcast Open Skills taxonomy is CC-BY, yet no open-source builder ties them into a usable job architecture workflow with grade/level hierarchies and gap analysis.
- **Static taxonomies decay quickly.** WEF Future of Jobs 2025 estimates 39% of existing skill sets will be outdated by 2030; manually-curated frameworks go stale within 18 months and emerging roles take 2–3 years to land in O*NET.
- **The demand is acute.** 63% of employers in the WEF 2025 survey cite skill gaps as the top barrier to business transformation, and 45% of large enterprises report moving to skills-based job architecture in 2025.

---

## Key Features

### Open Taxonomy Foundation

- Import and search the Lightcast Open Skills taxonomy (32,000–34,000 skills, CC-BY 4.0) as the default skills library.
- Bundle O*NET occupational data (1,000+ occupations, US public domain) for role and task descriptors.
- Optional ESCO multilingual data (CC-BY 4.0) for non-English skill display across 27 languages.
- Crosswalk mappings between Lightcast, O*NET, and ESCO for interoperability with existing HR tooling.

### Job Architecture Builder

- Job profile builder for defining roles with skill requirements and proficiency levels.
- Grade/level hierarchy builder supporting job families, levels, and pay banding — the structural core of any job architecture project.
- Career pathway visualisation showing progression routes through the architecture.
- Export to standard formats (CSV, JSON, HR-XML) for downstream HRIS import.

### Skills Gap Analysis

- Compare employee profiles (uploaded or HRIS-synced) to role requirements with visual gap heat maps.
- Aggregate organisational skills heat maps showing capability coverage and gaps across teams and business units.
- Optional Lightcast API integration to overlay external market demand and salary benchmarks against internally-defined roles.

### Integration & Interchange

- CSV import/export at minimum, with API connectors for HRIS systems such as Workday and BambooHR.
- HR-XML / JSON interchange schemas for exchanging job profiles and competency data with other systems.
- API-first design so the taxonomy and architecture data can be consumed by ATS, LMS, and talent marketplace tools.

### AI-Assisted Curation (v1.1+)

- AI-assisted role profiling: given a job title and description, auto-suggest skills and proficiency levels drawn from the open taxonomy.
- Self-maintaining taxonomy agent that monitors external sources to propose emerging skills for addition and obsolete skills for deprecation.
- Organisational capability risk surfacing — aggregating skills gaps to flag systemic risks (e.g. a complete absence of a critical capability) and recommend build/buy/partner responses.

---

## AI-Native Advantage

Most incumbent taxonomies are either static (O*NET) or proprietary and locked to a single platform (Gloat, Beamery). An AI-native approach lets the taxonomy continuously parse job postings, profiles, and course catalogues to keep skills current without manual curation, and lets organisations generate a credible first-draft job architecture from job postings and incumbent data instead of a multi-month consulting engagement. AI also enables natural-language role definition and organisation-wide capability risk detection — capabilities today's tools either gate behind enterprise pricing or do not offer at all.

---

## Tech Stack & Deployment

The project is intended as self-hostable open-source software with optional cloud deployment. It builds on the Lightcast Open Skills taxonomy (CC-BY 4.0), O*NET (US public domain), and ESCO (CC-BY 4.0), with crosswalks between them. Integration is API-first, with HRIS connectors (Workday, BambooHR), HR-XML / JSON interchange, and optional Lightcast API access for live labour-market signals. SFIA and the WEF Global Skills Taxonomy are candidates for plug-in extension to support IT-specific and global workforce-planning use cases.

---

## Market Context

The global skills management and talent intelligence market was approximately $3.5B in 2024 and is projected to reach $7.5B by 2030 (CAGR ~13%). Incumbent pricing ranges from $5–$10 PEPM at the SMB career-pathing tier (Fuel50) up to $150K–$600K+/year for enterprise platforms (Gloat, Beamery), with Lightcast data access at $50K–$300K+/year on top. Primary buyers are Compensation & Total Rewards teams, HR Transformation Leads, L&D Directors, and CHROs/CPOs driving skills-based workforce strategy.

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
