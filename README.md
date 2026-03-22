# Servcy — One For All Platform

[Servcy](https://servcy.com) is a comprehensive, self-hostable business toolkit combining project management, team collaboration, third-party integrations, billing, and analytics into a single unified platform.

> Servcy is still in its early days. Hiccups may happen — please report bugs and suggestions via [GitHub Issues](https://github.com/Servcy/Server/issues).

The quickest way to get started is with a [Servcy Cloud](https://web.servcy.com) account. For full control, self-host it yourself.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Configuration](#configuration)
  - [Database Setup](#database-setup)
  - [Running the Server](#running-the-server)
  - [Running Celery Workers](#running-celery-workers)
- [API Overview](#api-overview)
- [Integrations](#integrations)
- [Deployment](#deployment)
- [Development](#development)
- [Contributing](#contributing)
- [Security](#security)
- [Community](#community)
- [License](#license)

---

## Features

### Project Management
- Projects with customizable workflow **states**, **labels**, and **priorities**
- **Issues** with assignments, estimates, due dates, and sub-issues
- **Cycles** (sprints/iterations) for time-boxed planning
- **Modules** for grouping related issues
- **Pages** for inline team documentation
- **Custom Views** to filter and save issue queries
- **Time Tracking** with per-issue timers
- **Budget Tracking** per project

### Identity & Access Management
- Email/OTP and Google SSO authentication
- Multi-tenant **Workspace** model with slug-based routing
- Role-based permissions at workspace and project level
- Member invitations with accept/reject flow
- Onboarding and tour-completion tracking

### Integrations
| Provider | Capabilities |
|---|---|
| Google Workspace | Gmail, Drive, Calendar, Meet — via Pub/Sub |
| Microsoft 365 | Outlook, Teams, OneDrive — via MSAL |
| Slack | Messaging, notifications, webhook events |
| GitHub | Issue syncing, webhook handler |
| Asana | Task import/sync |
| Notion | Database sync |
| Figma | Design link embedding |
| Trello | Card sync |
| Jira | Issue mapping |

### Notifications & Communication
- In-app notification centre with user preferences
- Email notifications via **SendGrid** (scheduled every 5 minutes via Celery)
- SMS/phone verification via **Twilio**
- Slack-delivered notifications

### Billing & Payments
- **Paddle** payment integration
- Subscription plan management (Plus plan)
- Webhook-driven payment event handling

### AI Features
- **OpenAI** (GPT-4) integration for AI-assisted workflows
- Token counting via `tiktoken`

### Infrastructure
- Rate limiting: 30 req/min (anonymous), 300 req/min (authenticated)
- Request UUID tracking via custom middleware
- Structured logging with user/request ID filters
- **NewRelic** APM monitoring (production)
- AWS S3 for file storage

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.11.1 |
| Framework | Django 4.2, Django REST Framework 3.14 |
| Authentication | DRF SimpleJWT (3-day access / 7-day refresh tokens) |
| Task Queue | Celery 5.3 + Redis |
| Cache | Redis (django-redis) |
| Database | PostgreSQL (psycopg2) |
| File Storage | AWS S3 (boto3, django-storages) |
| Email | SendGrid |
| SMS | Twilio |
| Payments | Paddle |
| AI | OpenAI (GPT-4) |
| HTTP Server | Gunicorn (WSGI) / Uvicorn (ASGI) |
| Monitoring | NewRelic (production) |

---

## Architecture

```
servcy-be/
├── app/              # Django settings, WSGI/ASGI entry points, Celery config, base models
├── iam/              # Identity & Access Management — users, workspaces, members
├── project/          # Core project management — issues, cycles, modules, pages, states
├── integration/      # Third-party OAuth connections and event handling
├── inbox/            # Email/message inbox and thread management
├── document/         # Document and page storage
├── dashboard/        # Analytics, charts, and data export
├── notification/     # In-app and email notification system
├── billing/          # Subscription and payment management
├── webhook/          # Incoming webhook handlers (GitHub, Slack, Google, Microsoft, Figma, Paddle)
├── common/           # Shared utilities — permissions, exceptions, responses, validators
├── middlewares/      # Custom Django middleware (request UUID tracking)
├── mails/            # SendGrid email service wrapper
├── storage/          # S3 storage backend configuration
├── templates/        # Email HTML templates
├── seeders/          # SQL seed files for initial data
└── config/           # Configuration files (config.ini)
```

---

## Getting Started

### Prerequisites

- Python 3.11.1
- [Poetry](https://python-poetry.org/docs/#installation)
- PostgreSQL
- Redis
- AWS S3 bucket
- (Optional for production) NewRelic account, GCP service account key

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/Servcy/Server.git
cd Server

# 2. Install dependencies
poetry install

# 3. Activate the virtual environment
poetry shell

# 4. Create the logs directory
mkdir logs

# 5. Copy and fill in the configuration file
cp config/config.ini.example config/config.ini
```

### Configuration

Edit `config/config.ini` with your credentials. All sections are required unless noted as optional.

```ini
[main]
secret_key=<django-secret-key>          # Generate with: python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
fernet_key=<encryption-key>             # Generate with: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
host_type=development                   # or: production
frontend_url=http://localhost:3000
backend_url=http://localhost:8000

[database]
host=localhost
name=servcy_db
user=postgres
password=<password>
port=5432

[redis]
url=redis://localhost:6379

[aws]
bucket=<s3-bucket-name>
access=<aws-access-key-id>
secret=<aws-secret-access-key>
region=<aws-region>

[sendgrid]
key=<sendgrid-api-key>
endpoint=https://api.sendgrid.com/v3/mail/send
verification_template_id=<template-id>
workspace_invitation_template_id=<template-id>
analytics_export_template_id=<template-id>
new_signup_template_id=<template-id>
from_email=<sender-email>

[twilio]
account_sid=<account-sid>
auth_token=<auth-token>
verify_service_id=<verify-service-id>
from_number=<twilio-phone-number>

[google]
client_id=<oauth-client-id>
client_secret=<oauth-client-secret>
sso_client_id=<sso-client-id>
sso_client_secret=<sso-client-secret>
project_id=servcy
redirect_uri=<oauth-redirect-uri>
pub_sub_topic=<pubsub-topic-name>
pub_sub_subscription=<pubsub-subscription-name>

[microsoft]
display_name=<app-display-name>
client_id=<azure-client-id>
client_secret_id=<azure-secret-id>
client_secret=<azure-client-secret>
object_id=<azure-object-id>
tenant_id=<azure-tenant-id>
redirect_uri=<oauth-redirect-uri>

[slack]
client_id=<slack-client-id>
client_secret=<slack-client-secret>
app_token=<slack-app-token>
verification_token=<slack-verification-token>
redirect_uri=<oauth-redirect-uri>

[notion]
client_id=<notion-client-id>
client_secret=<notion-client-secret>
redirect_uri=<oauth-redirect-uri>

[figma]
client_id=<figma-client-id>
client_secret=<figma-client-secret>
redirect_uri=<oauth-redirect-uri>

[github]
client_id=<github-client-id>
client_secret=<github-client-secret>
webhook_secret=<github-webhook-secret>
redirect_uri=<oauth-redirect-uri>

[asana]
client_id=<asana-client-id>
client_secret=<asana-client-secret>
redirect_uri=<oauth-redirect-uri>

[trello]
app_key=<trello-app-key>
client_secret=<trello-client-secret>
redirect_uri=<oauth-redirect-uri>

[jira]
client_id=<jira-client-id>
app_id=<jira-app-id>
client_secret=<jira-client-secret>
redirect_uri=<oauth-redirect-uri>

[openai]
api_key=<openai-api-key>
model_id=gpt-4
organization_id=<openai-org-id>
max_tokens=2000

[paddle]
secret_key=<paddle-secret-key>
webhook_secret=<paddle-webhook-secret>
plus_price_id=<paddle-price-id>

[unsplash]
app_id=<unsplash-app-id>
access_key=<unsplash-access-key>
secret_key=<unsplash-secret-key>
```

**Additional setup for production:**
- Place `new_relic.ini` under the `config/` directory
- Place `servcy-gcp-service-account-key.json` under the `config/` directory (required for Google Workspace integrations)

### Database Setup

```bash
# Create the PostgreSQL database
createdb servcy_db

# Run migrations
python manage.py migrate

# Seed integration catalog data
psql -U postgres -d servcy_db -f seeders/integration.sql
psql -U postgres -d servcy_db -f seeders/integration_event.sql
psql -U postgres -d servcy_db -f seeders/widget.sql
```

### Running the Server

```bash
python manage.py runserver
```

The API is available at [http://localhost:8000](http://localhost:8000).

### Running Celery Workers

In a separate terminal (with the Poetry environment active):

```bash
celery -A app worker --loglevel=info
```

Celery handles:
- Email notification delivery (scheduled every 5 minutes)
- Integration event processing
- Dashboard analytics tasks
- Async project task operations

---

## API Overview

### Authentication

All protected endpoints require a JWT Bearer token:

```
Authorization: Bearer <access_token>
```

| Property | Value |
|---|---|
| Access token lifetime | 3 days |
| Refresh token lifetime | 7 days |
| Token rotation | Enabled on refresh |

### Rate Limits

| User Type | Limit |
|---|---|
| Anonymous | 30 requests/minute |
| Authenticated | 300 requests/minute |

### Endpoints

| Prefix | Description |
|---|---|
| `POST /authentication/` | OTP generation, verification, Google SSO login |
| `POST /logout/` | Token invalidation |
| `GET/PATCH /iam/me/` | Current user profile |
| `GET/POST /iam/workspaces/` | List and create workspaces |
| `/iam/{workspace_slug}/members/` | Workspace member management |
| `/iam/{workspace_slug}/invitations/` | Invite and manage members |
| `/project/{workspace_slug}/projects/` | Project CRUD |
| `/project/{workspace_slug}/issues/` | Issue management |
| `/project/{workspace_slug}/cycles/` | Sprints and cycles |
| `/project/{workspace_slug}/modules/` | Module grouping |
| `/project/{workspace_slug}/pages/` | Documentation pages |
| `/project/{workspace_slug}/states/` | Workflow states |
| `/project/{workspace_slug}/views/` | Custom issue views |
| `/project/{workspace_slug}/timers/` | Time tracking |
| `/project/{workspace_slug}/estimates/` | Effort estimates |
| `/project/{workspace_slug}/budgets/` | Project budgets |
| `/project/search/` | Global search |
| `/integration/list/` | Available integrations |
| `/integration/connect/` | OAuth connection flow |
| `/integration/user-integrations/` | User's connected integrations |
| `/inbox/mails/` | Email inbox |
| `/inbox/threads/` | Conversation threads |
| `/document/documents/` | Document management |
| `/dashboard/analytics/` | Analytics data |
| `/dashboard/export/` | Data export |
| `/notification/notifications/` | User notifications |
| `/notification/preferences/` | Notification settings |
| `/billing/subscription/` | Subscription management |
| `/webhook/github/` | GitHub webhook receiver |
| `/webhook/slack/` | Slack webhook receiver |
| `/webhook/google/` | Google Pub/Sub webhook |
| `/webhook/microsoft/` | Microsoft webhook receiver |
| `/webhook/figma/` | Figma webhook receiver |
| `/webhook/paddle/` | Paddle payment webhook |

---

## Integrations

OAuth connections are initiated via `/integration/connect/` and handled by provider-specific callback views. Each integration stores encrypted credentials (via Fernet) in the `UserIntegration` model.

Webhook events from external services are received at `/webhook/<provider>/` and dispatched to the corresponding handler in `webhook/`.

To add a new integration, refer to the `integration/services/` directory for existing patterns and open an issue or PR with your proposal.

---

## Deployment

The project ships with a GitHub Actions workflow (`.github/workflows/deploy.yml`) that deploys on push to `main`:

1. SSH into the production server
2. `git pull` latest code
3. `poetry install` — install/update dependencies
4. `python manage.py migrate` — apply database migrations
5. Restart Gunicorn, Nginx, and Celery worker systemd services

**Production stack:**
- **Gunicorn** — WSGI HTTP server
- **Nginx** — Reverse proxy / TLS termination
- **Celery** — Background task worker
- **Redis** — Celery broker and cache backend
- **PostgreSQL** — Primary database
- **AWS S3** — File storage
- **NewRelic** — APM monitoring

---

## Development

### Code Style

This project follows the [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html) with automated formatting enforced via pre-commit hooks.

```bash
# Format code
black .
isort .

# Type checking
mypy .

# Lint
pylint .

# Run tests
pytest

# Run tests with coverage
pytest --cov
```

### Pre-commit Hooks

```bash
# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

Hooks include:
- **Black** — code formatting
- **GitGuardian (ggshield)** — secret detection

### Project Conventions

- Docstrings follow the **NumPy/SciPy** style
- All public modules, classes, functions, and methods must be documented
- Repository layer handles data access; service layer holds business logic
- Custom exceptions are defined in `common/exceptions.py`
- Standardised API responses via `common/responses.py`

---

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before submitting a pull request.

**Quick summary:**
1. Search existing [issues](https://github.com/Servcy/Server/issues) before opening a new one
2. For new features, submit a proposal issue first
3. Follow the coding guidelines (Google Python Style Guide + Black + isort)
4. Include a minimal reproduction for bug reports
5. Open a PR using the provided [pull request template](.github/PULL_REQUEST_TEMPLATE.md)

Our [Code of Conduct](./CODE_OF_CONDUCT.md) applies to all community spaces.

---

## Security

If you discover a security vulnerability, **do not open a public issue**. Email [contact@servcy.com](mailto:contact@servcy.com) with details. We aim to respond within 3 business days and will coordinate responsible disclosure.

See [SECURITY.md](./SECURITY.md) for the full policy including scope and out-of-scope items.

---

## Community

- [GitHub Discussions](https://github.com/Servcy/Server/discussions) — questions, ideas, and project showcase
- [GitHub Issues](https://github.com/Servcy/Server/issues) — bug reports and feature requests

---

## License

See [LICENSE.txt](./LICENSE.txt) for details.
