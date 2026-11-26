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

Never build an EXPINC_INVOICE with `create_entity`: its totals and links would be wrong, and it would fail when it is posted. `create_expense_invoice` also takes the invoice's number itself, so do not call `fetch_counter_next` before it: every call uses up a number.

1. **Find the shipment and its cost lines.** `search` or `list_entities` for the JOURNEY. Then `list_entities` JOURNEY_STEP_EXPENSE filtered by `journeyId`. Use only lines whose `actualFullCh` is not Y: a line with Y is already settled in full.
2. **Check that the lines can share one invoice.** They must have the same vendor sub account, the same actual currency, the same sub department and the same side. At most 5 lines per invoice. Other lines go on a second invoice.
3. **No cost line for this cost?** Tell the user. A cost line is planned on the shipment. Add one only after reading the JOURNEY_STEP_EXPENSE schema, showing the user the line, and getting a yes.
4. **Show the user the invoice and get a yes:** vendor, shipment, each cost line with its amount in the invoice currency, invoice date, due date, the vendor's invoice number, for each line whether this bill settles it in full, and the file you will attach, if any.
5. **Store the vendor's file**, when the user gave you one. See "The vendor's file" below. Skip this step when there is no file.
6. **`create_expense_invoice`** with `invoiceDate`, `dueDate`, `currencyId`, `reference` (the vendor's invoice number), `explanation`, and `lines`: `[{"journeyStepExpenseId": "...", "amount": 1250.00, "invoiceActualFullCh": "Y"}]`.
   - `amount` is in the invoice currency, already converted.
   - `invoiceActualFullCh` Y closes the cost line. N leaves the rest of the accrual open.
   - The invoice date cannot be earlier than the latest accrual date of the lines.
   - `documents` takes the stored file: `[{"fileKey": "...", "fileName": "...", "fileType": "...", "fileSize": 48213}]`, the four values `upload_invoice_document` returned. Leave it out when there is no file.
7. **Ask for approval** when the user wants it posted.
   - `validate_invoice_for_approval` with `entityName` EXPINC_INVOICE.
   - Send only when it returns `success` true. Otherwise show what it names, correct it with the user and check again. `VALIDATION_UNAVAILABLE` means the check could not run: do not send, and tell the user.
   - Then `update_entity` EXPINC_INVOICE with `{"approvalCh": "P"}`.
8. **Read the result** with `get`. `activatedCh` Y means it is posted. If it was refused, `approvalCh` is back to N and `approvalExplanation` says why.

**When something fails.**

- If `create_expense_invoice` itself fails, report its message exactly. The tool undoes its own partial work where it can. Do not repair or delete records yourself.
- If a later step fails, the invoice exists as a draft and its cost lines are already consumed and linked. Tell the user exactly that, and leave it. Do not delete it here: it is corrected or taken back in the web application.

## The vendor's file

`upload_invoice_document` stores the vendor's document in the workspace, and `create_expense_invoice` attaches it. What is stored must be the file the user gave you, byte for byte.

**When you hold the file and can run a tool on it** (a code sandbox, a terminal):

1. From the file itself, compute its SHA-256 and its base64 encoding with a tool. For example `sha256sum invoice.pdf` and `base64 -w0 invoice.pdf`.
2. Call `upload_invoice_document` with `fileName`, `contentBase64` and `sha256`. Send the encoding exactly as the tool printed it.
3. The answer returns `fileKey`, `fileName`, `fileType` and `fileSize`. Pass all four as one item of `documents`.

The limit is 1 MB of file. The encoding is written out in the tool call, so from a chat you can send far less than that at once. When the encoding is longer than you can send whole in one call, do not split, shrink, convert or retype the file: use the upload link or the fallback below.

**From a terminal you can use an upload link instead.** Call `upload_invoice_document` without `contentBase64`. It returns a link that is valid for 5 minutes. Send the file to it with an HTTP PUT and the same content type you passed, for example `curl -X PUT -H "Content-Type: application/pdf" --data-binary @invoice.pdf "<link>"`, then pass the returned `fileKey` in `documents`. This also takes a file over 1 MB.

**The fallback.** Create the invoice without `documents` and tell the user to attach the file on the invoice's page in the web application. Use it when:

- You do not hold the file's bytes. Seeing a document's pages or its text is not holding the file.
- The file is too large to send.
- The upload was refused twice.

Never do any of these:

- Send the text of a document, a picture you made of it, or a copy you rebuilt, as if it were the vendor's file.
- Type, complete or correct base64 or a digest by hand. A refusal that says the bytes do not match `sha256` means the encoding was damaged on the way: encode again from the file and send that.

## A goods invoice

You do not build one here. For one that already exists you can:

- Read it and its lines: `get` with `includeDetails` and `includeReferences`.
- Check it: `validate_invoice_for_approval` with the invoice id. It names every field that would stop the approval.
- Send it for approval when the user asks, and only after the check returned `success` true: `update_entity` INVOICE with `{"approvalCh": "P"}`. On `VALIDATION_UNAVAILABLE` do not send.
- Read it again. `activatedCh` Y means posted, `approvalExplanation` says why when it was refused.

`approvalCh` can only be moved to P, or to D to ask for a posted invoice to be deactivated. `activatedCh` cannot be changed at all. The approval workflow writes both.
