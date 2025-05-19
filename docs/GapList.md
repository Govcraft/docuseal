# Gap List

1. **User roles beyond “admin” appear disabled.**
   - Views list “editor” and “viewer” options but they’re disabled and no logic exists to manage these roles. Clarify whether future features will support them or remove unused options.

2. **Environment variables lack a consolidated reference.**
   - Configuration files pull many variables (`DATABASE_URL`, `REDIS_URL`, `AWS_SECRET_MANAGER_ID`, etc.). Documenting all required variables and defaults would aid deployment.

3. **No explicit mention of high-availability or clustering strategy.**
   - Sidekiq and Redis are embedded in-process for single-node installs. Multi-node or fail-over design is not described.

4. **Security policy states some vulnerabilities are out of scope (CSRF, etc.).**
   - The codebase includes forms that might be subject to CSRF if misconfigured. Clarification from product owners may be needed.

5. **Some Pro features (SMS verification, conditional fields, advanced user roles) are referenced in UI texts but missing implementation details in the public repo.**
   - Determine which features are intended for the open-source release versus proprietary extensions.

6. **Internationalization coverage is broader than README implies.**
   - README claims 6 UI languages and 14 signing languages, while `i18n.yml` lists ~20 languages. Verify supported locales and update docs.

7. **Versioning mechanism is unclear.**
   - The `.version` file is empty. Determine if version numbers should be embedded for reproducible deployments.
