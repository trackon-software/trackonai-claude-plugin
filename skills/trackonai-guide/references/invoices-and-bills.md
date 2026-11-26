# Invoices and vendor bills

TrackonAI has two kinds of invoice.

| Kind | Type | Where it is made |
|---|---|---|
| Invoice for goods bought or sold | INVOICE | In the web application, from the shipment. Its lines are the shipment's product movements |
| Vendor's bill for a cost of a shipment (freight, inspection, survey, port charges), or an income invoice for a service | EXPINC_INVOICE | Here, with `create_expense_invoice` |

A cost or a payment that belongs to no shipment is not an invoice. It is a manual ledger entry: see `ledger-entries.md`.

Every invoice posts to the ledger only through approval. You create or check it, ask for approval and read the result.

## A vendor's bill for a shipment cost

The cost line of the shipment is the anchor. The bill turns its accrual into an actual, and the invoice is linked to it. `create_expense_invoice` does all of that in one call: the invoice with correct totals, one line per cost line, the cost lines consumed and linked, and the payment requests of a branch that uses them.

Never build an EXPINC_INVOICE with `create_entity`: its totals and links would be wrong, and it would fail when it is posted. The descriptions of `create_entity` and `fetch_counter_next` still describe that older, record by record way. Do not follow it.

1. **Find the shipment and its cost lines.** `search` or `list_entities` for the JOURNEY. Then `list_entities` JOURNEY_STEP_EXPENSE filtered by `journeyId`. Use only lines whose `actualFullCh` is not Y: a line with Y is already settled in full.
2. **Check that the lines can share one invoice.** They must have the same vendor sub account, the same actual currency, the same sub department and the same side. At most 5 lines per invoice. Other lines go on a second invoice.
3. **No cost line for this cost?** Tell the user. A cost line is planned on the shipment. Add one only after reading the JOURNEY_STEP_EXPENSE schema, showing the user the line, and getting a yes.
4. **Show the user the invoice and get a yes:** vendor, shipment, each cost line with its amount in the invoice currency, invoice date, due date, the vendor's invoice number, and for each line whether this bill settles it in full.
5. **`create_expense_invoice`** with `invoiceDate`, `dueDate`, `currencyId`, `reference` (the vendor's invoice number), `explanation`, and `lines`: `[{"journeyStepExpenseId": "...", "amount": 1250.00, "invoiceActualFullCh": "Y"}]`.
   - `amount` is in the invoice currency, already converted.
   - `invoiceActualFullCh` Y closes the cost line. N leaves the rest of the accrual open.
   - The invoice date cannot be earlier than the latest accrual date of the lines.
6. **Ask for approval** when the user wants it posted.
   - `validate_invoice_for_approval` with `entityName` EXPINC_INVOICE.
   - Send only when it returns `success` true. Otherwise show what it names, correct it with the user and check again. `VALIDATION_UNAVAILABLE` means the check could not run: do not send, and tell the user.
   - Then `update_entity` EXPINC_INVOICE with `{"approvalCh": "P"}`.
7. **Read the result** with `get`. `activatedCh` Y means it is posted. If it was refused, `approvalCh` is back to N and `approvalExplanation` says why.

**The vendor's file.** `upload_invoice_document` gives an upload link that needs an HTTP PUT. Claude apps usually cannot send one. Unless you can send HTTP requests yourself, skip the upload, create the invoice without documents and tell the user to attach the file on the invoice's page in the web application.

**When something fails.**

- If `create_expense_invoice` itself fails, report its message exactly. The tool undoes its own partial work where it can. Do not repair or delete records yourself.
- If a later step fails, the invoice exists as a draft and its cost lines are already consumed and linked. Tell the user exactly that, and leave it. Do not delete it here: it is corrected or taken back in the web application.

## A goods invoice

You do not build one here. For one that already exists you can:

- Read it and its lines: `get` with `includeDetails` and `includeReferences`.
- Check it: `validate_invoice_for_approval` with the invoice id. It names every field that would stop the approval.
- Send it for approval when the user asks, and only after the check returned `success` true: `update_entity` INVOICE with `{"approvalCh": "P"}`. On `VALIDATION_UNAVAILABLE` do not send.
- Read it again. `activatedCh` Y means posted, `approvalExplanation` says why when it was refused.

`approvalCh` can only be moved to P, or to D to ask for a posted invoice to be deactivated. `activatedCh` cannot be changed at all. The approval workflow writes both.
