# skills

A public collection of reusable AI skills. Each skill is self-contained in its own directory and can be installed independently.

## Available skills

| Skill | Path | Purpose |
|---|---|---|
| PayPal Integration | `paypal-integration/` | PayPal SDK and REST API integration guidance for checkout, subscriptions, refunds, payouts, webhooks, disputes, migration, and production-readiness review. |

## Install

List available skills:

```bash
npx skills add IdrisMalik/skills --list
```

Install a specific skill from this repository:

```bash
npx skills add IdrisMalik/skills --skill paypal-integration
```

Install directly from the skill path:

```bash
npx skills add https://github.com/IdrisMalik/skills/tree/main/paypal-integration
```

## Repository structure

```txt
.
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── skills.sh.json
└── paypal-integration/
    ├── SKILL.md
    └── references/
        ├── implementation-checklist.md
        ├── legacy-and-migration.md
        ├── rest-apis.md
        └── sdk-v6.md
```

## Skill layout convention

Each skill should use this shape:

```txt
skill-name/
├── SKILL.md
├── references/   # optional supporting guidance
├── scripts/      # optional deterministic helpers
└── assets/       # optional templates or reusable assets
```

Rules:

- Keep each skill self-contained.
- Keep `SKILL.md` concise and focused on behavior, routing, and workflow.
- Put longer domain guidance in `references/`.
- Add scripts only when they are necessary for repeatable or fragile operations.
- Update the table above when adding a new skill.
- Update `skills.sh.json` when adding or renaming a skill.

## Current skill: PayPal Integration

Use `paypal-integration` for PayPal SDK and REST API work involving:

- Checkout and Orders API v2 flows
- Payments API v2 capture, authorization, and refund flows
- Subscriptions, payouts, webhooks, disputes, and other supported REST API v1 product APIs
- JavaScript SDK v6 integration and migration from older SDK patterns
- Legacy PayPal API compatibility review
- Idempotency, reconciliation, and production-readiness audits

## License

MIT
