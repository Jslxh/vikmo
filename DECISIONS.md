# Vikmo DealerDesk — Technical Decisions

## 1. Backend Architecture

- **Framework:** Django with Django REST Framework (DRF).
- **Database:** PostgreSQL, configured through environment variables.
- **API design:** REST endpoints expose product catalogue, inventory, dealer, order, assistant, and forecasting functionality.
- **Validation:** Incoming API data should be validated before database writes, using the project's serializers and validation logic.
- **Error handling:** API errors should follow the response formats defined in `API_CONTRACT.md` so clients can handle failures consistently.
- **Separation of concerns:** Business operations, database access, assistant orchestration, and forecasting should remain separated so that each can be tested and maintained independently.

**Rationale:** Django and DRF provide a structured foundation for data-backed APIs, request validation, database integration, and reusable business logic.

**Implementation evidence to add:** The actual settings, serializer, view, and service file paths and line ranges.

## 2. Data Management and Order Integrity

- Product catalogue and dealer records are loaded from the provided data files through the `seed_data` management command.
- The seed operation is intended to be idempotent: repeated runs should not create duplicate seeded products or dealers.
- Product SKUs and dealer codes are used as identifiers.
- Order creation uses catalogue prices at the time of order creation and creates orders with `DRAFT` status.
- Inventory changes should be made through the inventory API or corresponding business logic rather than by relying on language-model-generated values.
- Order totals and line-item prices should be calculated or validated by the application using trusted catalogue data.

**Rationale:** The application database is the source of truth for catalogue, stock, and pricing information. This reduces inconsistencies between assistant responses and actual business records.

**Verification required:** Confirm the actual unique constraints, seed implementation, order-price snapshot behavior, and inventory update rules in the source code.

## 3. AI Assistant Architecture and Tool Calling

The assistant uses Groq as its language-model provider and supports tool-based interaction with DealerDesk operations.

The available tools are:

- `search_parts` — search the parts catalogue.
- `check_stock` — retrieve current inventory information.
- `create_draft_order` — create a draft order through the application.

### Tool-calling flow

1. Receive the user's message and relevant conversation context.
2. Send the message to the configured language model with the available tool definitions.
3. If the model requests a tool, validate and execute the request through the corresponding application function.
4. Return the tool result to the model in the format required by the provider.
5. Allow the model to formulate a response using the returned data, or execute another supported tool if needed.
6. Return the final response to the user.

The actual implementation should enforce a bounded tool-execution loop and handle invalid arguments, tool failures, and provider errors.

**Rationale:** Tool calling lets the language model interpret user intent while application functions perform data retrieval and state-changing operations.

**Implementation evidence to add:** The assistant endpoint, model request, tool dispatcher, tool implementations, and session-history logic, with exact line references.

## 4. Data Grounding, Clarification, and Conversation Context

- Product names, prices, and stock quantities should be retrieved from application data rather than invented by the model.
- The assistant should request clarification when essential order details are missing or ambiguous.
- A draft order should only be created when the required information is available and the requested operation passes application validation.
- Conversation history should be scoped to the intended session so that unrelated users or sessions do not share context.
- Previous conversation text must not override current database state when reporting prices or inventory.
- The assistant should remain within supported DealerDesk operations and respond appropriately to unrelated requests.

**Rationale:** Grounding factual responses in the database and validating actions in application code improves reliability and prevents the language model from becoming the authority for business-critical values.

**Verification required:** Confirm the exact session mechanism, clarification behavior, authorization checks, and off-topic handling implemented by the current code.

## 5. Prompt Injection and Security Boundaries

Product descriptions, catalogue fields, retrieved documents, and user messages must be treated as untrusted input.

- Catalogue content should be treated as data, not as instructions that can change the assistant's operating rules.
- The model should not be permitted to invent additional tools or perform unsupported operations.
- Tool arguments should be validated by application code.
- Stock changes and order creation should be subject to business rules enforced outside the model.
- The assistant should create draft orders rather than directly confirming or fulfilling orders.
- Secrets such as API keys and database credentials should be read from environment configuration and excluded from version control.
- `.env.example` should contain placeholders rather than real credentials.
- Errors should not expose credentials or unnecessary internal information.

**Rationale:** Prompt instructions alone are not a sufficient security boundary. Application-level validation, restricted tool access, and controlled database operations are required.

**Verification required:** Confirm that these protections are actually implemented and tested. Do not claim resistance to every prompt-injection attack without evidence.

## 6. Demand Forecasting Approach

The project generates weekly SKU-level forecasts in `ml/forecast.csv`.

The expected output contains:

- `sku`
- `week_start`
- `forecast_units`

The current output has been validated to contain 160 rows, representing 40 SKUs over four forecast weeks, with non-negative forecast values.

The current generator uses recent historical sales observations to calculate a per-SKU average and produces forecasts for the configured weeks. See `core/management/commands/generate_forecast.py` for the implementation.

### Forecasting baselines

The historical backtest compares three methods:

1. **Last value:** Use the most recently observed weekly demand.
2. **12-observation average:** Use the average of the previous 12 weekly observations.
3. **Same week last year:** Use the observation 52 weeks before the target week.

These provide simple reference points for evaluating whether a forecasting approach improves on straightforward historical baselines.

**Rationale:** Baselines are easy to interpret and establish a reproducible comparison before introducing more complex forecasting models.

## 7. Historical Backtest and Evaluation Results

The rolling-origin four-week backtest produced the following results:

| Method | MAE | WAPE |
|---|---:|---:|
| Last value | 13.583 | 38.22% |
| 12-observation average | 10.998 | 30.95% |
| Same week last year | 12.686 | 35.70% |

Evaluation details:

- **Forecast origins:** 1,862.
- **Forecast horizons:** Four weeks.
- **Predictions per method:** 7,448.
- **Evaluation set:** The same eligible forecast origins for all three methods.
- **History requirement:** 52 prior weekly observations and four subsequent actual observations.

The 12-observation average achieved the lowest overall MAE and WAPE among the three evaluated methods. Its MAE was approximately 19.0% lower than the last-value baseline.

The results support choosing the 12-observation average among these tested baselines on this historical dataset. They do not establish that it is optimal for every SKU or future period.

The backtest can be reproduced with:

```bash
python ml/backtest_four_week.py
```

### Evaluation limitations

- The backtest measures performance on the available historical data; it does not guarantee future accuracy.
- Promotions and Diwali-related demand changes have not been independently validated by the reported aggregate metrics.
- New SKUs with insufficient history require a separately defined cold-start strategy.
- Intermittent-demand SKUs may not be well represented by a simple moving average.
- Aggregate metrics can hide poor performance for particular SKUs or forecast horizons.
- The historical backtest evaluates baseline methods and does not establish that more advanced models would perform better.
- The generated four-week CSV and the rolling historical backtest serve different purposes and should not be treated as the same evaluation.

## 8. Reorder Suggestions and Inventory Planning

Reorder suggestions combine forecast demand, supplier lead time, safety stock, and current inventory.

The documented safety-stock approach uses:

- A service factor of 1.65.
- Sample standard deviation of recent demand.
- The square root of supplier lead time.

The reorder quantity is intended to be the non-negative difference between the reorder point and current stock, with rounding applied as required by the implementation.

**Rationale:** A lead-time demand estimate helps identify stock that may be insufficient before replenishment arrives. Safety stock provides an additional buffer against demand variability.

**Verification required:** Check `API_CONTRACT.md` and the actual reorder implementation for the precise equations, units, rounding behavior, and treatment of missing or invalid lead times. The formula in this document must match the code exactly.

## 9. Configuration and Reproducibility

- Runtime configuration is supplied through environment variables.
- `.env` is excluded from Git.
- `.env.example` documents the configuration interface with placeholder values.
- Database migrations are part of the repository.
- The `seed_data` command provides initial catalogue and dealer records.
- Forecast generation is available through the Django management command.

The documented setup sequence is:

```bash
python -m pip install -r requirements.txt
python manage.py migrate --noinput
python manage.py seed_data
python manage.py seed_data
python manage.py runserver
```

The database must exist and the environment must be configured before running migrations and seeding.

A clean-clone setup should be considered verified only after the procedure succeeds in a fresh working directory with the documented configuration.

## 10. Testing and Validation Evidence

The following checks have been observed during development:

- **Django system check:** Passed with no issues.
- **Forecast CSV validation:** Passed — 160 rows, 40 SKUs, four expected forecast weeks per SKU, and non-negative forecast values.
- **API smoke test:** Passed 10 of 10 checks.
- **Four-week backtest:** Completed for all three baselines.
- **Django test discovery:** `python manage.py test` discovered zero tests.

The smoke test covers core API contract behavior, assistant response shape, and forecast endpoint response shape. It does not fully evaluate the quality of assistant answers, correctness of every tool call, prompt-injection resistance, or future forecasting accuracy.

The zero-test result must not be represented as a passing automated unit-test suite.

Before submission, run:

```bash
python manage.py check
python manage.py makemigrations --check --dry-run
python smoke_test.py --base-url http://127.0.0.1:8000 --track aiml
python ml/backtest_four_week.py
git diff --check
```

The API smoke test requires the application server and configured database to be available.

**Outstanding verification:** Clean-clone setup, seed idempotency in a fresh database, and additional automated tests for important business logic should be completed or clearly reported as limitations.

## 11. Current Limitations and Future Improvements

The current implementation prioritizes a clear API contract, database-grounded assistant operations, reproducible baseline forecasting, and reorder suggestions.

Potential improvements include:

- Add automated unit and integration tests for the assistant's tool-calling loop and business rules.
- Evaluate the assistant on representative requests, ambiguous orders, invalid inputs, and malicious catalogue text.
- Add SKU-level and horizon-level forecast diagnostics.
- Evaluate promotion and festive-demand features using historical backtesting without future-data leakage.
- Define a cold-start policy for new SKUs.
- Evaluate approaches designed for intermittent demand.
- Measure forecast quality against additional baselines before introducing more complex models.
- Verify clean-clone setup and seed idempotency in a fresh PostgreSQL database.

## 12. Implementation References

Add exact source-file and line references before final submission. The following files are starting points to inspect; verify their contents and line numbers against the final commit.

- `core/management/commands/generate_forecast.py` — forecast generation.
- `ml/backtest_four_week.py` — rolling-origin baseline evaluation.
- `ml/forecast.csv` — generated forecast output.
- `API_CONTRACT.md` — API behavior and reorder formula.
- `README.md` — setup and execution instructions.
- Assistant endpoint, tool dispatcher, and tool implementation files — locate the actual files in the repository and add their exact paths and line ranges.
- Model, serializer, and seed-command files — verify the relevant database constraints and business rules before citing them.

Do not add guessed line numbers. Every reference should point to code that directly supports the associated claim.