# Manual ledger entries and bank statements

Payments, receipts, debit notes, credit notes and general entries (for example moving a balance from one account to another) are **direct postings**: DIRECT_POSTING records. "Direct posting" is the accounting name for a manual entry. It is not a way around approval.

Never use `create_entity` for them, and never write ACCOUNT_TRANSACTION.

A vendor's bill for a cost of a shipment is not a direct posting. It is an expense invoice: see `invoices-and-bills.md`.

## One entry

1. **Work out the entry and show it to the user:** the accounts, which side each is on, the amounts, the currency and the date. Get a yes before you save anything.
2. **`save_direct_posting`** with `data` holding:
   - `documentType`: DEBIT_NOTE or RECEIPT (the main account is debited), CREDIT_NOTE or PAYMENT (the main account is credited), or GENERAL with `mainSide` DEBIT or CREDIT.
   - `transactionDate` (YYYY-MM-DD), `companyId`, `currencyId`, `mainSubAccountId`.
   - `oppositeLines`: `[{"subAccountId": "...", "amount": 100, "explanation": "..."}]`, positive amounts in `currencyId`.
   - Optional `reference`, `documentNumber`, `explanation`.

   The main account takes the total of the opposite lines on the other side, so debit always equals credit. Every account is a SUB_ACCOUNT id of the same branch. Give amounts in the document currency only. The system converts them.

   Saving posts nothing. The answer returns the draft's `id` and `revision`. If the save is refused with a message that names a field, show it to the user, correct it and save again.
3. **`activate_direct_posting`** with the `id`. This sends the draft to the workspace's approval workflow. It never writes the ledger itself. With express approval the entry is posted within seconds once the checks pass. With manual approval it waits for a person in the web application.
4. **Read it again** with `get`. `activatedCh` Y means it is posted. If it was refused, `approvalExplanation` says why.

To change a draft before it is sent, call `save_direct_posting` with the tool's `id` argument and `expectedRevision` (the revision you last read). Only a draft can be changed.

To take a posted entry back, `activate_direct_posting` with `activate: false` asks for its deactivation, which also goes through approval. It is refused in a locked period or when a row is already matched.

**Example.** Customer A's debt of 100 USD moves to customer B: `documentType` GENERAL, `mainSubAccountId` B, `mainSide` DEBIT, `oppositeLines` `[{"subAccountId": "A", "amount": 100}]`. Afterwards the open items on A still have to be matched in the web application.

## A cost or a bill with no shipment

`documentType` GENERAL, `mainSubAccountId` the vendor, `mainSide` CREDIT, opposite lines the expense sub accounts. When the bank pays it, that statement line is a PAYMENT with the vendor as the opposite line. The two are matched later in the web application.

## A bank statement

One direct posting per statement line.

1. **Identify** the bank's SUB_ACCOUNT and the statement period.
2. **Find what is already booked.** Run the `Sub Account Statement` report for the bank sub account and the period (see `reports.md`). Skip every statement line that is already in the ledger with the same date, amount and reference.
3. **Work out each remaining line.** Nothing is saved yet.
   - `documentType` PAYMENT when the bank pays out, RECEIPT when it receives.
   - `mainSubAccountId` the bank sub account.
   - One opposite line: the vendor or customer when the line settles their invoice, otherwise the expense or income sub account (bank charges, interest).
4. **Show the user every entry** (date, accounts, side, amount, reference) and get one yes for the batch. List separately the lines you could not place, and ask.
5. **Save and send each confirmed line.** `save_direct_posting`, then `activate_direct_posting`, then `get`.
   - Give each line a stable id of your own, in the tool's `id` argument and never inside `data`: `BANK-<bank sub account id>-<date>-<line reference>`, or the line's sequence number within that day when the statement gives no reference.
   - Put the line reference in `documentNumber` inside `data`.
   - With a stable id, running the same statement twice is refused ("expectedRevision is required") instead of booking a line twice. Without it, the guard is lost.
6. **Close the statement.** From the same report, sum the debit and credit columns in the company currency from the opening row through the period and compare with the statement's closing balance. Report the difference, the lines still waiting for approval and the lines you could not place.

## What you never do

- Save or activate an entry the user has not seen and confirmed. With express approval it posts to the ledger.
- Move `approvalCh` or `activatedCh` with `update_entity`. Asking for an approval or a deactivation is done with `activate_direct_posting` only.
- Change or delete a posting after it was sent.
- Split, round or net amounts to make a statement tie out. Report the difference instead.
