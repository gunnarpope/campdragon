# 🐉 CampDragon

**Sign up for local summer camps — in a few clicks.**

CampDragon is a full-service, web-based platform that connects summer camp hosts with families. Hosts advertise their camp offerings; families register once, then click-to-subscribe to any camp they choose. Built to launch locally in Hanover, NH and designed to scale across the USA.

---

## Features

### For Families
- **One-time family profile**: Enter demographics, ages, interests, preferred activities, and location once
- **Saved payment method**: Store your payment info securely via Stripe — no re-entry per camp
- **Click-to-subscribe**: Browse available camps and sign up with a single click
- **Flexible signup models** (see below)
- **Multi-year enrollment**: Lock in future camp seasons for returning families

### For Camp Hosts
- **Camp listings**: Post events with descriptions, dates, capacity, age ranges, activities, and pricing
- **Host dashboard**: Manage enrollment, view rosters, and communicate with families
- **Verified host accounts**: Authenticated via the CampDragon host onboarding process
- **Signup window control**: Configure time-based registration opens with burst-safe queuing

### Platform
- **Serverless & scalable**: AWS Lambda + API Gateway handles burst traffic during signup windows
- **Multi-region ready**: Designed to start locally and replicate to new markets
- **Legal templates**: Built-in Terms of Service, Privacy Policy, host agreements, and liability waivers

---

## Signup Models

| Model | Description |
|-------|-------------|
| **FIFO** | First-come, first-served. Registrations processed in strict timestamp order using SQS FIFO. |
| **Patron Priority** | Supporters/donors receive an early-access window before general registration opens. |
| **Multi-Year** | Returning families can re-enroll for future seasons at registration time. |
| **Time-Based Window** | Camp hosts set a specific open datetime; burst traffic is absorbed by the signup queue. |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | [Python Dash](https://dash.plotly.com/) + Dash Bootstrap Components |
| Auth | AWS Cognito (families, hosts, admins) |
| Compute | AWS Lambda + API Gateway (via `apig-wsgi`) |
| User Metadata | AWS DynamoDB (Global Tables for multi-region) |
| Media & Assets | AWS S3 + CloudFront |
| Signup Queues | AWS SQS FIFO |
| Payments | Stripe (Checkout + EventBridge webhooks) |
| IaC | OpenTofu |
| CI/CD | GitHub Actions |

---

## Architecture

```
Families / Hosts
      │
      ▼
 CloudFront (CDN + static assets from S3)
      │
      ▼
 API Gateway
      │
      ▼
 Lambda (Dash App — apig-wsgi bridge)
   │        │        │        │
   ▼        ▼        ▼        ▼
Cognito  DynamoDB   S3    SQS FIFO
(auth)  (profiles, (media, (signup
        camps,     legal)   queue)
        queues)
              │
              ▼
         Stripe EventBridge
         (payment webhooks)
```

### DynamoDB Tables

| Table | Key Design | Purpose |
|-------|------------|---------|
| `families` | `pk=family_id` | Family profile, interests, ages, Stripe customer ID |
| `camps` | `pk=camp_id`, `sk=host_id` | Camp listings with metadata, capacity, dates |
| `registrations` | `pk=family_id`, `sk=camp_id#timestamp` | Enrollment records and status |
| `signup_queues` | `pk=camp_id`, `sk=enqueued_at` | FIFO queue state per camp |
| `legal_consents` | `pk=family_id`, `sk=consent_type#timestamp` | COPPA/waiver acknowledgments |

---

## Project Structure

```
campdragon/
├── app/
│   ├── app.py                  # Dash app factory
│   ├── lambda_handler.py       # AWS Lambda entry point (apig-wsgi)
│   ├── pages/
│   │   ├── home.py             # Landing & camp discovery
│   │   ├── profile.py          # Family profile setup
│   │   ├── signup.py           # Camp signup flow
│   │   └── host/
│   │       ├── dashboard.py    # Host management dashboard
│   │       └── listing.py      # Create/edit camp listings
│   ├── components/
│   │   ├── auth.py             # Cognito auth helpers
│   │   ├── camp_card.py        # Reusable camp display card
│   │   └── queue_status.py     # Live queue position indicator
│   └── services/
│       ├── dynamodb.py         # DynamoDB CRUD operations
│       ├── s3.py               # S3 media upload/retrieval
│       ├── cognito.py          # User pool management
│       ├── stripe_service.py   # Payment intents & webhooks
│       └── queue_service.py    # SQS FIFO signup queue logic
├── infra/
│   ├── modules/
│   │   ├── dash_app/           # Lambda + API Gateway
│   │   ├── auth/               # Cognito user pools & groups
│   │   ├── storage/            # S3 buckets + CloudFront
│   │   ├── database/           # DynamoDB tables + GSIs
│   │   ├── payments/           # Stripe EventBridge integration
│   │   └── queues/             # SQS FIFO queues
│   └── environments/
│       ├── dev/                # Local/dev deployment
│       └── prod/               # Production (per region)
├── legal/
│   └── templates/
│       ├── terms_of_service.md
│       ├── privacy_policy.md   # COPPA-compliant
│       ├── host_agreement.md   # Camp host verification & liability
│       └── liability_waiver.md # Per-enrollment waiver
├── tests/
│   ├── unit/
│   └── integration/
├── requirements.txt
├── requirements-dev.txt
└── README.md
```

---

## Local Development

### Prerequisites
- Python 3.12+
- AWS CLI configured (`aws configure`)
- Stripe CLI (for local webhook testing)

### Setup

```bash
# Clone and install
git clone https://github.com/your-org/campdragon.git
cd campdragon
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt

# Run locally
python app/app.py
# Open http://localhost:8050
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `COGNITO_USER_POOL_ID` | Cognito user pool ID |
| `COGNITO_CLIENT_ID` | Cognito app client ID |
| `DYNAMODB_TABLE_PREFIX` | Table name prefix (e.g. `campdragon-dev`) |
| `S3_ASSETS_BUCKET` | S3 bucket for camp media |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret (SSM path) |
| `APP_ENV` | `dev` / `prod` |

> **Note:** Sensitive values (Stripe keys, Cognito secrets) are stored in AWS SSM Parameter Store and never hardcoded.

---

## Multi-Region Expansion

CampDragon is designed to launch locally and replicate regionally:

1. **Phase 1 — Hanover, NH** (`us-east-1`): Single-region deployment, local camp market
2. **Phase 2 — New England**: DynamoDB Global Tables enabled, Route53 geolocation routing
3. **Phase 3 — National**: Per-region Lambda stacks, CloudFront global distribution, multi-account DynamoDB replication

Each new market is onboarded by deploying the OpenTofu `prod` environment module to the target region — data models and service interfaces remain identical.

---

## Legal & Compliance

All user-facing legal documents are provided as Markdown templates in `legal/templates/` and are versioned alongside the codebase. Key protections:

- **Terms of Service** — platform usage rules, host/family responsibilities, refund policy
- **Privacy Policy** — COPPA-compliant data handling for minors under 13; parental consent required
- **Host Agreement** — camp host identity verification, insurance requirements, conduct standards
- **Liability Waiver** — per-enrollment electronic signature collected at signup; stored in DynamoDB with timestamp

> **Disclaimer:** Legal templates are starting points. Consult a licensed attorney before operating in production.

---

## Contributing

1. Fork the repo and create a feature branch
2. Run tests: `pytest tests/`
3. Submit a PR — all changes require passing CI and at least one review

---

## License

MIT — see `LICENSE` for details.
