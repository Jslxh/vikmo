# Vikmo DealerDesk — Technical Decisions

## 1. Backend Architecture
- **Framework:** Django with Django REST Framework.
- **Database:** PostgreSQL, configured through `DATABASE_URL`.
- **API design:** REST endpoints with consistent JSON responses and structured error objects.
- **Validation:** Django REST Framework serializers validate incoming API data before database writes.

## 2. Data Management
- Product catalogue and dealer records are loaded from the provided CSV files.
- The `seed_data` management command is designed to be idempotent, so repeated execution does not create duplicate products or dealers.
- Product SKUs and dealer codes are unique identifiers.
- Order creation stores the catalogue price at the time of order creation and creates orders in `DRAFT` status.

## 3. AI Assistant
- The assistant uses Groq as the LLM provider.
- Tool calling allows the assistant to search parts, check live stock, and create draft orders.
- Product prices and stock levels are retrieved from the database rather than invented by the model.
- Conversation history is scoped by session ID to keep sessions separate.
- The assistant is restricted to supported DealerDesk operations.

## 4. Demand Forecasting and Reordering
- Forecast output is stored in `ml/forecast.csv`.
- The output contains 40 SKUs across four forecast weeks.
- Forecasting uses historical sales data to estimate future weekly demand.
- Lead-time demand is calculated from forecast units across the lead-time period.
- Safety stock uses a 1.65 service factor, sample standard deviation of recent sales, and the square root of lead time.
- Reorder quantity is the non-negative difference between the reorder point and current stock, rounded up as required.

## 5. Configuration and Security
- Database URLs, API keys, and the Django secret key are configured through environment variables.
- `.env` is excluded from Git; `.env.example` contains placeholders only.
- Supplier integration configuration is kept separate from application logic.

## 6. Testing and Reproducibility
- Django system checks and migration consistency checks are run before submission.
- The smoke test exercises the API.
- The seed command is tested for repeatability.
- Forecast output is checked for the required columns, SKU count, and forecast weeks.
