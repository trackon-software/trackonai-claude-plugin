# Creating a shipment

A shipment moves quantity off a purchase contract line onto a sale line, or into a warehouse, and then runs its own steps. Three records and a state machine stand behind it, so two tools do the whole job.

| Tool | What it does |
|---|---|
| `check_shipment` | Works out the whole shipment and reports it: the plan, the defaults and every problem. Writes nothing, so run it first, every time |
| `create_shipment` | Links the two lines and creates the shipment from that linking. Writes everything |

Never build a shipment with `create_entity`. A LINKING or JOURNEY record written that way has no linked quantity, no copied contract documents and no team grants. Its flow still starts, because that follows from the record being written, so what you get is a shipment running with nothing linked to it. `create_entity` refuses those three records, so there is no second way to make a shipment: if the two tools above are missing or answer an error, say so and stop.

**Both contracts have to be approved first.** Approval is what sets a contract's POSITION rows to Open, and only an Open line can be shipped. You cannot approve one: `check_shipment` reports an unapproved line as `CONTRACT_NOT_APPROVED`, and the user approves it in the web application or from the approval email. See `contracts.md`.

## The steps

1. **Pick the two ends.** `list_entities` POSITION filtered by the contract id in `sourceId`, or `search`. The purchase line is the source and the **sale** line is the destination: a purchase line is not a destination. A line's id is `{contractId}#{productId}`, and `availableQuantity` is how much of it is left to ship. Pass them as `sourcePositionId` and `destinationPositionId`.
   - To ship out of or into a warehouse instead, pass the SUB_DEPARTMENT id as `sourceWarehouseId` or `destinationWarehouseId`. Those are arguments of their own: a warehouse id never goes in a position argument. The warehouse must sit under a DEPARTMENT of type `COWA`, the only kind the shipment screen offers, and one end still has to be a contract line.
2. **Pick both ports.** Both are required: the load port as `loadingPortId` and the discharge port as `destinationPortId`. Both are checked against the ports of ONE contract, the same list the screen offers. That is the sale line's contract when the destination is a sale line, and the source purchase line's contract when the goods go into a warehouse. `get` that contract with `includeDetails` true: when it carries its own PORT rows, those are the only ones allowed, with `originCh` Y on the load side and N on the discharge side. When it carries none, any DISPATCH_POINT goes (`search`, or `list_entities` DISPATCH_POINT).
   - The load port is not cosmetic. The contract's load side price offset is matched against it when the cargo is costed, so a shipment created with the wrong one is costed wrongly and nothing warns anybody.
   - A port that is not decided yet is TBA, never blank. TBA is not one port: there is one per country, about 195 of them, all named TBA and told apart by `countryId`. Take the TBA from that contract's own PORT rows, or the one whose country fits the leg. Never the first one a name search returns.
3. **`check_shipment`.** It writes nothing, so run it without asking. Show the user `plan` (both resolved ends, the quantity and its unit, the dates, the ports, the in-transit warehouse) and `defaults` (what the system filled in). Every entry of `problems` carries a code, the input field and a sentence. Nothing was written, so every one of them is worth fixing: correct them and check again, and do not go on while `problems` is not empty.
   - A problem about one of the two ends names the end, under `source.` or `destination.` (`source.positionId`, `destination.warehouseId`, `destination.type`), rather than the argument you sent.
4. **`create_shipment`, once the user has said yes.** The same arguments, plus a `clientReference` you make up for this one shipment. Report `shipmentNo` and `shipmentId`. The shipment's own flow starts by itself from the record: there is nothing further to call and nothing to start by hand.

**Ask the user for `quantity`, the dates (`shipmentBegin` and `shipmentEnd`, both as YYYY-MM-DD) and both ports.** The rest has defaults worth keeping: `uom` comes from the source line and `subDepartmentId` from the source contract. A warehouse has no unit of its own, so a warehouse source needs `uom` passed.

## The reference is what stops two shipments

`clientReference` is required, one per shipment, and never reused for a different one. Sending the same reference again creates nothing and hands back the first call's answer with `repeated` true, which says that nothing new was written. It is the only thing standing between a slow run and two shipments, so keep the reference for as long as the conversation may still ask about that shipment.

## A timeout is not a failure

You are given up on after about 25 seconds, and a run can take longer than that. The work carries on at the other end, so the shipment is often created. An answer saying the outcome is not known is that, and so is an error carrying no answer at all.

Never tell the user it failed, and never start over with a new reference. Wait about 30 seconds and send the same call again, with the same `clientReference` and the same arguments, field for field.

- Once the first run has finished, that returns its answer and writes nothing.
- While it is still running, it is refused. That refusal is the protection working, not a fault.
- Only that answer says what happened, and the shipment number you report comes from it.

Spelling out a field that the first call left to a default makes it a different request. The answer then says how far the first call got and names the ids it recorded. Read the first call's answer by sending its reference with exactly the fields it used, or `get` the JOURNEY by the id named there. Neither is a reason to send the shipment again under a new reference.

## Refusals that no argument gets around

| Code | What to tell the user |
|---|---|
| `DISPATCH_WINDOWS_APART`, `CARRIAGE_TYPE_MISMATCH` | Two lines can only ship together when their contracts overlap in time and agree on carriage. The screen never offers such a pair either. The contracts themselves have to change. Expect this one often |
| `LOT_IN_TRANSIT`, `LOT_ON_QUALITY_HOLD` | Goods already on their way cannot be shipped again, and a held lot is released on the quality screens first. The screen offers neither, so neither does this |
| `INTER_BRANCH_NOT_SUPPORTED` | The two sides sit on different branches, which needs a transfer price and a second warehouse. That shipment is created on the Linking screen in the web application |
| `BOTH_SIDES_WAREHOUSE` | One end has to be a contract line |

**The in-transit warehouse often has no default.** It comes from a company parameter per branch and many workspaces have none set. `check_shipment` then answers `IN_TRANSIT_WAREHOUSE_MISSING`, and it will answer that every time until a warehouse is passed. Find the candidates: `list_entities` DEPARTMENT and keep those whose `departmentType` is `ITWA`, then `list_entities` SUB_DEPARTMENT and keep those whose `upId` is one of them and whose `branchId` matches the source line's branch, which `check_shipment` reported as `source.branchId`. Show the user the list and pass the one they pick as `inTransitWarehouseId`. Do not invent one: the wrong department type or the wrong branch is refused as `WAREHOUSE_NOT_IN_TRANSIT` or `WAREHOUSE_WRONG_BRANCH`.

**Sentences that begin "Also:" are not refusals.** A message can end with them. A shipment whose dates fall outside the contract's dispatch window is allowed, exactly as it is on the screen, and a check that a failed read could not make is reported the same way. Show them to the user and carry on.

## When something was already written

Four codes mean the shipment exists, or may exist. None of them is a fixable input error, and in none of them may you call `create_shipment` again. Report the message, the ids and `leftovers` to the user, and let them look on the Shipments screen, or on the Linking screen when the message names that one.

| Code | What it means |
|---|---|
| `SHIPMENT_INCOMPLETE` | A run that did not finish. `shipmentId` filled means the record was read back and the shipment is there. Empty means no shipment was found under the number that run took: the number is taken before the shipment is written, so it is no proof of one. The message says what became of the linking and what to do with it |
| `ROLLBACK_INCOMPLETE` | `leftovers` names the records that are still standing. They need a person |
| `SHIPMENT_UNCONFIRMED` | A shipment number was taken and the record could not be read back, so nobody knows whether the shipment is there. Nothing was undone on a guess |
| `SHIPMENT_UNKNOWN` | The earlier run holding this reference was lost before it recorded anything at all |

The one exception is a message that says nothing of that run is standing. Then, once the user agrees, the shipment can be sent again under a NEW `clientReference`.

Only problems found with nothing written, which is what `check_shipment` returns, are worth correcting and sending again.
