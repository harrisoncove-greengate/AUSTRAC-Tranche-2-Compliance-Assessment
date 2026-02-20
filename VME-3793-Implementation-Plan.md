# VME-3793 Enhanced PEP & Sanction Match Notification System - Implementation Plan [DRAFT]

**By Elia Scotto**  
**8 min read**

This initiative aims to overhaul the existing notification process for PEP (Politically Exposed Person) and Sanction screening matches. The current system provides insufficient context for support staff to effectively triage matches, leading to operational inefficiencies and potential risk.

---

## 1. Requirement Analysis

### Understanding & Goal
The current PEP & sanction notification system sends generic alerts that provide insufficient context for the support team. The current “Advanced search” URLs to OpenSanctions trigger API limits, and staff are required to navigate obvious false positives. The new email will be more intuitive, provide more information, and enable immediate manual resolution by directly linking to the internal VerifiMe Admin Portal.

**Example of the current email:**  
`Screenshot 2026-02-20 094450.png`

### Current Email Generation Flow
1. `PepSanctionLookupOutcome` contains minimal data (e.g., `id`, `matchFound`, `searchUrl`).
2. A processor converts this outcome into an `EmailNotificationMessage`.
3. `EmailNotificationMessage` currently only contains a single String message field (the email body).
4. `MessageGenerator` builds the SES Message object using:
   - **SubjectGenerator** → Supplies a subject line from configuration (static value).
   - **BodyGenerator** → Generates the email body content (TEXT or HTML).
5. `EmailNotificationMessageProcessor` sends the email using `SesClientWrapper`, which wraps the AWS SES client.

**Problem with Dynamic Search URLs**
- Current emails include an “Advanced Search” URL to OpenSanctions.
- These URLs are dynamic, contain query parameters, and trigger live API searches.
- Each click by support staff triggers a new API request.
- Multiple clicks can cause API limits to be exceeded.

---

### Problems Identified

| Problem                      | Technical Change                                                              | Expected Outcome                                                    |
|-------------------------------|-------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Lack of context in notifications | Include `name`, `confidence`, `entityType`, `country` in `PEPMatchDetails` and email | Support staff can quickly assess match severity                    |
| Operational risk: API limits   | Replace dynamic search URLs with static links using `entityID` and `dataSetID` | Avoid triggering OpenSanctions API, reduce failures               |
| Inefficient workflow           | Include `customerId` in email subject and generate `internalLink`             | Staff can go directly to the customer record in Admin Portal       |
| Outdated email format          | Update `EmailNotificationMessage` and body generators to HTML/structured format | Richer email content, easier to read and act upon                 |
| Incomplete message data        | Enrich outcome from Lambda with full `PEPMatchDetails`                         | All necessary match details are available downstream for emails   |
| Mock/testing gaps              | Update mock data and unit tests to include `customerId` and match details     | Ensure end-to-end testing validates rich email generation         |

---

## Scope and Constraints

**In-Scope:**
- **Email Notifications:** redesign to include structured HTML, confidence scores, entity type, country, and dataset.
- **Enriched Payloads:** convert minimal Match/No Match results into full `PEPMatchDetails` objects.
- **Static Linking:** all external links point to static entity/dataset URLs.
- **Internal Portal Integration:** emails include direct links to the VerifiMe Admin Portal.
- **Archiving:** store full `PEPMatchDetails` for auditing and compliance.
- **Testing & Validation:** unit, component, integration, and acceptance tests for accuracy and client compatibility.

**Out-of-Scope:**
- OpenSanctions platform changes.
- Admin Portal UI redesign.
- Automated decision-making (manual triage only).

---

## 2. Technical Approach

### Data Retention
Enhance `pepSanctionLookupResult` Lambda to parse and store relevant match details, not just Match/No Match. Archive `PEPMatchDetails` for auditing.

```typescript
type MatchStatus = "Match" | "No Match";

interface PEPMatchDetails {
  matchStatus: MatchStatus;
  name: string;
  confidence: number;
  entityType: string;
  country: string;
  dataSetID: string;
  entityID: string;
  customerId: string;
}

const mockPEPMatches: PEPMatchDetails = {
  matchStatus: "Match",
  name: "John A Doe",
  confidence: 95,
  entityType: "Person",
  country: "US",
  dataSetID: "us_sanctions_2026",
  entityID: "12345",
  customerId: "67890"
};```
Key Areas of Codebase to Modify

Modify PepSanctionLookupOutcome to include customerId.

Update EmailNotificationMessage to hold structured match data and links.

Update BodyGenerator and MessageGenerator to generate HTML emails.

Update SubjectGenerator to include customerId.

Update notification processors and SQS message converters.

Update tests and mock data for enriched payloads.

Safe Linking

Use static URLs for entity and dataset links.

Use internal portal links for direct access.

const dossierLink = `https://opensanctions.org/entities/${match.entityID}/`;
const datasetLink = `https://opensanctions.org/datasets/${match.dataSetID}/`;
const internalLink = `https://admin-portal.verifime.com/admin/individual/dashboard/{customerId}`;
Email Construction Example
From: Verifime <no-reply@verifime.com>
To: Verifime Support
CC: N/A
Subject: https://admin-portal.verifime.com/admin/individual/dashboard/{customerId}

Body:

Matched Name   Confidence  Entity Type  Country  Entity Dossier  Source Dataset
John A Doe     92%         Person       US       View Entity     View Dataset
3. Testing Strategy
Test Layers

Request Lambda converts OpenSanctions responses to enriched PEPMatchDetails.

Static links are verified to never trigger live API calls.

Email rendering is verified for subject, HTML body, and safe links.

End-to-end pipeline: SQS → Lambda → SNS → SQS → Notification Lambda → SES → Archive.

Edge cases: missing IDs, multiple matches, malformed data.

Performance: tolerates expected throughput.

Security: no sensitive data leaked; emails are XSS-safe.

A. Unit Tests

Validate parsers, URL builders, email subject, and HTML fragments.

Example:

buildEntityLink("abc123"); // "https://opensanctions.org/entities/abc123/"
buildInternalLink("0e27..."); // "https://admin-portal.verifime.com/admin/individual/dashboard/0e27..."
B. Component Tests

Validate Lambda functions in isolation with HTTP stubs.

Ensure correct SES payloads and S3 archiving.

C. Integration Tests

Validate SNS/SQS end-to-end with mocked external APIs.

Verify enrichment, notifications, and archive entries.

D. End-to-End & Acceptance Tests

Deploy in pre-prod/staging.

Run synthetic scenarios:

Single high-confidence match

Multiple matches

No matches

Missing fields

Validate email rendering, internal portal links, and archiving.

E. Email Rendering & Client Compatibility Tests

Validate HTML rendering across Gmail, Outlook, Apple Mail.

Fallback to anchor links for unsupported features.

F. Performance & Load Tests

Simulate realistic arrival patterns and bursts.

Validate Lambda concurrency, retries, SES throttling.

Acceptance: no live API overload, <1% errors under baseline load.

4. Implementation Steps
Enrich and Publish the Outcome Payload

Modify pepSanctionLookupResult Lambda:

Parse full match data.

Construct PEPMatchDetails.

Replace minimal outcome in PepSanctionLookupOutcomeTopic.

Update archiving to store enriched payloads.

Update SQS/SNS converters.

Refactor Notification Construction

Update EmailNotificationMessage for structured data.

Update PepSanctionOutcomeEmailNotificationProcessor:

Subject: PEP | {customerId}

Construct static links for entity, dataset, and internal portal.

Update BodyGenerator and MessageGenerator for HTML emails.

Update Tests, Mocks, and Validation

Update mock payloads to include full PEPMatchDetails.

Add unit tests for subject, static links, HTML body.

Ensure no external API calls from emails.

Validate archiving works with enriched payloads.
