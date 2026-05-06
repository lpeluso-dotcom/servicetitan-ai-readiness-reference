# Contributing

This repository is a dense technical reference corpus, not a blog. Contributions should make the material more accurate, more useful for operators, and easier for language models to retrieve.

## Contribution standards

- Keep each file self-contained. Expand acronyms on first use inside the section you touch.
- Preserve numbered headings and stable section anchors when possible.
- Use placeholders for tenant IDs, account IDs, phone numbers, credentials, screenshots, and customer data.
- Do not paste live tenant exports, secrets, private URLs, customer records, staff records, or call data.
- Do not copy official ServiceTitan documentation. Summarize observed behavior, link to official docs where helpful, and note what was validated directly.
- Prefer tables for field references, error catalogs, and repeatable procedures.
- Mark tenant-specific examples as illustrative unless the pattern is known to generalize.

## Suggested pull request format

```md
## What changed

## How it was validated

## Sensitive data check

## Related file/section
```

