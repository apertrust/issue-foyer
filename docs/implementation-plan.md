# IssueFoyer implementation plan

## First release goal

A person without a GitHub account can submit a bug report or feature suggestion through a project-branded page. IssueFoyer creates a GitHub issue and returns its link. GitHub remains the system of record; IssueFoyer stores no report archive.

## Proposed implementation choices

- **Frontend:** small TypeScript application built as static files with Vite. Keep the UI framework-free unless the form needs prove otherwise.
- **Submission service:** TypeScript Cloudflare Worker, with an API contract that can be implemented on another serverless host.
- **Configuration:** versioned YAML validated at build time and emitted as public JSON. Form fields, categories, labels, and branding are declarative.
- **GitHub access:** GitHub App installation token, scoped to the configured repository and required issue permissions. Secrets exist only in Worker secret storage.
- **Identity and data:** no reporter login, no contact collection, no database. The submitted issue is authored by the GitHub App.
- **Issue board:** off by default. If enabled later, serve public issue data through a cached read path.

These are implementation defaults. Keep the first vertical slice small and revisit a choice only when implementation evidence requires it.

## Build sequence

### 1. Repository skeleton and configuration

Create the static frontend, Worker, shared configuration schema, local development commands, and deployment examples. Include a sample project configuration with bug and feature categories. Reject unknown schema versions and unsafe configuration such as executable markup.

**Exit when:** a clean checkout can build the static page and run the Worker locally with a sample configuration and no credentials.

### 2. Report form

Build the responsive form from configuration. Include category-specific questions, required-field validation, a clear notice that reports become public GitHub issues, and accessible error and success states. The success state initially uses a local mock submission response.

**Exit when:** the full reporter journey works against the mock provider on desktop and mobile, including validation and failure states.

### 3. GitHub issue creation

Implement the GitHub provider in the Worker. Validate every field on the server, derive labels and repository only from trusted configuration, format a safe Markdown issue body, obtain a short-lived installation token, and create the issue. Return the issue URL and number. Keep request IDs in the issue body for best-effort duplicate detection; do not claim exactly-once delivery.

**Exit when:** a report submitted to a dedicated sandbox repository appears as a well-formatted issue and the reporter receives its link. Errors, token expiry, unavailable GitHub responses, and rate limiting return safe messages without exposing secrets.

### 4. Public endpoint hardening

Add origin allowlisting, request-size limits, per-project and per-IP throttling, a configurable production challenge, secret-safe logging, and backoff for GitHub `403` and `429` responses. Never retry issue creation aggressively. Do not put report bodies, contact details, or tokens in routine logs.

**Exit when:** automated traffic is constrained before reaching GitHub, rate-limit responses are handled without retry loops, and the service retains no report payload after the request ends.

### 5. Deployment and adoption

Document GitHub App setup, required permissions, static-site deployment to GitHub Pages, Worker deployment and secrets, custom branding, and local development. Provide a reusable project template and a setup checklist. Validate the instructions by deploying the IssueFoyer portal against its sandbox repository, then configure Apertrust as the first adopter.

**Exit when:** a maintainer can deploy the portal from a fresh copy without editing application code, and the Apertrust configuration can be expressed using project configuration alone.

### 6. Optional public issue board

After submissions are reliable, add a project option to show all or open issues in configured categories. Fetch through a cached read path; do not call GitHub separately from every visitor's browser. Keep the board disabled by default and do not add account-based “my reports” in this release.

**Exit when:** an enabled board displays current public issue details without persisting a separate issue database or making each browser consume GitHub's unauthenticated API quota.

## Out of scope for the first release

- Reporter accounts or Google/Facebook sign-in.
- Email notifications or private follow-up. These require verified contact details and a retained contact-to-issue mapping.
- File uploads, screenshots, a moderation dashboard, webhook-driven sync, comments, and additional issue providers.

## Maintainer setup needed for the GitHub milestone

The GitHub integration needs a GitHub App registration, installation on the sandbox repository, and its private key stored as a Worker secret. Keep those credentials outside the repository and never put them in GitHub Pages settings or frontend configuration. The App should receive only the repository access and issue permissions required by the implementation.
