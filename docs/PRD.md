# Product Requirements Document (Reverse-Engineered)

## 1. Functional Overview

DocuSeal is an open-source platform that lets users create fillable PDF forms, send them for completion, and collect electronically signed documents. Features are summarized in the README with additional “Pro” capabilities listed separately.

### 1.1 User Roles and Authentication
- Only an “admin” role is implemented in the OSS version; additional roles (“editor”, “viewer”) are placeholders and appear disabled in the UI.
- Two-factor authentication (OTP) is integrated via Devise; users may be required to enter an OTP code after login.
- OAuth logins for Google and Microsoft are available if configured.
- SAML SSO settings can be configured in the admin area.
- API authentication uses access tokens hashed and stored in `access_tokens` table, supplied in requests via `X-Auth-Token` header.

### 1.2 Account and User Management
- Each account has multiple users, templates, submissions and related records.
- Accounts can be linked to other accounts as “testing” or “integration” accounts via `account_linked_accounts` tables.
- Account configurations govern behaviors such as forced MFA, email templates, signing preferences, etc.
- Users can create/manage other users from the settings UI. System tests show creation, editing, archiving and restoration of users.

### 1.3 Template Management
- Templates contain form fields, schema info and a list of submitters; attachments (PDFs/DOCX) are stored via ActiveStorage.
- Templates may be cloned, moved between folders and archived via UI actions and API endpoints. System specs demonstrate these flows.
- API endpoints for templates provide CRUD and cloning functionality.

### 1.4 Submission Lifecycle
- Submissions are created from a template and hold per-signer data, including field states and attachments. Expiry and order of submitters are recorded.
- Public `start_form` and `submit_form` endpoints allow submitters to start or continue filling a form via unique slugs.
- Upon completion of a submitter’s portion, a background job `ProcessSubmitterCompletionJob` generates signed documents, combined PDFs and audit trails, and sends emails or next-signature invitations.
- When a submission’s `expire_at` is set, `ProcessSubmissionExpiredJob` is scheduled for that time to trigger expiration webhooks.
- API endpoints expose submission CRUD operations and retrieval of associated documents or events.

### 1.5 Submitter Features
- Submitters represent individual signers with their own values, preferences, attachments and status transitions (sent/opened/completed/declined).
- Submitters can upload attachments, draw signatures, or use stored signatures via the form UI (demonstrated extensively in system specs).
- API endpoints allow retrieving and updating submitters, including marking them completed via the API.

### 1.6 Webhooks and Events
- Webhook URLs can be configured per account with selectable event types (`form.completed`, `submission.created`, etc.), optional secret headers, and a testing feature to trigger a sample call.
- Outgoing requests are enforced to use HTTPS and reject localhost destinations when in multitenant mode.
- All webhook jobs retry up to 10 times with exponential backoff if responses are 4xx/5xx errors.

### 1.7 Storage and File Handling
- ActiveStorage is configured to support local disk, AWS S3, Google Cloud Storage or Azure Storage depending on environment variables.
- File downloads are proxied through custom controllers for access control, with optional pre-signed URLs using a configurable expiration time.

### 1.8 Tools API
- `/api/tools/merge` merges multiple PDF files (provided as base64) into a single PDF and returns the merged data as base64.
- `/api/tools/verify` verifies PDF signatures, checking both cryptographic signatures and whether the file matches a previously stored checksum.

### 1.9 Admin Settings
Routes under `/settings/...` provide pages for:
- Email SMTP configuration
- Storage backends (disk, S3, GCP, Azure)
- Webhook configuration
- SSO/SAML configuration
- User, API and notifications settings
- Personalization and e-sign settings (account-specific preferences)

### 1.10 Observability
- Lograge formats logs to JSON with additional request context (user IDs, account IDs, resource IDs, etc.) in production.
- Sidekiq is the background job processor; a Puma plugin starts it in-process when not running in multitenant mode, and a local Redis server is spawned if `LOCAL_REDIS_URL` is set.

### 1.11 Deployment and Configuration
- Docker images can run with `docker run -p 3000:3000 -v.:/data docuseal/docuseal` and default to SQLite; `DATABASE_URL` switches to PostgreSQL or MySQL.
- Docker Compose example uses the `HOST` environment variable to set the domain, expecting Caddy for HTTPS certificates.
- In production, environment variables control storage backends, SMTP settings, encryption secret, etc.
- On boot in production, migrations automatically run unless `RUN_MIGRATIONS=false`.
- If `AWS_SECRET_MANAGER_ID` is set, secrets are pulled from AWS Secrets Manager, otherwise `.env` is auto-generated in `docuseal.env` for local setup with fallback defaults.

### 1.12 Security Considerations
- Parameters such as passwords, tokens and OTP codes are filtered from logs.
- ActiveRecord encryption keys are derived from `SECRET_KEY_BASE` or an `ENCRYPTION_SECRET` environment variable.
- Webhook destinations must be HTTPS and not point to localhost unless a specific account configuration (`allow_http`) is present.
- The security policy invites vulnerability reports to `security@docuseal.com` and outlines excluded issues.

## 2. Non-Functional Expectations

### 2.1 Performance and Scalability
- Puma thread settings default to 15 max threads with worker processes determined by `WEB_CONCURRENCY_AUTO` or `WEB_CONCURRENCY`.
- Redis and Sidekiq are embedded in the same process for single-host setups unless `MULTITENANT=true`.
- Webhook jobs retry up to ten times with exponential delays on failures.
- `ProcessSubmissionExpiredJob` is scheduled at each submission’s `expire_at` timestamp.

### 2.2 Security and Compliance
- All account data may be stored on disk or selected cloud storage. ActiveRecord encryption for certain attributes is configured.
- HTTPS is assumed in production and `assume_ssl` ensures correct protocol detection behind reverse proxies.
- Webhook destinations must not be local addresses unless explicitly overridden.
- Two-factor authentication and optional forced MFA per account.

### 2.3 Observability
- Log output is JSON-formatted with selected fields.
- Additional custom payload includes request parameters and user/account identifiers.

### 2.4 Availability
- No explicit SLOs or replication strategies found. Single-node deployments rely on Sidekiq and embedded Redis.

### 2.5 Limits
- Maximum attachment size for outbound emails is 10 MB.
- API pagination defaults to 10 items, capped at 100 per request.

## 3. Data Flows and Edge Cases

- Submissions inherit default preferences from templates and account configs. On creation, submitters can be marked as completed immediately, triggering completion jobs and webhooks.
- Declined or expired submissions do not trigger combined document generation or subsequent invitations.
- Template cloning duplicates fields and attachments but updates submitter UUID references accordingly.
- Webhook events include `form.viewed`, `form.completed`, `form.declined`, `submission.created`, `submission.completed`, `submission.archived`, `submission.expired`, `template.created`, `template.updated`.
- `SendWebhookRequest` enforces a read and open timeout of 8 seconds each.

## 4. Hidden Coupling and Conventions

- Environment variables determine behavior extensively: storage backend, Redis URL, database connection, SMTP settings, host names, etc.
- The open source version uses a single user role but UI templates hint at additional roles reserved for Pro.
- `ProcessSubmissionExpiredJob` scheduling depends on `expire_at` being set at creation time.
- Local Redis and Sidekiq only start if `LOCAL_REDIS_URL` is set; otherwise an external Redis and Sidekiq service is expected.

