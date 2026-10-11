# Vikmo DealerDesk

Vikmo DealerDesk is a Django REST API for automotive parts dealer operations. It supports product catalogue management, inventory updates, dealer management, draft orders, an LLM-powered assistant, and demand forecasting with reorder suggestions.

## Technology Stack

- **Backend:** Python, Django, Django REST Framework
- **Database:** PostgreSQL
- **LLM provider:** Groq
- **Forecasting:** Python, CSV-based demand forecasts, and historical sales data
- **Configuration:** Environment variables loaded from `.env`

## Features

- Browse, search, filter, create, and update products.
- Manage dealer records.
- Update inventory quantities with a reason.
- Create and retrieve draft orders using current catalogue prices.
- Use the AI assistant to search parts, check stock, and create draft orders.
- Retrieve per-SKU forecasts and reorder suggestions based on lead times, historical sales variability, and current stock.
- Seed product and dealer data repeatedly without creating duplicates.

## Requirements

- Python 3.11 or later
- PostgreSQL database
- Groq API key for live assistant requests

## Setup

Clone or copy the project into a local directory and enter the project root.

Create and activate a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate
```

On Windows, activate it with `.venv\Scripts\activate`.

Install dependencies:

```bash
pip install -r requirements.txt
```

Create your local environment file:

```bash
cp .env.example .env
```

Edit `.env` and configure:

- `DATABASE_URL` — PostgreSQL connection URL
- `GROQ_API_KEY` — Groq API key for the assistant
- `DJANGO_SECRET_KEY` — a private secret key
- Supplier settings if the supplier integration is used

Never commit `.env` or real credentials.

## Database Setup

Run migrations and seed the initial catalogue and dealer data:

```bash
python manage.py migrate --noinput
python manage.py seed_data
```

Run the seed command a second time to verify idempotency:

```bash
python manage.py seed_data
```

## Run the Server

```bash
python manage.py runserver 0.0.0.0:8000
```

Health check:

```bash
curl http://localhost:8000/api/health/
```

Expected response:

```json
{"status":"ok"}
```

## API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/health/` | Health check |
| GET, POST | `/api/products/` | List or create products |
| GET, PATCH | `/api/products/<sku>/` | Retrieve or update a product |
| PUT | `/api/inventory/<sku>/` | Update inventory |
| GET, POST | `/api/dealers/` | List or create dealers |
| GET | `/api/dealers/<dealer_id>/` | Retrieve a dealer |
| GET, POST | `/api/orders/` | List or create draft orders |
| GET | `/api/orders/<order_number>/` | Retrieve an order |
| POST | `/api/assistant/chat/` | Chat with the AI assistant |
| GET | `/api/forecast/<sku>/` | Retrieve SKU forecast and reorder calculations |
| GET | `/api/reorder-suggestions/` | List products needing replenishment |

List endpoints support pagination where implemented. See `API_CONTRACT.md` for request fields, response formats, validation rules, and error codes.

## AI Assistant

The assistant uses Groq tool calling to perform supported DealerDesk operations. Its tools include:

- `search_parts`
- `check_stock`
- `create_draft_order`

Stock and prices are obtained from application data. The assistant creates draft orders rather than directly confirming or fulfilling orders.

Configure `GROQ_API_KEY` in `.env` before testing live assistant requests.

## Forecasting

The generated forecast file is `ml/forecast.csv`. It contains these columns:

- `sku`
- `week_start`
- `forecast_units`

The current file contains 40 SKUs over four forecast weeks. Reorder calculations combine forecast demand over the supplier lead time with safety stock calculated from recent historical sales variability. The current database stock determines the suggested reorder quantity.

Regenerate forecasts using:

```bash
python manage.py generate_forecast
```

## Validation and Smoke Testing

Run Django's system checks and migration consistency check:

```bash
python manage.py check
python manage.py makemigrations --check --dry-run
```

Run the supplied smoke test against a running server:

```bash
python smoke_test.py --base-url http://localhost:8000 --track aiml
```

The grading environment uses PostgreSQL and runs migrations and the seed command. Keep `requirements.txt`, migrations, the forecast CSV, and required data files in the repository.

## Project Decisions

See `DECISIONS.md` for the architectural and implementation decisions.

## Security

- Keep `.env` out of version control.
- Use placeholder values in `.env.example`.
- Never commit API keys, database passwords, or production secret keys.
