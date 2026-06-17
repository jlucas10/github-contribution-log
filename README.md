# Contribution #12167: Codeberg OAuth does not allow secondary e-mails to be used

**Contribution Number:** 1  
**Student:** Josiah Lucas  
**Issue:** https://github.com/WeblateOrg/weblate/issues/12167  
**Status:** Phase II Complete

---

## Why I Chose This Issue

I selected this issue for my first open source contribution because it offers a highly structured, clear introduction to working within a production-grade backend codebase. The core project maintainer has explicitly mapped out the architectural strategy for the fix. A functional blueprint already exists within the repository showing how GitHub handles fetching secondary emails via its API pipeline, which allows me to mirror this established pattern for Codeberg.

---

## Understanding the Issue

### Problem Description

When users authenticate with Weblate using Codeberg OAuth, the platform's current pipeline only captures and records the user's primary email address. This creates a friction point for developers who utilize distinct secondary email addresses for their Git commit identity on platforms like Hosted Weblate, as those secondary addresses are completely missing from their Weblate account settings. Codeberg is a non-profit, privacy-focused open-source hosting platform for Git repositories built entirely on Gitea, meaning its integration behaves through Weblate's Gitea authentication flows.

### Expected Behavior

When a user logs in via Codeberg OAuth, Weblate should interact with the underlying provider's API (`/user/emails`) to fetch all verified secondary email addresses associated with that account and cleanly make them available inside the user's Weblate account settings.

### Current Behavior

Only the main/primary email address is parsed and linked during the OAuth login handshake flow. Any secondary email addresses configured on the Codeberg profile are ignored and completely dropped from the account setup.

### Affected Components

- `weblate/weblate/accounts/pipeline.py` (Specifically the custom pipeline function `require_email`)
- Python Social Auth backend configuration hooks (`social_core.backends.gitea`)

---

## Reproduction Process

### Environment Setup

- **OS:** macOS (Apple Silicon M-Series)
- **Setup Process:** Cloned the repository using a shallow clone (`--depth 1`) to bypass over 15 years of historical commit bloat (86,000+ commits). Utilized the top-level `./rundev.sh` automation script provided by the project to auto-orchestrate the local developer Docker environment variables, verify host parameters, and build the required containers.
- **Challenges & Fixes:** Overcame an initial `Docker daemon` connection failure by explicitly launching the native Docker Desktop application, ensuring the background engine was fully active and running before executing the terminal commands.

### Steps to Reproduce

1. Configure a user profile on Codeberg containing a primary email and at least one verified secondary email address.
2. Initialize and run the local Weblate instance inside Docker via `./rundev.sh`.
3. Attempt a registration or login handshake utilizing the Codeberg OAuth provider option.
4. Navigate to the local account user profile dashboard (`/accounts/settings/`).
5. **Observed Result:** Only the primary email address is appended to the account profile data. The secondary email addresses are omitted completely.

### Reproduction Evidence

- **Branch Link:** https://github.com/JosiahLucas/weblate/tree/fix-issue-12167
- **My findings:** The reproduction confirms that the authentication mapping cycle cuts off short after capturing general parameters. Without an intermediate backend function capturing the array payload from the Codeberg/Gitea email list endpoint, secondary email addresses never make it to the state dictionary.

---

## Solution Approach

### Analysis

The root cause is located inside `weblate/weblate/accounts/pipeline.py` inside the `require_email` step hook. Weblate uses a custom authentication pipeline to clear social registrations. While there is a conditional check to catch `backend.name == "github"` and execute a helper method to extract secondary email arrays from the authentication response payload, no such interceptor exists for the `gitea` backend (which powers Codeberg).

### Proposed Solution

Implement a parallel pipeline parser step (`parse_codeberg_emails` or `parse_gitea_emails`) inside the authentication logic that targets the Gitea API email payload, loops through verified secondary records, and appends them cleanly to `details["verified_emails"]` so Weblate updates the database upon account association.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Weblate completely ignores secondary email arrays during Codeberg/Gitea OAuth handshakes, blocking users from signing commits with their preferred secondary Git identity emails.

**Match:** I will match the implementation pattern of the `parse_github_emails` function and its corresponding conditional invocation check inside `require_email` on lines 125-130 of `weblate/weblate/accounts/pipeline.py`.

**Plan:**
1. Verify the exact JSON response scheme for Gitea’s `/user/emails` endpoint (checking keys for `email`, `verified`, and `primary`).
2. Implement `parse_codeberg_emails(data: Iterable[GitHubEmailData])` right below the GitHub parser function in `pipeline.py`.
3. Add an entry filter to skip unverified emails and cleanly strip out the `@noreply.codeberg.org` system email variants.
4. Add an explicit `if backend.name == "gitea"` block inside the `require_email` pipeline function to trigger the new parser.
5. Identify the test suite file for user registration pipelines (typically `weblate/accounts/tests/test_pipeline.py` or similar) and implement a mock pipeline test mirroring the current GitHub pipeline testing patterns.

**Implement:** [In Progress - Branch Created: https://github.com/JosiahLucas/weblate/tree/fix-issue-12167]

**Review:** Code changes will be self-reviewed against Weblate's `CONTRIBUTING.md` standards, proper clean commit message titles, and strict data-type compliance matching `VerifiedEmailList`.

**Evaluate:** The fix will be confirmed operational when a new custom backend pipeline test case passes successfully using mocked HTTP payloads representing a Gitea email array structure.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Verify `parse_codeberg_emails` properly filters out unverified email structures.
- [ ] Test case 2: Verify primary email remains safely prioritized during user detail setup hooks.
- [ ] Test case 3: Verify system addresses like `@noreply.codeberg.org` are caught and flagged as non-deliverable.

### Integration Tests

- [ ] Integration scenario 1: Simulation of the user pipeline handshake with a mocked external Gitea response dictionary containing multiple secondary emails.

### Manual Testing

[In Progress]

---

## Implementation Notes

### Week 1 Progress

Selected the tracking issue, analyzed the maintainer's architectural suggestions, verified the underlying python files, cloned the codebase via a shallow fork setup, successfully spun up the environment via Docker, and finalized the Phase II target architecture blueprint.

---

## Pull Request

**PR Link:** [In Progress]

**PR Description:** [In Progress]

**Maintainer Feedback:**
- [In Progress]

**Status:** In Progress

---

## Learnings & Reflections

### Technical Skills Gained

[In Progress]

### Challenges Overcome

[In Progress]

### What I'd Do Differently Next Time

[In Progress]

---

## Resources Used

- **Weblate Repository Main Track:** https://github.com/WeblateOrg/weblate
- **GitHub Reference Target Block:** `weblate/weblate/accounts/pipeline.py`
- **Gitea API Developer Guidelines:** https://docs.gitea.com/
