# Contracts

A contract is a CONTRACT record with child records: one POSITION per product, its ports, and the documents it requires. It starts as a Draft. Only the approval workflow makes it Approved.

## What to know before you create one

The web application's contract screen does more than save the fields the user types. It converts quantities to the product's base unit, works out weights, reads the tax rate, and adds the company's default cost estimates, clauses and required documents. A contract created through the connector gets none of that by itself. So:

- Create it as a Draft and fill the computed fields as described below.
- Run `validate_contract_for_approval` and fix what it names.
- Tell the user which defaults the web application would have added, and ask them to review the contract there before it is sent for approval.

## 1. Gather the deal

Purchase or sale, counterparty, branch, warehouse, product, quantity and unit, price and currency, incoterm, payment term, shipment period, loading port (required) and discharge port (optional). Ask for what is missing. Do not guess commercial terms.

## 2. Find the records the contract points to

Use `search` or `list_entities`. Never invent an id.

| Needed | Type | If it is missing |
|---|---|---|
| Counterparty for that branch | SUB_ACCOUNT | Create it first: `counterparties.md` |
| Product, with its product line and base unit | PRODUCT | Ask the user. A product belongs to the company's product catalogue |
| Branch and warehouse | BRANCH, SUB_DEPARTMENT | Ask which to use |
| Currency, incoterm, payment term | CURRENCY, INCOTERM, PAYMENT_TERM | Ask which to use |
| Ports | DISPATCH_POINT | Ask which to use |

## 3. Read the schemas

`get_entity_schema` for CONTRACT and for POSITION. They list the required fields and the name fields stored beside ids (for example the sub department, currency, incoterm and product line names). Fill both the id and the name.

## 4. Show the contract and get a yes

One summary: side, counterparty, product, quantity, unit, price, total, currency, incoterm, payment term, shipment period and ports.

## 5. Create the records, in this order

**CONTRACT**

- `status` Draft. `purchaseCh` Y for a purchase, N for a sale.
- The references from step 2, the shipment period, and the totals: the sum of the position totals.

**POSITION**, one per product

- `id` `<contractId>#<productId>`, `sourceId` and `contractId` = the contract id.
- `positionType` PurchaseContract or SalesContract.
- `transactionType`: the contract transaction type of that side. It is the value of company parameter `FAST_CONTRACT_TRANSACTION_TYPE_PURCHASE` or `FAST_CONTRACT_TRANSACTION_TYPE_SALES` when the company has set one, otherwise BLP for a purchase and BLS for a sale. Never put PurchaseContract or SalesContract here: send for approval refuses it.
- `productId`, `productName`, `uom`, `baseUom` (the product's base unit), `currencyId`, `subDepartmentId`.
- `transactionQuantity`, and `availableQuantity` with the same value.
- `baseQuantity`, `netWeight` and `grossWeight`. When the deal's unit is the product's base unit they equal the quantity. Otherwise convert with the product's own unit definitions, show the conversion to the user, and say that the web application should be used to confirm it. Never leave them at 0: prices per base unit are computed from them.
- `grossUnitPrice`, `netUnitPrice` (the same value unless the user gives a discount) and `total` = price x quantity.
- `vatPercent`: the tax rate that applies to the product for this branch. Ask the user when you cannot read it. Use 0 only when they confirm there is none.
- `status` Draft.

**CONTRACT_PORT**

- `id` `<contractId>#<originCh>#<portId>`, `contractId`, `portId` and its name, the country.
- `originCh` Y for the loading port, which is required. N for the discharge port.

**CONTRACT_DOCUMENT**, one per document the contract requires

- `id` `<contractId>#<docTypeId>`, `contractId`, `docTypeId` from CONTRACT_DOCUMENT_TYPE.
- Ask the user which documents this contract needs.

If a step fails, stop. Report what was created and what was not. Delete the records you created only when the user agrees, child records first (documents, ports, positions) and the contract last. A record that other records point to is refused, and all of them are still drafts, so `delete_entity` accepts them in that order.

## 6. Report

Give the contract id and what was created. Name what the web application would have added and you did not: default cost estimates, standard clauses, default documents.

## 7. Send it for approval, only when the user asks

1. `validate_contract_for_approval` with the contract id. It checks the stored contract and its child records, so save everything first.
2. If `success` is false, show every entry of `errorDetails` (field and message), fix what the user confirms with `update_entity`, and validate again. `VALIDATION_UNAVAILABLE` means the check could not run: do not send, and tell the user.
3. When `success` is true, `update_entity` on the CONTRACT with `{"status": "Sent for Approval"}`. Only a Draft can be sent.
4. The workflow runs the same checks again. A few seconds later, read the contract with `get`. `Sent for Approval` means it waits for the approver. If the status went back, `approvalExplanation` says why.

## Changing a contract

- A Draft can be changed with `update_entity`. Show the old and the new value and get a yes.
- An approved contract is changed, cancelled, called back or closed through its workflows, which the user starts by asking for it. Read `entityStates` in the CONTRACT schema for the moves the workspace allows, confirm with the user, and set only a status that asks for the change. Approved, Cancelled, Closed and Archived are set by the workflows and are refused from you.
- A contract that is no longer a Draft cannot be deleted.
