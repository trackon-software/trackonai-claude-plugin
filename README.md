# TrackonAI plugin for Claude

TrackonAI is a commodity trading and risk management (CTRM) system. Trading companies run their contracts, shipments, stock, invoices and accounting in it.

This plugin lets Claude work in your TrackonAI workspace the way a trained colleague would. It contains two things:

- **The TrackonAI connector**, the same one that is listed in the Claude directory (`https://mcp.trackon.ai/mcp`). It gives Claude the tools to read and, when you are allowed to, change your trading data.
- **The TrackonAI guide**, a skill that teaches Claude how the system works: which record is which, the order records are created in, how approvals work, and the step by step procedure for each common task.

## What you can ask for

- "Which shipments arrive this week?" "How much sesame is still open on our purchase contracts?" "Who owes us money?"
- "Run the AR/AP aging report for our main branch."
- "Draft a purchase contract for 500 tonnes of sunflower oil from this supplier and check whether it can be sent for approval."
- "Book this freight invoice against the shipment." "Book the lines of this bank statement."
- "Load these opening balances from our old system."
- "Generate the shipping instructions for this shipment."
- "We are a new company on TrackonAI. Help us set it up."

## What you need

- A TrackonAI workspace and a user account in it.
- Connector access, which your workspace administrator switches on per user. Reading and writing are separate permissions.
- Some workspaces use a connector address of their own. If yours was given one, keep using it. The guide in this plugin works with it too.

## How it handles your data

- The plugin contains text and configuration only. It has no scripts, no hooks and no code that runs on your computer.
- The connector acts as you. You sign in with your own TrackonAI account, and Claude can see and do only what that account may see and do in your own workspace.
- Every change passes the same validation and approval workflows as the TrackonAI web application. Claude can prepare a document and ask for its approval. It cannot approve, post to the ledger or delete a posted record.
- The guide tells Claude to show you every change and wait for your confirmation before writing anything.
- The plugin sends nothing to any service other than TrackonAI.

Privacy: https://trackon.com/mcp-privacy.html

## Help

- Setup and troubleshooting: https://trackon.com/mcp-documentation.html
- Support: support@trackonsoftware.com

## License

MIT. See `LICENSE`.
