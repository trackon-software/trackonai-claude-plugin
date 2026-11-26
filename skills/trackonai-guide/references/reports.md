# Reports

TrackonAI has a catalogue of ready reports: shipment profit and loss, AR/AP aging, inventory aging, sales and purchases by product, open contract details, account statements and more. A workspace can have reports of its own. Always read the catalogue: do not assume a report exists or what its parameters are.

## Report or list?

- A question about a few records ("the ETA of this shipment", "open contracts with this supplier") is a `search` or a `list_entities`.
- A question that needs totals, aging, profit or a period ("who owes us what, by age", "profit by trader for March") is a report. Do not add up pages of records yourself when a report exists for it.

## Steps

1. **`list_analytics_queries`** to find the report and its parameters. If the answer says the catalogue was not read to the end, read the rest with `list_entities`, `entityName` ANALYTICS_QUERY, passing the `nextToken` it gave.
2. **Parameters.** For a parameter with a fixed list of values (a branch, a sub account, a product), call `get_parameter_values` with the report's id and the parameter name, and a `searchTerm` when the list is long. Ask the user for anything you cannot derive, such as the period.
3. **`run_analytics_query`** with the report's id and `parameters`, a JSON array with one entry per parameter: `[{"parameterName": "DATE_FROM", "value": "2026-01-01"}, {"parameterName": "DATE_TO", "value": "2026-01-31"}]`. Use the parameter names exactly as the catalogue gives them. Values are text.
4. **`get_query_results`** with the `executionId`. Reports run in the background: call again until the status is COMPLETED. Report a failed run with its message.
5. **The result** is a temporary download link to a CSV file. If you can open links, read the file and answer from it. Otherwise give the link to the user. It is for the user only.

## Answering from a report

- Say which report you ran, with which parameters and for which period.
- Show amounts with their currency. A report can carry the same amount in the document currency and in the company's currencies: say which column you used.
- When a figure looks wrong, say so and show the rows behind it. Do not smooth it over.

## Common questions

| Question | Where to look |
|---|---|
| Which shipments arrive this week? | `list_entities` JOURNEY filtered by `eta` |
| How much of a product is still open? | `list_entities` POSITION filtered by product and status |
| Open purchase contracts with a supplier | `list_entities` CONTRACT filtered by counterparty, `purchaseCh` and status, or the open contract details report |
| Who owes us money? What do we owe? | The AR/AP aging report |
| Profit of a shipment, a trader or a month | The shipment profit and loss report |
| What is in a warehouse? | The inventory reports, or POSITION records of the SUB_DEPARTMENT |
| What happened on an account in a period? | The `Sub Account Statement` report |
