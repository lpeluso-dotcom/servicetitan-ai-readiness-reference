# Security and Sensitive Data

This project should never contain live credentials, tenant exports, customer data, staff data, call recordings, phone numbers, private screenshots, or internal system URLs.

If sensitive data is found:

1. Remove it from the repository immediately.
2. Rotate any exposed credential or token.
3. If the repository is public, treat the value as compromised even if it was visible briefly.
4. Replace the sensitive detail with a placeholder such as `{tenantId}`, `{customerId}`, `{trackingNumber}`, or `{oauth2_access_token}`.

Please do not open a public issue that repeats the sensitive value.

