# Domain Glossary

The business nouns of IDURAR, each tied to code. Knowing these is what lets you talk about the app precisely — and an interviewer can tell within a minute whether you understand the *domain* or just the framework.

## Core entities

| Term | Meaning | Code location | Notes |
| --- | --- | --- | --- |
| **Admin** | the only user type; a business operator | `backend/src/models/coreModels/Admin.js` | `role` enum is just `['owner']` — single-tenant, single-role |
| **AdminPassword** | credentials split from the user record | `coreModels/AdminPassword.js` | holds `password`, `salt`, `loggedSessions[]` — 1:1 with Admin |
| **Client** | a customer the business invoices | `appModels/Client.js` | `createdBy`, `assigned` refs to Admin |
| **Invoice** | a bill with line items and a running balance | `appModels/Invoice.js` | the central document; see fields below |
| **Payment** | money received against an invoice | `appModels/Payment.js` | references invoice + client; mutates invoice on write |
| **Setting** | a key-value config row | `coreModels/Setting.js` | currency format, language, `last_invoice_number` |
| **Upload** | a stored file record | `coreModels/Upload.js` | for admin/setting images |

## Invoice fields that confuse newcomers

| Field | What it actually is | Code |
| --- | --- | --- |
| **items** | embedded line items (name, qty, price, total) | `Invoice.js:51-95` — embedded array, not a separate collection |
| **subTotal / taxTotal / total** | denormalized money totals, computed server-side | `Invoice.js:100-111`, set in `invoiceController/create.js:40-43` |
| **credit** | **the amount paid so far** — a running total of payments | `Invoice.js:118-121`, updated via `$inc` in `paymentController/create.js:70` |
| **discount** | amount subtracted before "paid" is judged | `Invoice.js:122-125` |
| **paymentStatus** | derived enum: `unpaid` / `partially` / `paid` | `Invoice.js:132-136`; computed from `total - discount` vs `credit` |
| **status** | workflow enum: `draft`/`pending`/`sent`/`refunded`/`cancelled`/`on hold` | `Invoice.js:148-152` — **different from paymentStatus** |
| **payment** | array of Payment ids attached to this invoice | `Invoice.js:126-131` |
| **number / year** | human-facing invoice number, seeded from a setting | `Invoice.js:10-17`; number comes from client form (`InvoiceForm.jsx:40`) |

**The single most important distinction:** `status` (where the invoice is in its workflow) vs `paymentStatus` (how much has been paid). They're independent — a `sent` invoice can be `unpaid`, `partially`, or `paid`. Confusing them is the classic newcomer mistake.

## Money/consistency vocabulary (also interview vocabulary)

| Term | In this repo | Anchor |
| --- | --- | --- |
| **denormalization** | `credit`/`paymentStatus` cached on the invoice instead of aggregated from payments | `Invoice.js:118-136` |
| **invariant** | `credit === SUM(non-removed payments.amount)` — must always hold | maintained in `paymentController/{create,update,remove}.js` |
| **soft delete** | `removed:true` instead of physical deletion | `createCRUDController/remove.js:3` |
| **server-authoritative** | server recomputes totals, ignores client's | `invoiceController/create.js:30-43` |
| **autopopulate** | Mongoose auto-joins referenced docs | `Invoice.js:178`, `Payment.js:56` |

## Auth vocabulary

| Term | In this repo | Anchor |
| --- | --- | --- |
| **JWT** | signed token carrying the admin `id` | `authUser.js:15-21` |
| **loggedSessions** | server-side allow-list of valid tokens (enables revocation) | `AdminPassword.js:30-33`, checked at `isValidAuthToken.js:45-53` |
| **salt** | a random `shortid` prepended to the password before bcrypt | `authUser.js:5`, `setup.js:16-18` |

## Near-synonyms / potential confusion

- **"paid" vs "credit":** the UI column labeled "paid" (`pages/Invoice/index.jsx:60-73`) actually renders the `credit` field. Same concept, two names.
- **"delete" vs "remove":** the API route is `/delete`, but the operation sets `removed:true` (a soft delete). Nothing is deleted.
- **"entity" vs "model" vs "controller":** derived from one filename — `Invoice.js` → entity `invoice`, model `Invoice`, controller `invoiceController` (`models/utils/index.js:28-47`).
- **"setting" (singular row) vs "settings" (the loaded config object):** the collection stores individual key-value rows; the app loads them into one object (`middlewares/settings/loadSettings.js`).

## Drill

Without looking, explain the difference between `status` and `paymentStatus`, and say which field the invoice list's "paid" column reads. Then explain what `credit` means and why it's stored on the invoice rather than computed from payments. Self-grade — Strong: you connect `credit` to denormalization and can state the invariant that keeps it correct.
