# Standards & API Reference

> Project: Job Architecture & Skills Taxonomy · Generated: 2026-05-06

## Industry Standards & Specifications

### Occupational Taxonomy Standards

**O*NET (Occupational Information Network)**
- **Publisher:** US Department of Labor, Employment and Training Administration
- **Current version:** v29.0 (updated annually)
- **URL:** https://www.onetonline.org / https://services.onetcenter.org/
- **Relevance:** Authoritative US occupational taxonomy covering 1,000+ occupations with structured attributes for knowledge, skills, abilities, tasks, work activities, and work context. Foundational reference for any job architecture project; data is US government public domain with no IP restrictions. The O*NET-SOC classification codes serve as the primary occupational identifiers in Schema.org JobPosting structured data.

**ESCO (European Skills, Competences, Qualifications and Occupations)**
- **Publisher:** European Commission
- **Current version:** v1.2.0 (default v1.0.9)
- **URL:** https://esco.ec.europa.eu/en/about-esco/what-esco
- **Licence:** EUPL 1.2 (free software licence)
- **Relevance:** Multilingual occupational and skills classification across 27 official EU languages, with maintained crosswalk to O*NET. Provides a non-US-centric complement to O*NET for international job architecture projects and enables multilingual skills display by leveraging ESCO's translations. Its coverage of Latin America makes it increasingly relevant outside Europe.

**Lightcast Open Skills Taxonomy**
- **Publisher:** Lightcast (formerly Emsi Burning Glass)
- **Current version:** Monthly release cycle; 33,000+ skills as of 2026
- **URL:** https://lightcast.io/open-skills
- **Licence:** Creative Commons Attribution 4.0 (CC-BY 4.0)
- **Relevance:** The largest real-world-signal-derived skills dataset publicly available, generated from 500 million+ job postings. Three skill types (specialised, common, software) across 31 top-level categories. Freely embeddable in open-source and commercial products with attribution. Serves as the de facto shared skills language embedded in many HR tech products. Official crosswalk to O*NET and ESCO available.

**SFIA 9 (Skills Framework for the Information Age)**
- **Publisher:** SFIA Foundation (sfia-online.org)
- **Current version:** v9 (October 2024)
- **URL:** https://sfia-online.org/en/sfia-9/sfia-9
- **Relevance:** Global skills and competency framework specifically for digital and IT roles, covering 121 skills organised across seven levels of responsibility. Download formats include PDF, Excel, JSON, and RDF. Widely adopted by IT departments for technology job architecture; version 9 added AI and data skills. Freely accessible under a registered-user free licence; content can be embedded via API links or downloaded for tooling integration.

**WEF Global Skills Taxonomy (Future of Jobs)**
- **Publisher:** World Economic Forum
- **URL:** https://www.weforum.org/reports/the-future-of-jobs-report-2025
- **Relevance:** Framework categorising skills relevant to workforce transformation; used in global workforce planning discussions. The 2025 Future of Jobs Report provides a reference categorisation of emerging and declining skills across industries, useful as a forward-looking signal layer for a self-maintaining taxonomy agent.

---

### Learning Technology & Competency Standards

**IEEE 1484.20.1-2007 — Data Model for Reusable Competency Definitions**
- **Publisher:** IEEE Standards Association
- **URL:** https://standards.ieee.org/ieee/1484.20.1/4012/
- **Relevance:** Defines an XML data model for describing, referencing, and sharing competency definitions in a way that is independent of any particular context or system. Establishes how to formally represent the key characteristics of a competency (whether skill, knowledge, ability, or attitude). Inactivated in 2021 but remains widely cited as the foundational competency interoperability standard; superseded by ongoing IEEE P1484.20.2 work.

**IEEE P1484.20.2 — Defining Competencies (Active Working Group)**
- **Publisher:** IEEE Learning Technology Steering Committee (LTSC)
- **URL:** https://sagroups.ieee.org/1484-20-2/important-documents/
- **Relevance:** Active working group developing the successor to IEEE 1484.20.1, with participation from Lightcast, 1EdTech, and the Advanced Distributed Learning (ADL) initiative. Relevant for ensuring the data model of a new job architecture tool aligns with emerging interoperability standards.

**IEEE 9274.1.1-2023 — Experience API (xAPI) Specification**
- **Publisher:** IEEE (standardising the ADL/Rustici xAPI 2.0 spec)
- **URL:** https://xapi.com/overview/
- **Relevance:** Standard for tracking learning experiences using an "[actor] [verb] [object]" statement model. Enables skills gained through formal and informal learning to be recorded against a Learning Record Store (LRS) and surfaced in job architecture tools. Critical integration point for the gap-to-learning-pathway automation feature (v1.1).

**IMS/1EdTech Reusable Definition of Competency or Educational Objective (RDCEO)**
- **Publisher:** 1EdTech Consortium (formerly IMS Global)
- **URL:** https://www.imsglobal.org/competencies/rdceov1p0/imsrdceo_bestv1p0.html
- **Relevance:** Specification for representing competency definitions for interoperability between learning systems; preceded and influenced IEEE 1484.20.1. Widely adopted in LMS platforms; relevant for aligning job architecture skill definitions with LMS-stored competencies.

---

### HR Data Interchange Standards

**HR Open Standards (HR-XML / HR-JSON)**
- **Publisher:** HR Open Standards Consortium
- **URL:** https://www.hropenstandards.org/
- **Relevance:** Industry consortium defining data interchange schemas for exchanging HR data (including job profiles, competency frameworks, and skills) between systems. The HR Open Skills Data Workgroup is developing a new API data standard for exchanging skills data including proficiency levels, assessment scores, and learning recommendations — with participation from Lightcast, Skillsoft, 1EdTech, and IEEE LTSC. JSON Schema and XML schemas are available for free download at hropenstandards.org/standards-downloads.

**Schema.org JobPosting / Occupation**
- **Publisher:** W3C/Schema.org community (Google, Microsoft, Yahoo, Yandex)
- **URL:** https://schema.org/JobPosting / https://schema.org/Occupation
- **Relevance:** Structured data vocabulary used to mark up job postings for search engine indexing. The `occupationalCategory` property references O*NET-SOC codes; `skills` and `educationRequirements` properties define competency requirements. An AI job architecture tool that outputs Schema.org-compliant job descriptions improves SEO findability and integrates with job distribution platforms. JSON-LD is the recommended serialisation format.

---

### Human Capital Reporting Standards

**ISO 30414:2025 — Human Resource Management: Human Capital Reporting**
- **Publisher:** International Organisation for Standardisation
- **URL:** https://www.iso.org/standard/30414
- **Relevance:** The 2025 revision of the world's first international standard for human capital reporting, covering 69 metrics across 11 areas including skills, capabilities, and development. A job architecture platform that structures skill data consistently with ISO 30414 enables organisations to feed workforce capability data directly into their ESG and human capital disclosures — an important enterprise selling point. Skills coverage, gap severity, and reskilling investment metrics are explicitly within scope.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect Core 1.0**
- **Publisher:** IETF (OAuth 2.0) / OpenID Foundation (OIDC)
- **URL:** https://openid.net/specs/openid-connect-core-1_0.html
- **Relevance:** De facto standard for API authorisation and authentication. All major HRIS platforms (Workday, SAP SuccessFactors, BambooHR) require OAuth 2.0 for API access. An open-source job architecture tool must implement OAuth 2.0 client flows to authenticate against HRIS systems and expose its own API securely. OpenID Connect enables SSO integration with enterprise identity providers (Okta, Azure AD, Ping).

**OpenAPI Specification 3.1 (OAS 3.1)**
- **Publisher:** OpenAPI Initiative (Linux Foundation)
- **URL:** https://swagger.io/specification/
- **Relevance:** Standard format for describing RESTful APIs in a machine-readable YAML/JSON document. All modern HR tech APIs (Merge, Knit, Beamery, iMocha) publish OpenAPI specs. Designing the job architecture tool's own API against OAS 3.1 from the start enables auto-generation of SDKs, client libraries, and interactive documentation — critical for developer adoption.

---

## Similar Products — Developer Documentation & APIs

### O*NET Web Services API

- **Description:** Free REST API providing programmatic access to O*NET's full occupational database including occupations, skills, tasks, knowledge, and abilities attributes.
- **API Documentation:** https://services.onetcenter.org/reference/ (v2.0) / https://services.onetcenter.org/v1.9/reference/ (v1.9)
- **GitHub Samples:** https://github.com/onetcenter/web-services-v2-samples
- **Developer Guide:** https://services.onetcenter.org/ (registration required for API key)
- **Standards:** REST/JSON; read-only GET requests; v2.0 launched November 2025 with 12+ new features including technology skills search and registered apprenticeship reports
- **Authentication:** API key (obtained via free developer registration)
- **Licence:** US government public domain; no restrictions on use of returned data

### ESCO API

- **Description:** Web service providing linked-data access to the European Skills, Competences, Qualifications and Occupations classification across 27 languages.
- **API Documentation:** https://esco.ec.europa.eu/en/use-esco/use-esco-services-api/esco-web-service-api
- **Developer Guide:** https://esco.ec.europa.eu/en/use-esco/use-esco-services-api
- **Standards:** REST/JSON-LD; each concept identified by a URI; supports content-negotiation for different response formats
- **Authentication:** None required for public read access
- **Licence:** EUPL 1.2 (free software licence)

### Lightcast Skills API

- **Description:** REST API providing access to Lightcast's Open Skills taxonomy (33,000+ skills) and proprietary labor market data including skill demand, salary benchmarks, and job posting analytics.
- **API Documentation:** https://docs.lightcast.dev/apis/skills
- **Free Tier Documentation:** https://docs.lightcast.io/lightcast-api/docs/free-api-access
- **Access Page:** https://lightcast.io/open-skills/access
- **Standards:** REST/JSON; OAuth 2.0 client credentials; versioned monthly with changelog
- **Authentication:** OAuth 2.0 (client credentials grant)
- **Notes:** Free tier available for Skills API and Titles API (33,000+ skills, 75,000+ job titles); broader data access (job postings, salary data) requires enterprise contract

### Gloat Developer API

- **Description:** API for Gloat's AI-powered talent marketplace and dynamic skills ontology, enabling integration of internal mobility, skills matching, and workforce planning data.
- **API Documentation:** https://developer.gloat.com/
- **Standards:** REST/JSON; OpenAPI support; SDKs available
- **Authentication:** OAuth 2.0
- **Modules:** Skills Foundation, Job Architecture, Talent Marketplace, Candidacy API, Company API, Authorization API
- **Notes:** Requires enterprise contract; documentation is gated behind registration; integrates with Workday, SAP SuccessFactors, Greenhouse, LinkedIn Learning, Degreed, EdCast

### Beamery API (Skills Knowledge Graph)

- **Description:** REST API and AI Gateway for Beamery's talent lifecycle platform including Skills Knowledge Graph for managing skills inventory, job architecture, and candidate/employee profiling.
- **API Documentation:** https://api-docs-314159.beamery.engineer/
- **Support Docs:** https://support.beamery.com/hc/en-us/articles/4408168425105-Beamery-Developer-API-Documentation
- **Standards:** REST/JSON; OpenAPI; GraphQL
- **Authentication:** OAuth 2.0 or API key (basic auth for legacy endpoints)
- **Notable:** Provides a Workday Job Architecture Skills Push Integration for syncing Beamery-defined job architectures into Workday's Skills Cloud — a relevant integration pattern for any new tool in this space

### iMocha Skills Intelligence API

- **Description:** RESTful API for iMocha's skills intelligence cloud, providing access to skill taxonomy, job profile management, career path data, employee skill records, and assessment results.
- **API Documentation:** https://developer.imocha.io/ / https://developer.imocha.io/introduction-1260410m0
- **Standards:** REST/JSON; standard HTTP methods; versioned with last update June 2025
- **Authentication:** API key
- **Modules:** Authentication, Employee Data, JobProfiles, Taxonomy, CareerPath, Account
- **Integrations:** SAP SuccessFactors, Workday, BambooHR, major ATS/LMS platforms

### SAP SuccessFactors HCM OData API

- **Description:** OData v2-based API for SAP SuccessFactors providing CRUD access to HR data including competency frameworks, job classifications, and employee skill records across the HCM suite.
- **API Documentation:** https://help.sap.com/doc/a7c08a422cc14e1eaaffee83610a981d/2511/en-US/SF_HCM_OData_API_DEV.pdf
- **API Hub:** https://api.sap.com/ (SAP Business Accelerator Hub — full API catalog)
- **Standards:** OData v2 (REST-based protocol); JSON and XML response formats
- **Authentication:** OAuth 2.0 (SAML bearer assertion flow for enterprise SSO)
- **Notes:** Most widely deployed enterprise HRIS globally; a native connector to SuccessFactors is a key differentiator for any job architecture tool targeting enterprise buyers

### Eightfold AI Talent Intelligence API

- **Description:** API for Eightfold's AI talent intelligence platform, providing access to skills inference (from 1.6B+ career profiles), talent matching, and workforce intelligence capabilities.
- **API Documentation:** https://apidocs.eightfold.ai/
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0
- **Notes:** Eightfold's deep skill inference (including adjacent and inferred skills beyond stated resume skills) is a benchmark capability for AI-native job architecture tools; the API enables potential partnership integration for an OSS project that wants to offer AI skill inference without training its own model from scratch

### TechWolf Skill Engine API

- **Description:** REST API providing access to TechWolf's AI-powered skills intelligence layer — skill supply data (employee profiles), skill demand data (role requirements), and skill taxonomy governance tools.
- **API Documentation:** https://developers.techwolf.ai/reference/ / https://developers.techwolf.ai/reference/2025-01-29/API%20Documentation/Introduction/Introduction
- **Standards:** REST/JSON; OAuth 2.0; API versioned with date-stamped releases (e.g., 2025-01-29)
- **Authentication:** OAuth 2.0 (access tokens)
- **Integration Ecosystem:** Native connectors to SAP SuccessFactors, Workday, Gloat, Visier, ServiceNow; recognised by Everest Group 2025 PEAK Matrix for integration capability

### Merge Unified HRIS API

- **Description:** Unified API abstracting 220+ HRIS, ATS, CRM, and payroll integrations (including Workday, SAP SuccessFactors, BambooHR, ADP) behind a single normalised data model — reducing the integration burden for tools targeting multiple HRIS platforms.
- **API Documentation:** https://docs.merge.dev/hris/
- **Standards:** REST/JSON; OpenAPI 3.1; normalised data models across all connected platforms
- **Authentication:** OAuth 2.0 (for end-customer HRIS connections); API key for Merge platform access
- **Notes:** A job architecture OSS project could use Merge (or competitor Knit — https://developers.getknit.dev/) to add multi-HRIS connectivity without building individual connectors, reducing time-to-market significantly

### BambooHR REST API

- **Description:** REST API for BambooHR's HRIS platform, widely used by SMB and mid-market companies — the primary buyer segment underserved by current job architecture tools.
- **API Documentation:** https://documentation.bamboohr.com/docs/getting-started
- **Standards:** REST/JSON and XML; HTTP Basic Auth with API key
- **Authentication:** API key (HTTP Basic Auth)
- **Notes:** BambooHR is the dominant HRIS for the mid-market segment (the primary underserved buyer for an OSS job architecture tool); a BambooHR connector should be treated as MVP-priority HRIS integration alongside Workday

---

## Notes

**Emerging and Evolving Standards:**
- The HR Open Skills Data Workgroup is actively developing a system-to-system API standard specifically for skills data exchange (proficiency levels, assessment scores, learning recommendations). This standard is not yet published but reflects broad industry consensus on the need for skills interoperability. Monitoring hropenstandards.org for the release of this specification is advisable.
- IEEE P1484.20.2 (successor to the 2007 competency definition standard) is in active development. Aligning the internal data model of a new job architecture tool with the draft specification would position it well for interoperability when the standard is finalised.

**MCP Server Relevance:**
- The Anthropic Model Context Protocol (MCP) could be used to expose job architecture data (role profiles, skill taxonomies, gap analyses) to AI agents and LLMs as a context source. An MCP server wrapping the job architecture API would enable integration with Claude-powered HR advisory tools and automated workforce planning agents. No HR-specific MCP standard exists yet; this represents a first-mover opportunity for an OSS project in this domain.

**Unified HR API Layer:**
- Given the fragmentation of HRIS platforms (Workday, SAP SuccessFactors, BambooHR, ADP, Oracle HCM, Personio), building individual connectors is expensive. Integrating with a unified HR API layer (Merge, Knit, or Finch) from the outset is recommended to achieve broad HRIS coverage with minimal connector maintenance overhead.
