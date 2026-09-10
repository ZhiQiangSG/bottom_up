---
domain: discovery
requires_specs:
  - ARCHITECTURE.md
  - domains/discovery/domain.md#search-contract
  - domains/identity-access/domain.md#rbac-roles
---

# 04 — Search and Filter Job Listings

| ID | 04 |
| --- | --- |
| Name | Search and Filter Job Listings |
| Description | Student searches and filters available internship and entry-level job listings. |

## Goal

Let a logged-in student query active listings by text plus multi-select filters and sorting, with the URL carrying the active parameters so results are shareable.

## Non-goals

- "Save Search" toggle and alert preferences (optional scope, Carousell-style) — deferred.
- Scraper / employer ingestion pipeline that populates listings.
- Personalised ranking or recommendations.

## Actors

- Student

## Preconditions

- Student is logged in.
- Active job listings exist in the database (via scraper or employer postings).

## Postconditions

- Matching, active listings are displayed with key details (title, company, location, deadline).
- Page URL carries query parameters matching the active filters (shareable results).

## Gherkin scenarios

```gherkin
Feature: Search and Filter Job Listings
  Scenario: Default listing is newest first
    Given active listings exist
    When the student navigates to the job search page
    Then listings are displayed newest first with title, company, location, and deadline

  Scenario: Filter by work type
    Given published jobs exist
    When GET /api/jobs/?work_type=internship (proposed)
    Then only internships are returned

  Scenario: Text search plus filters narrows results
    Given active listings exist across roles, locations, and industries
    When the student enters a search query and applies role, location, industry, and salary filters
    Then only matching listings are returned
    And the page URL updates to include the active query parameters

  Scenario: Sort control reorders results
    Given filtered results are shown
    When the student selects "Highest Salary" or "Nearest Deadline"
    Then results reorder accordingly and the URL reflects the sort parameter

  Scenario: Zero matches show empty state
    Given no listings match the active criteria
    When the student applies the restrictive filters
    Then an empty state is shown suggesting broader criteria or removing filters

  Scenario: Malicious or oversized input is sanitised
    Given the student is on the search page
    When the student submits special characters or an excessively long query string
    Then the query is validated/sanitised and the database does not error
```

## Acceptance criteria

- [ ] Search bar plus multi-select filters (Job Role, Location, Working Arrangement Remote/Hybrid/Onsite, Industry, Salary Range, Duration, Timeframe) and sort dropdown (Newest First, Highest Salary, Nearest Deadline) query only active listings.
- [ ] Default sort is newest first; listing cards show title, company, location, deadline; clicking a listing opens full details.
- [ ] Active query + filters + sort are reflected in shareable URL parameters.
- [ ] Zero matches render the empty state with broaden-search guidance; invalid input is sanitised without query crash.
- [ ] OpenAPI schema exposes the search endpoint (`GET /api/schema/`) once implemented.

## Open questions

- Final router path and query-param names for `GET /api/jobs/` (proposed) — confirm in implementation plan.
- Salary-range bucket definitions and deadline semantics for internships.
