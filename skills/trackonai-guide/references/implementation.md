# Setting up a new company

Use this when the user is new to TrackonAI, adds a branch, or moves from another system. You are the guide and the user decides. Work in short rounds: ask, propose, confirm, write, report.

Keep a running checklist in the conversation of what is set up, what is open and the ids you created. A setup takes more than one sitting, and the user needs to see where it stands.

## Choose how to start

| Way | How it goes | Fits |
|---|---|---|
| **Deal first** | Take one real deal the company has now. Build exactly what that contract needs, send it through approval, then widen from there | A new or small company. It works in the first session, and every record created is one that is needed |
| **Structure first** | Set up the structure, the chart of accounts and the master data, load the opening balances, then start trading | A running business that moves its accounting from another system |

Ask the user which fits. Either way the order below holds: it is the order the records depend on each other.

## Step 0. See what the workspace already has

A new workspace comes with standard reference data, and it may hold sample records. Before you propose anything, list what exists and tell the user: what is there, what is missing, what looks like a sample.

Read with `list_entities`: COMPANY, BRANCH, DEPARTMENT, SUB_DEPARTMENT, PRODUCT_LINE, CURRENCY, GL_ACCOUNT, GL_CLASS, ACCOUNT_TYPE, MODULE, TRANSACTION_TYPE, UOM_REF, INCOTERM, PAYMENT_TERM, PACKAGE_TYPE, DISPATCH_POINT, COMPANY_PARAMETER, REPORT_TYPE, REPORT_BITMAP.

Do not treat sample records as the company's own, and do not delete them unless the user asks.

## The order of a setup

Each step needs the ones above it. Skip what already exists and suits the company.

1. **Company and branches.** COMPANY carries the names, the main currency and the accounting period. BRANCH carries its code, country, reporting currency and default letterhead. Everything else hangs on a branch, so confirm these with care.
2. **Departments and warehouses.** DEPARTMENT with its type, then one SUB_DEPARTMENT per warehouse and branch, including one for stock in transit. Then the PRODUCT_LINE records the company trades in.
3. **Currencies and rates.** The CURRENCY records in use, and an EXCHANGE_RATE per currency and date (`coefficient`). A document in a foreign currency needs the rate of its date.
4. **Chart of accounts.** GL_ACCOUNT is a tree. See the rules below. Agree the chart with the company's accountant before you create anything, and prefer adjusting the standard chart to building a new one.
5. **Posting rules.** GL_CLASS decides which GL accounts a kind of account or product posts to, and ACCOUNT_TYPE groups accounts per kind. See below.
6. **Accounts.** Customers and vendors (`counterparties.md`), income and expense accounts, bank and cash accounts, each with a sub account per branch.
7. **Products.** Units (UOM_REF), then PRODUCT with its product line, base unit and GL class.
8. **Trade references.** INCOTERM, PAYMENT_TERM, PACKAGE_TYPE, ports (DISPATCH_POINT), the document types a contract can require (CONTRACT_DOCUMENT_TYPE) and standard clauses.
9. **Company parameters.** See below.
10. **Documents.** The templates and letterheads the company sends out. A letterhead (REPORT_BITMAP) belongs to one of the company's own accounts. Logos and images are uploaded in the web application.
11. **People and approvals.** Done by the workspace administrator in the web application: users, roles, who may read or write through the connector, and whether documents are approved automatically or by a person.
12. **Opening balances** at the cutover date: `opening-balances.md`.
13. **One real deal from start to end** as the acceptance test: contract, approval, shipment, invoices, posting, reports. A setup is done when this works, not when the lists are full.

## Chart of accounts rules

- The `code` of a GL account is its identity. It cannot be changed after the account is used, so agree the coding first.
- The code carries the tree: a child's code starts with its parent's code and a dot (`1.1.10` under `1.1`). Set `upCode` to the parent's code.
- Only an account with no children takes postings. Adding a child under a posting account turns it into a group, so never add a child under an account that already has postings without asking.
- Keep sibling codes the same width (`01`, `02`, not `1`, `2`) so that they sort correctly.

## Posting rules (GL_CLASS)

- A class for counterparties or taxes points to one GL account (`glCode`). Every account of that class posts there.
- A class for products (`moduleCode` PRO) holds one GL account per transaction type (`glCodes`), because a purchase, a sale and a stock movement of the same product post to different accounts. It also carries the product's tax rates.
- A wrong class sends correct documents to wrong accounts. Propose classes, and let the accountant confirm them.

## Company parameters

- COMPANY_PARAMETER holds the company's settings. The id is the parameter's name.
- A parameter that applies to one branch carries the branch in its name: `<NAME>_FOR_BRANCH_<branchId>`.
- A parameter that holds a list is stored as text wrapped in commas: `,B2,B7,`.
- Read the existing parameters before proposing one, and change one only when the user asks. A parameter changes how the system behaves for everyone.
- Parameters this guide relies on: `FAST_CONTRACT_TRANSACTION_TYPE_PURCHASE` and `FAST_CONTRACT_TRANSACTION_TYPE_SALES` (the default transaction type of a contract line), `ALLOW_UNBALANCED_OPENING_BALANCES`, and `OPENING_BALANCE_EQUITY_FOR_BRANCH_<branchId>`.

## Many records at once

Through the connector you create records one at a time, each shown to the user first. That suits tens of records, not hundreds.

For lists, point the administrator to the Import wizard in the web application. It loads customers and vendors with their addresses, products, income and expense accounts and contract clauses from a spreadsheet, and it offers a template for each. Help the user prepare the spreadsheet: map their columns to the template, clean duplicates, and check codes against what the workspace already has.

## Moving from another system

1. **Agree the cutover date** per branch. Balances are taken at that date and the old system is no longer changed for earlier dates.
2. **Master data first:** the chart of accounts, customers, vendors, products. Clean it before loading it. Old duplicates are easier to merge in a spreadsheet than in a ledger.
3. **Balances, not history.** Load the closing balances and the open items of the old system, not its past transactions. Open receivables and payables are loaded one line per open invoice, with the original invoice number, so that later payments can be matched to them.
4. **Open business.** Contracts that are still running are entered as contracts, with their open quantities.
5. **Prove it.** The opening entry is complete when total debit equals total credit in `opening_balance_position`, and the account balances agree with the old system's trial balance. Show the user both.

## What to be careful with

- **Codes are forever.** Account codes, GL codes, product codes and branch codes show up on every later document. Settle the conventions before the first record.
- **Ask, do not assume, on accounting choices.** Which account a balance belongs to, which class a product takes and what the cutover date is are the accountant's decisions.
- **Label trial records.** If the user wants to try something, name the record so that it is clearly a trial, keep it a draft, and delete it when they are done.
- **Say what is outside your reach.** Users, approval settings, templates and images, and period locks are set in the web application.
