# Opening balances

Opening balances are the closing figures of the company's previous system, carried into TrackonAI at a cutover date, per branch. They have their own tools. Never use `save_direct_posting`, and never use `create_entity`, `update_entity` or `delete_entity` on OPENING_BALANCE_BATCH or OPENING_BALANCE_LINE.

A migration may be split over several batches, one per file or per day, with the same branch and cutover date. Together they form the branch's opening entry.

## 1. Read the file and classify every line

The file is usually a trial balance, lists of open receivables and payables, and bank balances. Ask for the branch and the cutover date when the file does not say.

| Line | `lineType` | Needs |
|---|---|---|
| A balance on a GL account only: inventory, fixed assets, equity, a bank or cash balance | GL | `glCode` |
| An open item of one customer | AR | `subAccountId`, and the original invoice's `documentNumber` and `reference` so it can be matched against a payment later |
| An open item of one vendor | AP | The same |

- Resolve every account and counterparty with `search` or `list_entities`. Never guess an id. A missing counterparty is created first: `counterparties.md`.
- There is no line type for a bank or cash account. Load its balance as a GL line on the bank's GL account, and tell the user that it will not appear on the bank sub account's statement.
- Each line carries its amount on exactly one side, `debit` or `credit`, in its `currencyId`. Add `rcDebit` or `rcCredit` and `brcDebit` or `brcCredit` when the file states the reporting currency amounts.
- When the file does not state them, pass the batch's `exchangeRate`, `rcExchangeRate` and `brcExchangeRate` and the import converts the lines. Ask the user for the rates of the cutover date. Never guess a rate.

## 2. Import

Show the user the line count and the debit and credit totals per currency, and get a yes. Then `import_opening_balances` with a `reference` that names the file, the `cutoverDate`, the `branchId` and the `lines`.

- Nothing reaches the ledger yet. The answer's `batchId` is a draft.
- If the import is refused, the message lists every bad line. Fix all of them and import again with the same `reference`: that replaces the draft instead of adding a second one.

## 3. Activate

Show the user the batch (reference, branch, cutover date, line count, totals) and get a yes. Then `activate_opening_balances` with the `batchId`.

- Like every ledger entry, this asks for approval: express or by a person. Read the batch with `get`. `activatedCh` Y means posted. `approvalExplanation` says why when it was refused.
- A batch that waits for approval cannot be imported again or sent again.
- Never edit or delete its lines. The posting refuses lines that changed after the batch was sent. Correct a draft by importing it again.
- Whether one batch may post unbalanced is company parameter `ALLOW_UNBALANCED_OPENING_BALANCES` (Y allows it). If activation is refused because the batch does not balance and the user is loading the migration in parts, tell them that parameter decides it.
- `activate: false` asks for approval to remove a posted batch's rows. It is refused in a locked period.

## 4. Check the position

`opening_balance_position` for the branch and cutover date reports total debit, total credit and the difference of every posted batch, in the company currency. Report it after every activation.

- While a migration is being entered, a difference is expected. The migration is complete when total debit equals total credit.
- It fails when nothing is posted yet for that branch and date, or when company parameter `OPENING_BALANCE_EQUITY_FOR_BRANCH_<branchId>` is not set. That parameter names the equity sub account the difference closes to. Tell the user.

## 5. Close

Only when the user confirms that every file is loaded, and has seen the remaining difference: `close_opening_balances`. It sends the difference to the equity sub account for approval. The difference is the one figure the company did not supply, so never close on your own judgement.

- It is refused while any batch of the entry, or its closing, waits for approval, and when the entry already ties out.
- `close: false` asks for approval to reopen the entry, for example when a forgotten file turns up. Once that is approved, call it once more with `close: false` to remove the closing draft. Then the file can be loaded and the entry closed again.
