# TrackonAI data model

Use this to turn the user's words into the right record type. Field names here are examples. `get_entity_schema` has the full and current list for the user's workspace.

## From trader language to record type

| The user says | Type | Notes |
|---|---|---|
| Contract, deal, purchase or sale agreement | CONTRACT | `purchaseCh` Y is a purchase, N a sale. `status` starts as Draft. |
| Product line of a contract, open position, remaining quantity, exposure | POSITION | One per product on a contract. `sourceId` is the contract id. |
| Stock in a warehouse | POSITION with `positionType` Inventory | Per SUB_DEPARTMENT. The inventory reports give the same picture. |
| Loading and discharge ports of a contract | CONTRACT_PORT | `originCh` Y is the loading port, N the discharge port. `portId` points to DISPATCH_POINT. |
| Documents a contract requires | CONTRACT_DOCUMENT | One per document type (CONTRACT_DOCUMENT_TYPE). |
| Containers planned on a contract | CONTRACT_CONTAINER | |
| Standard clause texts | CONTRACT_CLAUSE | The library. A contract's own clause text is stored on the CONTRACT record. |
| Shipment, voyage, vessel, ETA, arrival | JOURNEY | Dates such as `etd`, `eta` and `arrivalDate`, and `vesselName`. |
| Cargo allocated to a shipment | CONSIGNMENT | Links the shipment to contract positions. Its product movements are PRODUCT_TRANSACTION. |
| Containers on a shipment | JOURNEY_CONTAINER, CONTAINER_LINKING | |
| A shipment's cost lines: freight, inspection, port charges | JOURNEY_STEP_EXPENSE | Estimate, accrual and actual per expense account. |
| Documents received for a shipment (B/L, COA, packing list) | JOURNEY_DOCUMENT | `documentKey` is what `get_document_url` needs. |
| Documents generated for a shipment (shipping instructions, delivery order) | JOURNEY_STEP_DOCUMENT | |
| Invoice for goods, receivable, payable | INVOICE | `purchaseCh` Y is payable, N receivable. Its lines are the shipment's PRODUCT_TRANSACTION rows. |
| Vendor bill for a cost, service invoice, income invoice | EXPINC_INVOICE | `expenseCh` Y is an expense. Lines are EXPINC_INVOICE_DETAIL. |
| Payment request | PAYMENT_REQUEST | |
| Payment, receipt, debit note, credit note, journal entry, bank statement line | DIRECT_POSTING | A manual ledger entry. See `ledger-entries.md`. |
| Opening balances from a previous system | OPENING_BALANCE_BATCH | Lines are OPENING_BALANCE_LINE. See `opening-balances.md`. |
| Ledger rows | ACCOUNT_TRANSACTION | Read only. Written by the posting services. |
| A person, a company or a vessel | CONTACT | The identity. `contactType` C company, P person, V vessel. |
| Address of a party | ADDRESS | Belongs to the CONTACT. |
| Customer, vendor, partner or employee account | ACCOUNT | Belongs to a CONTACT. `moduleCode` says which kind. |
| The account used on contracts, invoices and ledger entries | SUB_ACCOUNT | Belongs to an ACCOUNT, one per branch. |
| Income account, expense account, bank account | ACCOUNT and SUB_ACCOUNT | The same pair without a contact. MODULE lists the kinds. |
| Product, commodity | PRODUCT | Belongs to a PRODUCT_LINE. `baseUom` is its base unit. |
| Unit of measure | UOM_REF | |
| Packaging | PACKAGE_TYPE | |
| Warehouse, stock in transit | SUB_DEPARTMENT | "In Transit" is a sub department too. |
| Business division | DEPARTMENT | Parent of SUB_DEPARTMENT. |
| Branch, office | BRANCH | Accounts, warehouses and documents belong to a branch. |
| The company itself | COMPANY | |
| Port, loading point | DISPATCH_POINT | |
| Country, city, town | COUNTRY, CITY, TOWN | |
| Currency, exchange rate | CURRENCY, EXCHANGE_RATE | A rate is per currency and date (`coefficient`). |
| Chart of accounts | GL_ACCOUNT | A tree: `code` is the id, `upCode` the parent. |
| Posting rules of an account or a product | GL_CLASS | Maps a kind of account or product to GL accounts. |
| Transaction types | TRANSACTION_TYPE | |
| Payment terms, incoterms | PAYMENT_TERM, INCOTERM | |
| Company settings | COMPANY_PARAMETER | The id is the parameter name. |
| Document kinds and templates | REPORT_TYPE, REPORT_TEMPLATE | Used by `generate_document`. |
| Letterheads | REPORT_BITMAP | |
| Reports | ANALYTICS_QUERY | Use the report tools. See `reports.md`. |

A workspace can have types of its own, and a type listed here can be missing. If a type is not found, ask `get_entity_schema` and tell the user what you found.

## How the types hang together

```
CONTACT                         the person, company or vessel: always first
├── ADDRESS                     contactId
└── ACCOUNT                     contactId, moduleCode (customer, vendor, partner, employee)
    └── SUB_ACCOUNT             upId = account id, one per BRANCH: used on contracts,
                                invoices and ledger entries

DEPARTMENT
└── SUB_DEPARTMENT              upId = department id, belongs to a BRANCH: warehouses, In Transit
PRODUCT_LINE
└── PRODUCT                     productLineId, baseUom, glClassId

CONTRACT                        purchase or sale
├── POSITION                    sourceId = contract id, one per product
├── CONTRACT_PORT               contractId, originCh Y loading, N discharge
├── CONTRACT_DOCUMENT           contractId, required documents
└── CONTRACT_CONTAINER          contractId

JOURNEY                         the shipment
├── CONSIGNMENT                 cargo, linked to contract positions
│   └── PRODUCT_TRANSACTION     product movements, also the lines of the goods INVOICE
├── JOURNEY_STEP_EXPENSE        journeyId, cost lines
├── JOURNEY_DOCUMENT            journeyId, documents received
└── JOURNEY_STEP_DOCUMENT       documents generated

EXPINC_INVOICE                  expense or income invoice
└── EXPINC_INVOICE_DETAIL       upId = invoice id

OPENING_BALANCE_BATCH
└── OPENING_BALANCE_LINE        upId = batch id

GL_ACCOUNT                      tree by code and upCode
```

## Conventions

- **Flags** end in `Ch` and hold Y or N: `purchaseCh`, `activatedCh`, `inactiveCh`.
- **Names next to ids.** Many records store a name beside an id (`subAccountId` with `subAccountName`, `currencyId` with `currencyName`). When the schema has both, write both, with the name taken from the record you point to.
- **Approval fields.** `approvalCh` N is a draft, P waits for approval, D waits for a deactivation, Y is approved. `activatedCh` Y means posted to the ledger. `approvalExplanation` says why a request was refused.
- **Currencies.** Amounts are kept in the document currency and converted to the company's base and reporting currencies (fields starting with `bc`, `rc` and `brc`). Give amounts in the document currency unless a tool asks for more.
- **Ids.** Leave the id empty and the system makes one, except where the web application composes it, so that both ways of working produce the same record:

| Type | Id |
|---|---|
| POSITION | `<contractId>#<productId>` |
| CONTRACT_PORT | `<contractId>#<originCh>#<portId>` |
| CONTRACT_DOCUMENT | `<contractId>#<docTypeId>` |
| GL_ACCOUNT | its `code` |
| COMPANY_PARAMETER | its `parameterName` |
| DIRECT_POSTING for a bank statement line | a stable id built from the bank account and the line. See `ledger-entries.md` |
