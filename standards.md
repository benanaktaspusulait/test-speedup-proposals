# Confluence Documentation Standards

## 1. Purpose

This document defines a common standard for creating, maintaining, and structuring Confluence documentation for technical proposals, improvement plans, KT material, decision records, and implementation guides.

The aim is to make documentation easier to access, review, collaborate on, and use for decision-making across technical and non-technical teams.

Going forward, Confluence is the primary authoring platform for shared technical documentation. New proposals, approach documents, KT pages, and decision records should be created directly in Confluence following this standard. Existing Confluence pages should be updated to align with this structure over time.

A small number of existing GitLab repositories still contain proposals and technical documentation that need to be migrated to Confluence. This standard also covers that migration process (see Section 8). Once those repositories are migrated, the migration guidance remains available for reference but the day-to-day workflow is Confluence-first.

---

## 2. Objectives

The objectives of this standard are to:

* Provide a consistent structure for Confluence documentation — whether created from scratch, migrated from GitLab, or updated over time.
* Make technical information easier for non-technical stakeholders to understand.
* Separate high-level decision information from deep technical implementation details.
* Improve collaboration, review, and feedback.
* Support delivery, architecture, and platform-level decision-making.
* Make proposals easier to prioritise using risk, complexity, effort, value, and MoSCoW ratings.
* Ensure all teams produce documentation in a consistent style.
* Make creating new Confluence pages faster by providing clear templates and patterns.
* Make updating existing pages straightforward by defining what each section should contain.

---

## 3. When to Use This Standard

This standard applies in three situations:

### 3.1 Creating New Confluence Pages (Primary Workflow)

Use this standard when creating Confluence pages from scratch for:

* CI/CD improvement proposals.
* Test pipeline optimisation proposals.
* Deployment process documentation.
* KT and onboarding documentation.
* Known issues and solution suggestions.
* Technical investigation notes.
* Architecture or platform improvement proposals.
* Cross-team technical improvement work.
* DACI decision records.
* Implementation guides.
* Any documentation that needs review from engineering, architecture, delivery, or product stakeholders.

### 3.2 Updating Existing Confluence Pages

Use this standard when updating or restructuring existing pages to:

* Align with the standard structure.
* Add missing sections (e.g. proposal matrix, phased plan, DACI areas).
* Improve readability for wider stakeholders.
* Add metadata (owner, status, last updated).
* Move technical details lower in the page.

### 3.3 Migrating from GitLab (Transitional)

Use this standard when converting existing GitLab repository documentation to Confluence. This applies to the remaining repositories that still hold proposals or technical documentation in Markdown. Once those migrations are complete, this workflow becomes dormant but the guidance remains available for reference.

**Current migration backlog:**

- `test-speedup-proposals` — CI/CD and test pipeline speed-up proposals (in progress)
- TBC — additional repositories to be confirmed

Once all repositories in this list are migrated, Section 8.3 becomes reference-only and should be marked as complete.

---

## 4. Key Principles

### 4.1 Confluence as the Primary Platform

Confluence is the primary platform for shared technical documentation. New proposals, decision records, KT material, and technical guides should be authored directly in Confluence.

GitLab remains the source for:

* Code.
* Build configuration.
* Pipeline configuration.
* Repository-level technical files (e.g. README for developer setup).
* Implementation-specific files that live alongside the code.
* Version-controlled technical artefacts.

Confluence should be used for:

* Proposals and approach documents.
* Decision records (DACI).
* Cross-team technical summaries.
* Delivery improvement plans.
* KT and onboarding material.
* Process documentation.
* Known issues and resolution guides.
* Stakeholder-friendly summaries.
* Implementation guides that need wider review.

If information belongs in both places (e.g. a developer setup guide), keep the GitLab version as the source of truth for developers and create a Confluence summary that links to it.

---

### 4.2 Front-Load the Summary

Each Confluence page should start with a clear summary before deep technical content.

The first section should help a reader quickly understand:

* What this page is about.
* Why it matters.
* What problem it solves.
* What outcome is expected.
* Whether a decision or action is required.

Avoid starting with long technical details, command examples, or repository-specific implementation notes.

---

### 4.3 Separate Decision Content from Technical Detail

Decision-makers often need a concise view of value, risk, complexity, effort, and impact.

Technical readers may need deeper implementation details.

The page should support both groups by placing:

* Executive summary, problem statement, approach, proposal matrix, and phased plan near the top.
* Detailed technical notes, commands, configuration examples, and file references lower down.

---

### 4.4 Use “Approach” Instead of “Recommended Approach”

Avoid wording such as:

* Recommended approach
* Recommendation
* Suggested only
* Maybe we should

Prefer clearer wording such as:

* Approach
* Proposed approach
* Delivery approach
* Implementation approach
* Candidate approach

This makes the document feel more structured and actionable while still allowing discussion.

---

### 4.5 Do Not Invent Missing Information

When creating or updating Confluence pages, do not guess missing details.

Use:

```text
TBC
```

where information is not yet confirmed.

Examples:

* Owner: TBC
* Target date: TBC
* Value: TBC
* Effort: TBC
* Risk: TBC
* Success metric: TBC

**TBC resolution rule:** All TBC items must be resolved before a page moves to "Approved" status. Pages in "Draft" or "In Review" may contain TBC items, but reviewers should flag them for resolution. A page should not remain in "In Review" indefinitely with unresolved TBCs — set a deadline or assign an owner to each TBC.

---

## 5. Standard Confluence Page Structure

Each converted proposal or technical improvement page should follow this structure where applicable.

---

# Page Title

Use a clear title that describes the purpose of the page.

Examples:

* CI/CD and Test Pipeline Speed-Up Approach
* Deployment Process Improvement Approach
* KT and Repository Onboarding Approach
* Testcontainers Cache Strategy
* Maven Surefire and Failsafe Configuration Approach
* E2E Test Harness Improvement Approach

---

## 1. Executive Summary

A short summary for readers who only have a few minutes.

Include:

* What the proposal or document is about.
* Why it matters.
* The main problem.
* The expected outcome.
* Whether any decision or review is needed.

Example:

```text
This page summarises proposed improvements to reduce CI/CD and test pipeline execution time. The current pipeline feedback cycle can be slow, which affects developer productivity and may delay urgent fixes. The approach is to identify low-risk quick wins first, then assess medium and larger structural improvements using value, risk, complexity, and effort.
```

---

## 2. Context / Problem Statement

Explain the background and current problem.

Include:

* What is happening today.
* Why it is a problem.
* Who is affected.
* What risk it creates.
* Why it is worth addressing.

Example areas:

* Long-running tests.
* Slow CI feedback loop.
* Deployment delays.
* Repeated manual investigation.
* Difficult onboarding.
* Fragmented documentation.
* Cross-team dependency issues.

---

## 3. Objectives

List the main objectives.

Example:

```text
The objectives are to:
- Reduce pipeline and test execution time.
- Improve developer feedback cycle.
- Reduce avoidable CI/CD bottlenecks.
- Improve delivery confidence.
- Keep test coverage and reliability stable.
- Identify quick wins and longer-term improvements.
- Provide a structured approach for review and prioritisation.
```

---

## 4. Scope

### In Scope

List what the page covers.

Example:

```text
- Maven Surefire and Failsafe configuration.
- JUnit 5 parallel execution options.
- Testcontainers and Docker caching.
- E2E test harness improvements.
- CI pipeline optimisation opportunities.
```

### Out of Scope

List what the page does not cover.

Example:

```text
- Rewriting all tests.
- Replacing the CI platform.
- Large architectural redesign.
- Production runtime performance optimisation.
```

Use `TBC` if the scope is not clear yet.

---

## 5. Approach

Describe the high-level approach.

This should explain:

* How the work will be reviewed.
* How proposals will be prioritised.
* Whether work will be phased.
* Whether separate DACI decision records are needed.
* Whether quick wins will be separated from larger changes.

Example:

```text
The approach is to group improvement opportunities into target areas, assess each proposal using value, risk, complexity, and effort, and then prioritise the work into phases. Low-risk and high-value items should be considered first. Larger or cross-team changes may require separate DACI-style decision records before implementation.
```

---

## 6. Proposal Overview Matrix

Each proposal should be summarised in a matrix.

| Proposal         | Description       |  Value |   Risk | Complexity | Effort | MoSCoW | Notes |
| ---------------- | ----------------- | -----: | -----: | ---------: | -----: | ------ | ----- |
| Example proposal | Short explanation |   High |    Low |        Low |    Low | Must   | TBC   |
| Example proposal | Short explanation | Medium | Medium |       High | Medium | Should | TBC   |

Use standard values:

### Value

* High
* Medium
* Low
* TBC

### Risk

* High
* Medium
* Low
* TBC

### Complexity

* High
* Medium
* Low
* TBC

### Effort

* High
* Medium
* Low
* TBC

### MoSCoW

* Must
* Should
* Could
* Won’t
* TBC

Guidance:

* Low risk + low effort + low complexity + high value = strong quick win.
* High risk + high complexity + unclear value = needs more investigation.
* Anything with cross-team impact may need a DACI record.

---

## 7. Phased Plan

Where possible, structure the work into phases.

### Phase 1: Low-Risk Quick Wins

Include:

* Objective
* Candidate changes
* Expected outcome
* Success criteria
* Risks / dependencies

### Phase 2: Medium-Complexity Improvements

Include:

* Objective
* Candidate changes
* Expected outcome
* Success criteria
* Risks / dependencies

### Phase 3: Larger Structural Improvements

Include:

* Objective
* Candidate changes
* Expected outcome
* Success criteria
* Risks / dependencies

### Phase 4: Optional / Future Improvements

Include:

* Objective
* Candidate changes
* Expected outcome
* Success criteria
* Risks / dependencies

Use only the phases that make sense for the topic.

---

## 8. Success Criteria

Define how success will be measured.

Examples:

```text
- Pipeline execution time reduced.
- Test execution time reduced.
- Developer feedback cycle improved.
- No reduction in test coverage.
- No reduction in test reliability.
- Fewer avoidable pipeline failures.
- Urgent fixes can move through the pipeline with less delay.
- KT material becomes easier to find and reuse.
```

Use exact numbers only if they are already known. Otherwise use `TBC`.

---

## 9. Risks and Mitigations

Use a table to make risks easy to review.

| Risk | Impact | Mitigation | Owner / Follow-up |
| ---- | ------ | ---------- | ----------------- |
| TBC  | TBC    | TBC        | TBC               |

Examples of risk areas:

* Test reliability risk.
* Flaky tests.
* Pipeline instability.
* Cross-team dependency.
* Version compatibility.
* Missing ownership.
* Lack of access.
* Documentation becoming outdated.

---

## 10. DACI / Decision Areas

Some proposals may need a separate DACI-style decision record.

Use this section when a proposal:

* Affects multiple teams.
* Changes shared libraries.
* Changes CI/CD behaviour.
* Has delivery risk.
* Requires ownership agreement.
* Needs architecture or platform review.
* Requires prioritisation from senior stakeholders.

| Decision Area | Why DACI May Be Needed | Suggested Participants | Status |
| ------------- | ---------------------- | ---------------------- | ------ |
| TBC           | TBC                    | TBC                    | TBC    |

Example decision areas:

* JUnit 5 parallel execution.
* Maven Surefire / Failsafe configuration changes.
* Docker / Testcontainers cache strategy.
* E2E test harness improvements.
* CI pipeline sharing options.
* Shared dependency version alignment.
* Deployment process changes.

---

## 11. Technical Details

Place deep technical content here.

This section may include:

* Maven configuration notes.
* Surefire / Failsafe configuration.
* JUnit 5 notes.
* Docker / Testcontainers details.
* CI configuration examples.
* Build logs.
* Pipeline timings.
* Repository file references.
* Code snippets.
* Commands.
* Investigation notes.

Technical details should be organised and readable, but they should not overload the top of the page.

---

## 12. References

List all supporting material.

Examples:

* GitLab repository links.
* GitLab MR links.
* Pipeline examples.
* Relevant source files.
* Existing Confluence pages.
* Related Teams discussions.
* Related DACI records.
* Existing architecture or delivery documents.

---

## 6. Multi-Page Confluence Structure

For large repositories, broad proposals, or documents with several decision areas, avoid creating one long Confluence page.

Use a parent page with focused child pages. This keeps the overview easy for stakeholders to read while still preserving the technical detail needed by engineers.

Suggested structure:

```text
Parent page: Overview and Approach
  ├── Child: Proposal Overview Matrix
  ├── Child: Phased Plan
  ├── Child: Risks and DACI
  ├── Child: Technical Details
  └── Child: References
```

### 6.1 Parent Page

The parent page should be short and decision-friendly.

Include:

* Executive summary.
* Context / problem statement.
* Objectives.
* Scope.
* High-level approach.
* Links to child pages.
* Short success summary.

Avoid putting long tables, command examples, code snippets, or deep technical investigation notes on the parent page.

### 6.2 Child Pages

Use child pages to keep information focused.

| Child Page | Purpose |
| ---------- | ------- |
| Proposal overview matrix | Value, risk, complexity, effort, MoSCoW, and short notes |
| Phased plan | Phase purpose, candidate changes, expected outcome, success criteria, risks |
| Risks and DACI | Main risks, mitigations, ownership, and decision areas |
| Technical details | Commands, configuration examples, implementation notes, code snippets |
| References | Source documents, GitLab links, pipeline evidence, external docs |

### 6.3 When to Split a Page

Split content into a child page when:

* The page becomes hard to review in one sitting.
* A table dominates the page.
* Technical detail distracts from the decision summary.
* Multiple stakeholder groups need different levels of detail.
* A topic may need a separate DACI-style decision.
* A section is likely to be updated independently.

### 6.4 Page Length Guidance

There is no strict line limit, but pages should stay easy to scan.

As a practical guide:

* Parent overview pages should be concise.
* Matrix, phased plan, risks, and references pages should usually be short.
* Technical details pages may be longer, but should still be split if they cover unrelated topics.

If in doubt, split the content and link between pages.

---

## 7. Link Handling Standard

When moving content from GitLab to Confluence, links must be checked carefully.

### 7.1 Relative Links

GitLab Markdown often contains relative links such as:

```text
./README.md
../pom.xml
docs/test-speedup.md
src/test/java/...
```

These may not work in Confluence because Confluence does not understand the GitLab repository structure.

### 7.2 Absolute GitLab Links

Important links should be converted to absolute GitLab links where possible, provided the GitLab instance is accessible to the intended Confluence audience.

Guidelines:

- If the GitLab instance is internal but accessible to all Confluence readers (e.g. organisation-wide GitLab), absolute links are safe to include.
- If the GitLab instance requires specific access permissions that not all readers have, add a note: "(requires GitLab access)".
- If the URL is classified as restricted (e.g. points to a security-sensitive internal system), do not include it — use inline code with the file path instead.

Examples:

```text
GitLab MR: <absolute MR link> (requires GitLab access)
Pipeline example: <absolute pipeline link>
Relevant file: <absolute file link>
```

### 7.3 File Paths

If a relative link is only a file path and cannot be safely converted, keep it as inline code.

Example:

```text
Relevant file: `pom.xml`
Relevant package: `src/test/java/...`
```

### 7.4 Unknown Links

If a link cannot be verified, mark it clearly:

```text
TODO: verify link
```

Do not create fake or guessed links.

---

## 8. Creating and Updating Confluence Pages

### 8.1 Creating a New Page from Scratch

When starting a new Confluence page:

#### Step 1: Choose the Page Type

Select the appropriate standard page type (see Section 9):

* Proposal / Approach page.
* DACI decision record.
* Technical reference page.
* KT / onboarding page.
* Known issue and resolution page.
* Implementation guide.

#### Step 2: Apply the Structure

Use the section structure defined for that page type. Start with the high-level sections and add detail progressively.

Do not start writing from the middle or bottom. Begin with:

* Title (clear, descriptive).
* Executive summary or purpose.
* Problem or context.
* Then add depth.

#### Step 3: Add Metadata

Include at the top or in a Confluence panel:

```text
Owner: <name or team>
Status: Draft
Created: YYYY-MM-DD
Last updated: YYYY-MM-DD
```

#### Step 4: Apply Labels

Apply standard labels (see Section 16) for discoverability.

#### Step 5: Share for Review

Share the draft with relevant stakeholders. Update status to "In Review" when ready for feedback.

---

### 8.2 Updating an Existing Page

When updating a page that already exists in Confluence:

#### Step 1: Check Current Structure

Compare the existing page against the standard structure for its type.

#### Step 2: Identify Gaps

Common gaps in older pages:

* Missing executive summary.
* Technical details at the top instead of lower down.
* No proposal matrix or phased plan.
* No metadata (owner, status, last updated).
* No success criteria.
* No risks section.
* Missing labels.

#### Step 3: Restructure If Needed

If the page needs significant restructuring:

* Keep the existing content — do not delete information.
* Move sections into the standard order.
* Add missing sections with `TBC` where information is not yet available.
* Move deep technical content lower.

#### Step 4: Update Metadata

Update:

```text
Last updated: YYYY-MM-DD
Last reviewed: YYYY-MM-DD
Status: <current status>
```

#### Step 5: Notify Stakeholders

If the restructuring changes how people find information on the page, notify relevant readers.

---

### 8.3 Migrating from GitLab (Transitional)

This section applies to the remaining GitLab repositories that need conversion. Once those are complete, this workflow becomes dormant.

#### Step 1: Review the Repository

Identify:

* README files.
* Docs folder.
* Existing Markdown proposals.
* CI/CD configuration.
* Pipeline notes.
* KT material.
* Known issue documents.
* Architecture notes.
* Scripts or commands that need explanation.
* Relevant MRs or issues.

#### Step 2: Classify the Content

Group content into categories:

* Proposal.
* KT / onboarding.
* Technical reference.
* Known issue.
* Implementation guide.
* Decision record.
* Process documentation.
* CI/CD or pipeline improvement.
* Dependency or version management.

#### Step 3: Decide the Confluence Page Type

Choose one of the standard page types (Section 9).

#### Step 4: Apply the Correct Structure

Use the relevant Confluence structure for the chosen page type.

#### Step 5: Move Technical Details Lower

Keep the top of the page easy to read. Move commands, code snippets, configuration examples, and detailed implementation notes into lower sections or child pages.

#### Step 6: Add Decision Metadata

Where relevant, add:

* Value, Risk, Complexity, Effort, MoSCoW.
* Owner, Status, Dependencies.
* Success criteria.

#### Step 7: Validate Links

Check:

* Relative links (convert to absolute or inline code).
* GitLab links (verify they are accessible to readers).
* MR and pipeline links.
* Mark unverifiable links as `TODO: verify link`.

#### Step 8: Review with Stakeholders

Share the page for review with relevant people.

---

## 9. Standard Page Types

### 9.1 Proposal / Approach Page

Use for improvement proposals or future work.

Typical sections:

* Executive Summary
* Context / Problem Statement
* Objectives
* Scope
* Approach
* Proposal Matrix
* Phased Plan
* Success Criteria
* Risks and Mitigations
* DACI / Decision Areas
* Technical Details
* References

---

### 9.2 DACI Decision Record

Use when a decision requires multiple stakeholders.

Suggested structure:

```text
# DACI: <Decision Title>

## Decision Summary
## Context
## Problem
## Options Considered
## Proposed Approach
## Decision Needed
## DACI Roles
## Risks
## Impact
## Decision Outcome
## Follow-up Actions
## References
```

### DACI Roles

| Role | Definition |
|------|-----------|
| Driver | The person responsible for driving the decision to completion. Gathers input, facilitates discussion, and ensures a decision is made on time. Does not make the final call. |
| Approver | The person (or people) with authority to make the final decision. Accountable for the outcome. |
| Contributors | People whose input and expertise are sought. They provide options, analysis, and advice but do not have decision authority. |
| Informed | People who need to know the outcome but are not involved in making the decision. Notified after the decision is made. |

Note: In DACI pages, when comparing options before a decision is made, use "Proposed Approach" or "Preferred Option". Avoid "Recommendation" to stay consistent with the tone guidance in Section 4.4. Once a decision is finalised, the chosen direction should be described as the "Decision Outcome".

---

### 9.3 Technical Reference Page

Use for stable technical knowledge.

Suggested structure:

```text
# <System / Component> Technical Reference

## Purpose
## Audience
## System Overview
## Key Components
## Dependencies
## Configuration
## Local Development
## Testing
## Deployment
## Observability
## Common Issues
## References
```

---

### 9.4 KT / Onboarding Page

Use for developer onboarding or knowledge transfer.

Suggested structure:

```text
# <Repository / System> KT Guide

## Purpose
## Audience
## Prerequisites
## Repository Overview
## Local Setup
## How to Run
## How to Test
## Common Tasks
## Common Issues
## Important Links
## Glossary
## References
```

---

### 9.5 Known Issue / Resolution Page

Use for recurring technical problems.

Suggested structure:

```text
# Known Issue: <Issue Title>

## Summary
## Impact
## Symptoms
## Root Cause
## Workaround
## Permanent Fix
## Affected Repositories
## Owner
## Status
## References
```

---

### 9.6 Implementation Guide

Use when a change needs step-by-step execution.

Suggested structure:

```text
# Implementation Guide: <Change Title>

## Objective
## Prerequisites
## Assumptions
## Step-by-Step Plan
## Validation
## Rollback Plan
## Risks
## References
```

---

## 10. Naming Standards

Use clear and consistent page names.

### Good Examples

```text
CI/CD and Test Pipeline Speed-Up Approach
Deployment Process Improvement Approach
Repository KT Guide - <Repository Name>
Known Issue - JUnit 5 Assertion Signature Change
DACI - Testcontainers Cache Strategy
Implementation Guide - Maven Surefire Parallel Execution
```

### Avoid

```text
Notes
Ideas
Random Findings
Recommended Improvements
Stuff to Fix
My Suggestions
```

---

## 11. Tone and Writing Style

Confluence pages should be:

* Clear.
* Practical.
* Structured.
* Easy to scan.
* Professional.
* Decision-friendly.
* Accessible to both technical and non-technical readers.

Avoid:

* Too much technical detail at the top.
* Long unstructured notes.
* Overuse of “maybe”.
* Overuse of “recommend”.
* Unclear ownership.
* Unverified claims.
* Broken relative links.
* Large code blocks before the problem is explained.

Prefer:

* “Approach”
* “Proposal”
* “Candidate change”
* “Expected outcome”
* “Success criteria”
* “Risk”
* “Mitigation”
* “TBC” where something is unknown

---

## 12. Review Checklist

Before publishing or sharing a Confluence page, check:

```text
[ ] Is there a clear executive summary?
[ ] Is the problem statement clear?
[ ] Are objectives defined?
[ ] Is the scope clear?
[ ] Are proposals grouped logically?
[ ] Is there a risk / complexity / effort / value view?
[ ] Is there a MoSCoW rating where useful?
[ ] Is there a phased plan?
[ ] Are success criteria included?
[ ] Are risks and mitigations documented?
[ ] Are DACI candidates identified where needed?
[ ] Are technical details moved lower down?
[ ] Are relative GitLab links fixed or marked as TODO?
[ ] Are references listed clearly?
[ ] Is the page readable by non-technical stakeholders?
[ ] Is the tone structured and professional?
[ ] Are there any secrets, tokens, credentials, or PII accidentally included?
[ ] Is metadata present (owner, status, last updated)?
[ ] Are standard labels applied?
[ ] Are all TBC items acceptable for the current page status?
```

---

## 13. Security and Sensitive Data

### 13.1 What Must Not Be Copied to Confluence

When converting GitLab content, the following must never appear on a Confluence page:

- Passwords, tokens, API keys, or secrets of any kind.
- `.env` file contents or environment variable values that contain credentials.
- Private SSH keys or certificate material.
- Internal IP addresses, hostnames, or URLs that are classified as restricted.
- Personal data (PII) beyond what is necessary for the document's purpose.
- Unredacted security scan results or vulnerability details that have not been triaged.

### 13.2 How to Handle Sensitive References

If a GitLab file contains sensitive values that are relevant to the documentation context:

- Reference the file by name (e.g. "see `.env.example` in the repository") without reproducing values.
- Use placeholder notation: `${SECRET_NAME}` or `<redacted>`.
- Describe what the value does, not what it is.

### 13.3 CI/CD Configuration

Pipeline configuration files (`.drone.star`, `.gitlab-ci.yml`, `Jenkinsfile`) may contain:

- Registry URLs — acceptable to include if not restricted.
- Service account names — check classification before including.
- Inline secrets or token references — must be removed or replaced with placeholders.

When in doubt, check with the security team before publishing.

---

## 14. Diagrams and Visual Content

### 14.1 ASCII Diagrams

GitLab Markdown often contains ASCII art diagrams. These render well in monospace code blocks but can be hard to read in Confluence's proportional fonts.

Options:

| Approach | When to Use |
|----------|-------------|
| Keep as code block | Simple flow diagrams that are short and still readable |
| Convert to Confluence draw.io diagram | Complex architecture or flow diagrams that need to be maintained |
| Convert to Mermaid (if Confluence plugin available) | Sequence diagrams, flowcharts, or state diagrams. Requires a Mermaid plugin (e.g. "Mermaid Diagrams for Confluence" or equivalent). Check with your Confluence admin whether the plugin is installed. |
| Screenshot from rendered Markdown | Quick conversion where the diagram is stable and unlikely to change |

### 14.2 Images and Screenshots

- Store images as Confluence page attachments, not as external links that may break.
- Add alt text or a caption to every image.
- Avoid screenshots of text — use actual text content instead where possible.
- If a diagram is likely to change, prefer an editable format (draw.io, Mermaid) over a static image.

### 14.3 Tables

GitLab Markdown tables transfer directly to Confluence. No conversion is needed for simple tables.

For complex tables with many columns, consider using Confluence's responsive table macro or splitting into multiple tables.

---

## 15. Confluence Macros and Formatting

### 15.1 Recommended Macros

Use Confluence macros to improve readability:

| Macro | When to Use |
|-------|-------------|
| Table of Contents | Pages longer than 5 sections |
| Expand | Long code blocks, verbose technical details, or optional context |
| Status | Showing proposal status (Draft, In Review, Approved, In Progress, Done) |
| Info / Note / Warning panels | Calling out important context, caveats, or prerequisites |
| Children Display | Parent pages that link to child pages |
| Jira Issue macro | Linking proposals to Epics or Stories |

When a proposal has been converted into Jira work items (Epics, Stories, Tasks), use the Jira Issue macro to link directly from the Confluence page. This creates a live connection between the proposal and its delivery tracking.

### 15.2 Status Labels

Use consistent status indicators on proposal pages:

| Status | Meaning |
|--------|---------|
| Draft | Initial conversion, not yet reviewed |
| In Review | Shared with stakeholders for feedback |
| Approved | Accepted for implementation |
| In Progress | Work has started |
| Done | Implemented and verified |
| Parked | Deferred or deprioritised |

### 15.3 Code Blocks

- Always specify the language for syntax highlighting (e.g. `java`, `xml`, `bash`, `properties`).
- Keep code blocks short in overview sections — use Expand macros for long examples.
- Prefer inline code (backticks) for file names, commands, and short references within prose.

---

## 16. Labels and Discoverability

### 16.1 Standard Labels

Apply consistent Confluence labels to help with search and filtering:

| Label | When to Apply |
|-------|---------------|
| `proposal` | Any proposal or approach page |
| `ci-cd` | CI/CD related content |
| `test-pipeline` | Test execution improvements |
| `daci` | Decision records |
| `kt` | Knowledge transfer pages |
| `known-issue` | Known issue documentation |
| `implementation-guide` | Step-by-step guides |
| `technical-reference` | Stable technical reference documentation |
| The repository name (e.g. `fdp-cmd-adaptor-sns`) | Any page relating to a specific repository |
| The team name | Any page owned by a specific team |

### 16.2 Label Rules

- Use lowercase, hyphenated labels.
- Apply at least 2 labels per page (type + topic).
- Do not create labels that duplicate existing ones (check before creating).
- Review labels when page status changes.
- Labels are applied at the page level. They do not inherit from parent pages or spaces.
- To find all pages with a specific label, use Confluence's label search or the "Content by Label" macro.

---

## 17. Versioning and Keeping Pages Current

### 17.1 The Staleness Problem

Confluence pages can become outdated quickly if:

- The source repository changes after conversion.
- A proposal is partially implemented but the page is not updated.
- Ownership is unclear and no one maintains the page.

### 17.2 Page Freshness Indicators

Add a metadata section at the top or bottom of each page:

```text
Last updated: YYYY-MM-DD
Last reviewed: YYYY-MM-DD
Owner: <name or team>
Status: Draft / In Review / Approved / In Progress / Done / Parked
Source: <GitLab repository link or "original content">
```

**Owner format:** Use one of the following consistent formats:

- Individual: `FirstName LastName (@slack-handle)` or `FirstName LastName (team-name)`
- Team: `Team-Name`

Confluence also supports `@mentions` — use the Confluence user mention where possible so that ownership is clickable and verifiable.

**Note:** Confluence displays its own "Last modified" timestamp automatically. The manual "Last reviewed" field is still useful because a page can be modified without being meaningfully reviewed.

### 17.3 Review Cadence

- Proposal pages: review status monthly until implemented or parked.
- KT / onboarding pages: review quarterly or when the repository changes significantly.
- Technical reference pages: review when the referenced system changes.
- Known issue pages: update immediately when the issue is resolved.

### 17.4 Archiving

When a page is no longer relevant:

- Move it to an "Archive" space or parent page.
- Add a clear banner: "This page is archived and may no longer be accurate."
- Do not delete pages — they may still have historical value or be linked from other pages.
- If other pages link to the archived page, those links will continue to work (Confluence preserves links on move). No redirect is needed.
- Add a `deprecated` or `archived` label to the page for filtering.
- Pages should never be deleted. If content is outdated, archive it; if it is wrong, correct it.

---

## 18. Access and Permissions

### 18.1 Default Access

Confluence pages following this standard should generally be accessible to:

- The owning team.
- Related engineering teams.
- Architecture and platform teams.
- Delivery and management stakeholders.

### 18.2 Restricted Content

If a page contains:

- Security-sensitive implementation details.
- Unresolved vulnerability information.
- Commercially sensitive information.

Then restrict access to the relevant group and add a note explaining why access is restricted.

### 18.3 External Sharing

Do not share Confluence pages externally (outside the organisation) without explicit approval. This applies even if the source GitLab content is in a public or shared repository.

Approval must be obtained from both:

- The page owner (or team lead if owner is unavailable).
- The security team or information governance lead.

Document the approval in the page comments or link to the relevant approval thread.

---

## 19. General Policies

### 19.1 Documentation Language

All Confluence documentation covered by this standard should be written in English. This ensures consistency across teams and readability for all stakeholders including architecture, delivery, and platform teams.

### 19.2 Feedback Mechanism

Each page should include a way for readers to provide feedback. Options:

- Add a line at the bottom of the page: "Feedback or questions? Contact the page owner or comment below."
- Use Confluence's built-in page comments for inline feedback.
- For broader feedback, link to a relevant Slack channel or Teams thread.

### 19.3 Accessibility and Readability

- Avoid relying solely on colour to convey meaning (e.g. in status indicators).
- Keep table column widths reasonable — very wide tables with many columns may require horizontal scrolling on smaller screens or mobile devices.
- Use headings consistently to support screen readers and Confluence's Table of Contents macro.
- Prefer text over images of text.

### 19.4 Page Deletion Policy

Pages should never be permanently deleted. If content is outdated, archive it (see Section 17.4). If content is incorrect, correct it and note the change. Confluence version history preserves all edits, but deleted pages cannot be easily recovered.

---

## 20. Future Improvements

This standard can evolve over time.

Potential future improvements:

* Create reusable Confluence page templates matching the standard page types (Section 9).
* Create a standard DACI template with pre-filled role sections.
* Create a repository conversion checklist as a standalone Confluence page.
* Define ownership for each converted page and add it to page metadata.
* Add examples of good converted pages and link them from this standard.
* Create a shared index page for all converted repository documentation.
* Align proposal pages with ETO / platform / pipeline management themes.
* Track which proposals have become Epics, Stories, or Tasks (Jira integration).
* Automate staleness detection (e.g. flag pages not updated in 90 days).
* Create a Confluence space structure guide for teams adopting this standard.
* Add a glossary of common terms used across proposals.
* Define a standard for linking Confluence pages back to GitLab source files bidirectionally.
* Evaluate Confluence analytics to measure which pages are actually being read and used.

---

## 21. Summary

This standard provides a consistent way to create, update, and structure Confluence documentation for technical proposals, decision records, KT material, and implementation guides.

The primary workflow is Confluence-first: new documentation is authored directly in Confluence following the standard structures. Existing pages are updated to align over time. A transitional migration process covers the remaining GitLab repositories that still hold proposals in Markdown.

The main goals are:

* Consistent documentation structure across teams.
* Clear separation between decision-level content and technical detail.
* Easier review, feedback, and prioritisation.
* Better collaboration between developers, architects, delivery teams, and wider stakeholders.
* Documentation that stays current through defined ownership and review cadence.

Once the remaining GitLab migrations are complete, the day-to-day workflow is simply: create in Confluence, follow the standard, keep it updated.
