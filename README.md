# AI Financial Admin Automation

A six-workflow n8n system that turns a mailbox full of invoices, quotes, receipts and bank statements into a traceable financial record, where **no payment is marked paid until its supporting document is attached**.

Built and delivered for a client running a property construction project. The client is not named and no client data appears here. See [Why there is no workflow JSON in this repository](#why-there-is-no-workflow-json-in-this-repository).

**In production since June 2026.**

![System architecture](images/architecture.png)

## The problem

Financial administration on a construction project arrives as email. An invoice from a supplier, a quote to approve, a payment receipt, a bank statement, a contribution from one of the people funding the project. Each one has to be read, understood, recorded somewhere, and its document filed where someone can find it again six months later when a number is questioned.

Done by hand, two things fail. The recording is inconsistent, and the evidence drifts away from the record. By the time anyone asks "what is this payment for", the email is buried and the PDF is in a download folder.

## What the system does

```
Gmail (invoices, quotes, receipts, statements, contributions)
      |
      v
  Extract email context, including attachments
      |
      v
  Claude: structured extraction into typed records
      |
      v
  Route to the right table      ---------> Timeline (audit trail)
      |
      v
  Upsert the record             <--------- Documents sheet
      |                                      ^
      v                                      |
  Upload attachment to Drive  ---------------+
      |
      v
  Write the readable link back into the visible row
      |
      v
  Label the email as processed
```

Six workflows:

| Workflow | Role |
|---|---|
| **Email ingestion** | The live pipeline above. Gmail trigger, extraction, routing, upsert, document upload, `processed` label |
| **Historical backfill** | A one-shot pass over the existing mailbox to build the record retroactively. Frozen after its run, deliberately kept separate from the live pipeline |
| **Document linking** | Uploads attachments to Drive, records them in a hidden `Documents` sheet, and propagates readable links back into the rows the client actually looks at |
| **Daily payment alerts** | Scheduled check for overdue, due-soon and review-required items, pushed to Telegram |
| **Weekly summary** | Scheduled financial digest, pushed to Telegram |
| **Error handler** | Catches any failure in any of the above and alerts on a separate channel |

A Telegram assistant sits on top: an authorised user asks a question in plain language, the system fetches the relevant sheet context, Claude answers against it, and the answer carries links back to the source email and the source document.

## Results

![Operational results](images/results.png)

Measured on the delivered system:

| | |
|---|---|
| In production since | **June 2026**, with no intervention from me |
| Records linked to their source email | **46 / 46** |
| Payment records structured | **46** |
| Documents centralized | **213** |
| Contribution transactions verified | **92** |
| Production workflows delivered | **6** |
| Project funding tracked | **$2.0M+**, on the historical back-fill |
| Source emails processed during onboarding | **70+**, in one day |

**Honest framing.** These are onboarding-phase figures, not a running cumulative total. The time saved on that phase is a client estimate of around 17.5 hours, derived from their own estimate of handling time per email, not a measured post-deployment figure. The traceability figure applies to the anonymized public proof layer shown in the screenshots.

## The design decisions worth reading

**A record is not complete until its evidence is attached.** A payment stays `pending` while its supporting document is missing, even when the amount and the date are known. The status field answers "can I prove this", not "did I type this in". That one rule is what makes the record defensible six months later.

**Document links are owned by the nodes that write documents, and by nothing else.** An early version let the core upserts write every field on a row, and they silently wiped the document links each time a record was updated. The fix was ownership: the upsert owns the financial fields, the document nodes own the document fields, and the two never write to the same columns.

**A hidden table for machines, visible tables for humans.** Documents live in their own sheet with their own identifiers and their relation to the record they support. What the client sees is a readable link in the row they were already looking at. The structure serves the system, the surface serves the person.

**The backfill is a separate, frozen workflow.** Rebuilding history and processing today's mail are different problems with different failure modes. Keeping them apart means a change to the live pipeline can never corrupt the historical record, and the backfill can be re-read as evidence of what was imported and when.

**Errors alert on a different channel from business events.** A failed execution concerns the person who built the system. An overdue payment concerns the person running the project. They do not share an inbox.

**Error notifications are sent as plain text, on purpose.** Raw error messages routinely contain characters that break Telegram's Markdown parser, so the alert that tells you something is broken breaks too. Plain text is the reliable choice.

**The assistant checks who is asking before it answers.** A Telegram bot that reads a financial tracker is an access path. The sender is verified before any context is fetched.

**Routing uses phrase matching, not clever regular expressions.** Intent routing covers links, pending payments, contributions, operating expenses, invoices, payments, quotes, documents, timeline and summaries. Simple matching proved more maintainable, and it sidesteps a known parsing issue with regular expressions in n8n code nodes.

**Every production workflow declares the error workflow explicitly.** It is not inherited and it is not assumed. If it is not set, failures are silent.

## What the mailbox looks like going in

![Source email](images/source-email.png)

The screenshots in this repository come from the anonymized demonstration layer built for the public case study, not from the client's live account.

## Why there is no workflow JSON in this repository

The exported workflows carry client-specific content in prompts, node labels and routing conditions, including names. Publishing them with names replaced would require certifying that every occurrence was caught across six workflows and several hundred kilobytes, and one missed label in one condition would expose a client who trusted me with their project finances.

So this repository documents the architecture and the decisions, and keeps the code private. If you want to see production n8n code I wrote, that repository exists: [opspilot-ai-lead-intake](https://github.com/AntoineMoineaux/opspilot-ai-lead-intake), a lead intake and qualification system published in full, with its four workflows.

## License

MIT, for the documentation in this repository.
