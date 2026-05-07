# Standards & API Reference

> Project: Interview Intelligence Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 42001:2023 — AI Management Systems**
- URL: https://www.iso.org/standard/42001
- The world's first AI management system standard. Provides an integrated approach to managing AI projects, covering risk assessment, governance, and responsible AI practices. In 2026, ISO/IEC 42001 and ISO/IEC 27001 are converging as baseline requirements for organisations implementing AI governance. Any interview intelligence platform using AI for scoring, transcription, or bias detection should align with this standard.

**ISO 10667-1:2020 — Assessment Service Delivery (Client Requirements)**
- URL: https://www.iso.org/standard/74716.html
- Establishes requirements and guidance for clients commissioning assessment services in employment and organisational settings. Covers recruitment, selection, development, appraisal, promotion, and outplacement. Directly relevant to structured interview delivery and candidate evaluation processes.

**ISO 10667-2:2020 — Assessment Service Delivery (Service Provider Requirements)**
- URL: https://www.iso.org/standard/74717.html
- Establishes requirements and guidance for service providers delivering assessments for work-related purposes. Specifies obligations for validity, reliability, fairness, and transparency in assessment design and delivery. A platform generating structured interview kits and scoring competencies is acting as an assessment service provider under this standard.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The global benchmark for information security management. Enterprise buyers of HR technology universally require ISO 27001 certification or SOC 2 equivalence as a procurement prerequisite. Covers asset management, access control, cryptography, incident response, and continuous audit.

**ISO/IEC 27701:2025 — Privacy Information Management Systems (PIMS)**
- URL: https://www.iso.org/standard/27701
- As of October 2025, this is now a stand-alone standard (previously an extension to ISO 27001/27002). Provides a privacy management system framework aligning with GDPR, CCPA, and other privacy regulations. Directly relevant for platforms handling candidate interview recordings, biometric data, and personal employment information. The 2025 edition adds specific guidance for AI-related processing, biometrics, and cloud services.

---

### W3C & IETF Standards

**W3C WebVTT — Web Video Text Tracks Format**
- URL: https://www.w3.org/TR/webvtt1/
- The W3C standard for timed text tracks in web video and audio. Defines the `.vtt` format used for captions, subtitles, and transcripts. WebVTT is the interchange format for interview transcripts: recordings stored in MP4 container format pair with WebVTT transcripts. Relevant for storing, displaying, and exchanging interview transcription output.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization framework used by all major ATS platforms (Greenhouse, Lever, Workday), video conferencing APIs (Zoom, Teams, Google Meet), and identity providers (Okta, Azure AD). Required for authenticating third-party integrations and enabling secure scoped access to candidate data.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWTs are the token format underpinning OAuth 2.0 and OpenID Connect. Used for secure transmission of session and identity claims between the platform and integrations. Enterprise SSO deployments rely on JWTs in conjunction with SAML or OIDC.

**RFC 7617 — HTTP Authentication: Basic and Bearer Token**
- URL: https://datatracker.ietf.org/doc/html/rfc7617
- Some ATS platforms (e.g. Greenhouse Harvest API) use API key-based authentication with Bearer tokens rather than OAuth flows. Both authentication models must be supported for the breadth of ATS integrations required.

**IETF RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Standard machine-readable format for API error responses. Recommended for consistent error handling across all outbound API integrations and the platform's own REST API.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on OAuth 2.0. Required for enterprise SSO integrations with identity providers such as Okta, Azure Active Directory, and Google Workspace. Enables verified user identity within the hiring workflow.

**SCIM 2.0 — System for Cross-domain Identity Management (RFC 7642, 7643, 7644)**
- URL: https://datatracker.ietf.org/doc/html/rfc7644
- Standard protocol for automating user provisioning and deprovisioning across enterprise SaaS applications. Required for enterprise HR deployments where interview intelligence platform seats are managed via Okta, Azure AD, or similar identity providers.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for describing RESTful APIs in a machine-readable format (JSON or YAML). OpenAPI 3.1 is a superset of JSON Schema Draft 2020-12, enabling schema reuse and automated SDK generation. Any public API surface of the platform should be documented as an OpenAPI spec to facilitate ATS partner integrations.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/draft/2020-12/json-schema-core
- Describes data structures for JSON payloads. Used to define canonical schemas for candidate profiles, interview transcripts, scorecards, and competency frameworks exchanged between the platform and ATS integrations.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/
- Still widely used in enterprise identity management for SSO, particularly in large organisations using legacy identity providers. Many enterprise ATS customers will require SAML 2.0 support alongside OIDC for single sign-on to the interview intelligence platform.

---

### Security & Authentication Standards

**OWASP Top 10 (2021 edition, current)**
- URL: https://owasp.org/www-project-top-ten/
- The authoritative reference for common web application security risks. An interview intelligence platform processing sensitive hiring data must mitigate all OWASP Top 10 risks, particularly broken access control, cryptographic failures, injection vulnerabilities, and insecure design patterns.

**NIST AI Risk Management Framework (AI RMF 1.0) — NIST AI 100-1**
- URL: https://www.nist.gov/itl/ai-risk-management-framework
- Voluntary framework published January 2023 providing structured guidance for identifying, assessing, and managing AI risks across the full lifecycle (Govern, Map, Measure, Manage). Includes specific provisions for fairness and bias management (MEASURE 2.11), directly applicable to AI scoring, competency tagging, and bias detection features. Used as the technical reference framework by enterprise procurement teams evaluating AI hiring tools.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- The predominant compliance attestation demanded by enterprise SaaS buyers in North America. SOC 2 Type II certifies that security controls (and optionally: availability, confidentiality, processing integrity, and privacy) have been operating effectively over an audit period of 6–12 months. All production commercial interview intelligence platforms (BrightHire, Metaview, Humanly) hold SOC 2 Type II certification. Essential for enterprise deals.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/docs/
- The open protocol (introduced by Anthropic) defining how AI systems communicate with external tools and data sources. While not currently used by incumbent interview intelligence platforms, MCP is architecturally relevant for AI-native implementations that expose interview data, candidate profiles, scorecards, and competency frameworks as context sources for AI agents. An AI-native build may expose MCP server endpoints for: retrieving interview transcripts, querying candidate scorecards, accessing competency framework definitions, and triggering interview kit generation workflows.

---

## Regulatory Frameworks

**EU AI Act — Title III (High-Risk AI Systems in Employment)**
- URL: https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai
- Key requirements effective 2 August 2026. AI systems used in recruitment and selection are classified as high-risk, triggering mandatory obligations: risk management documentation, data governance (training/validation datasets must be representative and free of errors), technical documentation for regulatory inspection, transparency requirements (candidates must be informed when AI influences hiring decisions), human oversight by design, accuracy and robustness testing, bias audits, and continuous monitoring. Penalties from August 2027: up to €30–35 million or 6–7% of global annual turnover for violations.

**NYC Local Law 144 (Automated Employment Decision Tools)**
- URL: https://www.nyc.gov/site/dca/about/automated-employment-decision-tools.page
- Enforced since July 2023. Requires employers using automated employment decision tools (AEDTs) in NYC to: (1) conduct independent annual bias audits covering impact ratios by race, ethnicity, and sex; (2) publish audit summaries on their careers website; (3) notify NYC-resident candidates at least 10 business days before AEDT use. Any platform used to score or rank candidates touching NYC-based roles must support these disclosure workflows. Non-compliance penalties start at $500 per violation escalating to $1,500/day for ongoing violations.

**Colorado AI Act (SB 24-205)**
- Effective 1 February 2026. Requires rigorous impact assessments for high-risk AI systems in employment, including bias audits and human oversight provisions. Applicable to any platform screening or ranking Colorado-based candidates.

**GDPR (General Data Protection Regulation) — Articles 6, 9, 22**
- URL: https://gdpr-info.eu/
- Recording interviews and processing candidate voice data constitutes processing of personal data under Article 6. Article 9 covers special categories: biometric or health data inferred from voice or video analysis requires explicit consent or a specific legal basis. Article 22 governs automated individual decision-making, requiring human oversight for consequential hiring decisions. Platforms operating in the EU must provide lawful basis documentation, privacy notices, data retention policies, right-of-access fulfilment, and cross-border transfer mechanisms (Standard Contractual Clauses or adequacy decisions).

**CCPA/CPRA (California Consumer Privacy Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- Since January 2023, California workers are fully included in CCPA protections. Voice recordings used in interviews qualify as "sensitive personal information" under CPRA. Employers using biometric data (including voiceprints derived from recordings) must provide specific notices, limit use to disclosed purposes, and honour employee requests to restrict use. Platforms must support data access and deletion request workflows for California candidates.

**Illinois BIPA (Biometric Information Privacy Act)**
- URL: https://www.ilga.gov/legislation/ilcs/ilcs3.asp?ActID=3004
- Any collection of biometric data (including voiceprints derived from interview recordings) from Illinois residents requires informed written consent, a retention schedule, and a published biometric data destruction policy. BIPA carries a private right of action with statutory damages of $1,000–$5,000 per violation — the highest enforcement risk of any US biometric privacy law. Platforms must implement opt-in consent flows for Illinois candidates.

**EEOC Uniform Guidelines on Employee Selection Procedures (29 CFR Part 1607)**
- URL: https://www.eeoc.gov/laws/regulations/qanda-clarify-uniform-guidelines
- The long-standing regulatory framework for employment selection procedures in the US. Requires that selection tools producing adverse impact on protected groups must demonstrate job-relatedness and business necessity. Applies to AI-driven scoring, screening, and ranking. Employers remain liable under Title VII for disparate impact regardless of whether the tool is vendor-supplied.

---

## Similar Products — Developer Documentation & APIs

### Greenhouse (Harvest API)

- **Description:** Greenhouse is the most widely integrated ATS in the interview intelligence space, with native integrations across all major competitors. The Harvest API provides full programmatic access to jobs, candidates, applications, interviews, scorecards, and offers.
- **API Documentation:** https://developers.greenhouse.io/harvest.html
- **Developer Portal:** https://developers.greenhouse.io/
- **SDKs/Libraries:** No official SDKs; community Python/Ruby/JS clients available. Harvest v3 is current (v1/v2 deprecated after August 31, 2026).
- **Standards:** REST/JSON; OpenAPI documented via third parties; webhook payloads signed with HMAC-SHA256.
- **Authentication:** Basic auth with API key as username; OAuth 2.0 supported for partner integrations.
- **Webhooks:** Supported for candidate stage changes, interview events, offer creation. Webhook signature verification uses HMAC-SHA256 header `X-Grnhse-Signature`.
- **Key Objects:** Candidates, Applications, Jobs, Interviews, Scorecards, Offers, Users.

### Lever (Data API)

- **Description:** Lever is a widely used ATS with a strong API ecosystem. The Data API provides full programmatic access to opportunities (candidate records), pipeline stages, feedback forms, offers, and users.
- **API Documentation:** https://hire.lever.co/developer/documentation
- **GitHub (Postings API):** https://github.com/lever/postings-api
- **Standards:** REST/JSON; webhook payloads signed with HMAC-SHA256 embedded in request body.
- **Authentication:** Basic auth with API key; OAuth 2.0 for partner integrations.
- **Webhooks:** Events include `candidateHired`, `candidateStageChange`, `applicationCreated`. Payloads signed; retries up to 5 times; HTTPS only.
- **Rate Limits:** 10 requests/second per API key (burst up to 20 r/s with token bucket algorithm).
- **Key Objects:** Opportunities, Contacts, Postings, Stages, Feedback, Tags, Files.

### Workday Recruiting (HCM API)

- **Description:** Workday is the dominant enterprise HCM and ATS for large organisations. Integration is significantly more complex than Greenhouse or Lever, with critical functionality split between SOAP and REST.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html (WWS v46.1)
- **REST API Guide:** https://community-content.workday.com/en-us/public/products/platform-and-product-extensions/soap-api-reference.html
- **Standards:** SOAP (primary for HCM/Recruiting) and REST. SOAP uses WSDL/XSD schemas; REST uses OAuth 2.0 and JSON.
- **Authentication:** REST: OAuth 2.0 with registered API client and scoped access tokens. SOAP: WS-Security headers with a dedicated integration service account (ISU).
- **Integration Complexity:** High; many critical recruiting objects (job requisitions, candidate applications, interview events) are SOAP-only in current versions.
- **Key Objects:** Job Requisitions, Job Applications, Candidates, Interview Events, Offer Letters, Workers.

### Merge.dev (Unified ATS API)

- **Description:** Merge provides a normalised unified API that aggregates 30+ ATS platforms into a single REST interface. BrightHire uses Merge to expand ATS integrations without per-platform engineering effort.
- **API Documentation:** https://docs.merge.dev/ats/
- **ATS Overview:** https://docs.merge.dev/merge-unified/ats/overview
- **SDKs:** Python, Node.js, Java, Go, Ruby. All available on the Merge documentation site.
- **Standards:** REST/JSON; normalised pagination and rate limiting; unified authentication layer.
- **Authentication:** API key per linked account; OAuth 2.0 for end-user ATS authorisation flows.
- **Key Objects (normalised):** Candidates, Applications, Jobs, Interviews (ScheduledInterview), Scorecards, Offers, Departments, Offices.
- **Note:** Particularly valuable for an interview intelligence platform targeting multiple ATS platforms simultaneously, allowing normalised read/write of interview and scorecard data.

### Zoom Meeting SDK & REST API

- **Description:** Zoom is the most widely used video conferencing platform in enterprise hiring. Its REST API and Meeting SDK enable building bots that join, record, and retrieve transcripts from Zoom meetings.
- **API Documentation:** https://developers.zoom.us/docs/api/
- **Meeting SDK:** https://developers.zoom.us/docs/meeting-sdk/
- **Standards:** REST/JSON; Server-to-Server OAuth for backend API access; WebSocket for real-time transcription events.
- **Authentication:** Server-to-Server OAuth 2.0 (recommended for interview recording bots).
- **Recording & Transcription:** `GET /meetings/{meetingId}/recordings` returns recording files including `TRANSCRIPT` (`.vtt` format). `recording.transcript_completed` webhook fires when VTT is available.
- **Key Limitations:** Host must have cloud recording enabled; transcript quality depends on Zoom's own ASR (typically lower accuracy than AssemblyAI/Deepgram); only available with paid plans.

### Microsoft Teams (Graph API)

- **Description:** Microsoft Teams is the dominant video/messaging platform in large enterprise. The Microsoft Graph API provides programmatic access to meeting recordings and transcripts.
- **API Documentation:** https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/meeting-transcripts/overview-transcripts
- **Graph API Reference:** https://learn.microsoft.com/en-us/graph/overview
- **Standards:** REST/JSON via Microsoft Graph; OAuth 2.0 with either organisation-wide application permissions or resource-specific consent (RSC).
- **Authentication:** OAuth 2.0 via Azure AD; two permission models: app-level (read all transcripts) or RSC (per-meeting consent from meeting organiser).
- **Transcript Format:** VTT or JSON. Available via `GET /me/onlineMeetings/{meetingId}/transcripts`.
- **Key Limitation:** Real-time transcript access during live meetings requires Teams Live Transcription (admin-enabled tenant setting).

### AssemblyAI (Speech-to-Text API)

- **Description:** AssemblyAI provides a high-accuracy speech-to-text API with speaker diarisation, PII redaction, topic detection, and auto-chapters. Widely used for post-interview transcript processing.
- **API Documentation:** https://www.assemblyai.com/docs
- **API Reference:** https://www.assemblyai.com/docs/api-reference/overview
- **GitHub (Python SDK):** https://github.com/AssemblyAI/assemblyai-python-sdk
- **Standards:** REST (pre-recorded) and WebSocket (streaming). JSON request/response; API key authentication.
- **Authentication:** API key in `Authorization` header.
- **Key Features:** Speaker diarisation (identifying separate speakers), PII redaction policy, language detection, low-latency streaming (~300 ms P50 for real-time), audio intelligence models (topic classification, sentiment, entity detection).
- **Relevance:** Primary alternative to platform-native transcription for higher accuracy. Critical for competency evidence extraction accuracy.

### Deepgram (Nova-3 API)

- **Description:** Deepgram offers competitive speech-to-text with ultra-low latency streaming optimised for real-time transcription during live interviews.
- **API Documentation:** https://developers.deepgram.com/docs/stt/getting-started
- **Migration Guide (from AssemblyAI):** https://developers.deepgram.com/docs/migrating-from-assembly-ai-speech-to-text-to-deepgram
- **Standards:** REST (pre-recorded) and WebSocket (streaming). JSON responses; API key authentication.
- **Authentication:** API key in `Authorization` header (Bearer token format).
- **Key Features:** Nova-3 model as of 2026 with improved speaker diarisation; ultra-low latency streaming suitable for real-time interviewer coaching; competitive with AssemblyAI on accuracy.
- **Relevance:** Alternative transcription provider for real-time use cases (coaching prompts during interviews).

### Recall.ai (Unified Meeting Recording API)

- **Description:** Recall.ai provides a single API for joining, recording, and obtaining transcripts from Zoom, Google Meet, Microsoft Teams, and Webex meetings via managed bots. Widely used by interview intelligence platforms including BrightHire.
- **API Documentation:** https://docs.recall.ai/
- **Standards:** REST/JSON. OAuth 2.0 for meeting platform authentication is managed by Recall internally.
- **Authentication:** API key.
- **Key Features:** Platform-agnostic bot joins meeting as a participant; real-time transcript stream via WebSocket; recording in MP4; VTT transcript; no host permissions required (bot joins as a named participant). Handles consent notifications automatically.
- **Relevance:** Most practical path to unified multi-platform video interview recording without managing Zoom/Teams/Meet SDKs separately.

### iCIMS Talent Cloud API

- **Description:** iCIMS is a large enterprise ATS with 300+ marketplace integrations. API access typically requires enterprise agreement; integration more complex than Greenhouse/Lever.
- **Developer Community:** https://developer-community.icims.com/getting-started/integrating-icims
- **Standards:** REST/JSON; documented API with enterprise access controls.
- **Authentication:** OAuth 2.0.
- **Key Objects:** Candidates, Jobs, Applications, Workflow Statuses, Assessments, Users.

### SmartRecruiters API

- **Description:** SmartRecruiters provides enterprise recruiting infrastructure with an open marketplace and REST API.
- **API Documentation:** https://developers.smartrecruiters.com/
- **Standards:** REST/JSON; OpenAPI documented.
- **Authentication:** API key and OAuth 2.0.
- **Key Objects:** Jobs, Candidates, Applications, Interviews, Assessments, Offers.

### Ashby API

- **Description:** Ashby is a modern API-first ATS popular with high-growth tech companies. Offers GraphQL alongside REST, making it developer-friendly.
- **API Documentation:** https://developers.ashbyhq.com/
- **Standards:** REST and GraphQL; OpenAPI available; all plans include API access.
- **Authentication:** API key.
- **Key Objects:** Job Postings, Applications, Candidates, Interviews, Scorecards, Users.
- **Note:** Ashby's GraphQL endpoint is particularly efficient for fetching nested interview and scorecard data in a single query.

---

## Notes

**Unified API Layer as Implementation Strategy**

Given the breadth of ATS integrations required (Greenhouse, Lever, Workday, iCIMS, SmartRecruiters, Ashby), building and maintaining direct integrations with each platform is significant engineering overhead. Services like Merge.dev and Unified.to provide normalised unified ATS APIs that abstract per-platform differences. BrightHire's use of Merge.dev demonstrates this pattern in production. A new platform should evaluate whether using a unified API layer (accepting the per-seat cost) or building direct integrations (accepting engineering cost) better fits the business model.

**Interview Recording Infrastructure**

The three major video conferencing platforms (Zoom, Teams, Google Meet) each have different API models, permission requirements, and transcript quality levels. Services like Recall.ai and Nylas Notetaker abstract these differences behind a single API. For a new platform, using Recall.ai or similar provides faster time-to-market at the cost of a dependency on a third-party recording infrastructure provider.

**Transcription Quality and Latency Trade-off**

Platform-native transcription (Zoom, Teams) is lower quality but zero marginal cost and immediately available in existing infrastructure. Third-party transcription (AssemblyAI, Deepgram) is higher quality but adds latency and cost per interview minute. Real-time coaching features (requiring live transcript) should use Deepgram's streaming API; post-interview analysis should use AssemblyAI's asynchronous API for highest accuracy.

**Regulatory Compliance as an API Feature**

NYC Local Law 144, EU AI Act Article 22, and GDPR Article 22 collectively require audit trails, bias audit data export, candidate notification workflows, and human oversight documentation. These are not policy concerns alone — they require specific API capabilities: exportable decision logs, bias metric reporting endpoints, candidate notification automation, and data deletion workflows. Building these as first-class API capabilities positions the platform for enterprise procurement.
