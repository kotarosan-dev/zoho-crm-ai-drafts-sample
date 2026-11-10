# Zoho CRM × AI, with a human approval boundary

A Deluge prototype that adds AI summaries, classification, and email drafts to
Zoho CRM Leads — designed so the model **cannot write to your records**.

> Prototype note: this was built and verified in the author's own Zoho CRM
> organization with synthetic data only (November 2026). It is not client work.
> All numbers below are measured in that environment.

## The idea

Most "AI in CRM" demos let the model write straight into records. This sample
takes the opposite shape:

- The AI can only create records in a dedicated custom module, **AI Drafts**.
- A human reviews a draft and approves it. Approval freezes a snapshot of the
  approved content plus a fingerprint of the inputs it was generated from.
- A separate function — the only one allowed to write AI results into Leads —
  applies an approved draft into exactly four fields (`AI_Summary`,
  `AI_Summary_At`, `AI_Category`, `AI_Category_At`), and only after re-checking
  that neither the lead, its notes, nor the draft itself changed.
- Email drafts end at approval. There is no send function.

## Components

| Piece | What it does |
|---|---|
| Leads buttons: AI分類 / AI要約 / AIメール | Run `aiGenClassify` / `aiGenSummary` / `aiGenEmail` for one lead |
| AI Drafts buttons: AI承認 / AI反映 | Run `aiApprove` / `aiApply` for one draft |
| `aiGenClassify` | Closed-list classification (5 categories) with strict JSON validation; out-of-list values or wrong types fail the run instead of being stored |
| `aiGenSummary` | Summary + bullet points + next action from whitelisted fields and notes (each note truncated to 800 characters) |
| `aiGenEmail` | First-contact email draft (never sent) |
| `aiApprove` | Rebuilds the input fingerprint; mismatch → draft becomes **Stale**. On success, stores `Approved_Snapshot` |
| `aiApply` | Verifies the draft still matches its snapshot, then updates Leads with an `If-Unmodified-Since` precondition and reads back the result |

Design details that matter:

- **Whitelisted inputs.** Only named lead fields and notes are sent to the model.
  Email addresses, phone numbers, and postal addresses are never sent.
- **Generation is gated.** Only leads with `AI_Demo_Flag` on can generate, and an
  org-wide daily cap (100 generations) is enforced by scanning the draft ledger.
- **Every run is a ledger record** — including failures, with provider, model,
  prompt version, token usage, and timestamps.
- **Note edits are covered.** Zoho does not update a lead's `Modified_Time` when
  a note is edited (measured). The fingerprint therefore stores each note's ID
  and modified time, not just the lead's timestamp.
- **Regeneration supersedes.** Generating again marks the previous draft
  `Superseded`; only the newest generation can be approved.

## Setup

1. **Fields and module.** Add the five Leads fields and create the `AI_Drafts`
   custom module with the 24 fields in [docs/fields.md](docs/fields.md).
2. **Connections.** Create two Zoho Connections and either name them as below or
   find-and-replace the names in the code:
   - `your_crm_connection` — a Zoho CRM connection with scopes
     `ZohoCRM.modules.ALL`, `ZohoCRM.settings.ALL`,
     `zohosearch.securesearch.READ`, `ZohoCRM.coql.READ`
   - `your_openai_connection` — a custom connection to the OpenAI API with your
     own API key (BYOK). The prototype used `gpt-4o-mini`.
3. **API domain.** The code uses `https://www.zohoapis.com/...`. Replace the
   domain with your data centre's (`zohoapis.eu`, `zohoapis.in`, `zohoapis.jp`,
   …) if it differs.
4. **Functions.** Create the five functions from [deluge/](deluge/) as button
   functions. When wiring a button, pick the record-ID argument from the button
   form's field picker — typing a token by hand passes an encoded string, not
   the ID.
5. **Buttons.** Leads: three buttons calling the `aiGen*` functions.
   AI Drafts: two buttons calling `aiApprove` and `aiApply`.
6. **Try it.** Flag a test lead with `AI_Demo_Flag`, generate, approve, apply.

Two implementation notes learned the hard way:

- `invokeurl` calls to external APIs need an explicit
  `Content-Type: application/json` header. Zoho's own API tolerates its absence;
  OpenAI does not, and the error message points at the wrong thing.
- A Deluge string literal `"\n"` is **not** a newline — it stays two characters.
  `aiGenSummary` normalizes this by decoding a real newline from JSON and
  rejoining the text before storing it.

## Verified results (synthetic fixtures, 2026-11-10)

- Classification matched the expected category on **12 of 12** fixture leads,
  including an empty lead (classified "other" with Low confidence) and a lead
  with a ~4,700-character note (correct after truncation).
- **Prompt injection:** two fixtures carry instruction text planted in the lead
  data (one in the description, one in the company name). Both were classified
  from their actual content; the planted instructions appeared in neither the
  output nor the recorded reason.
- **Staleness:** editing the lead after generation, or editing only a note
  (parent `Modified_Time` unchanged), both made approval refuse and mark the
  draft Stale.
- **Approval integrity:** approving a draft and then editing the draft made
  apply refuse (snapshot mismatch). A draft hand-created in the UI with Status
  set directly to Approved was also refused — approval has to be a state the
  system reached, not a value someone typed.
- **Email terminal state:** an approved email draft returns "not a reflect
  target" from the apply function and stays Approved; nothing is sent.
- **Out-of-scope leads:** pressing generate on a lead without the flag returns a
  refusal and creates no draft.
- Generation took **4.8–9.1 seconds** per record end to end and used roughly
  190–910 tokens per run. Total API cost of the whole experiment was under
  US$1 (the prototype ran with a US$10 cap).

## Limitations — read before reusing

- **Synchronous generation is near its ceiling.** Zoho button functions have a
  10-second limit; the slowest measured run took 9.1 seconds as the draft
  ledger grew. One-record-at-a-time use in a small org works. Bulk processing
  needs an asynchronous queue (drafts marked Queued, processed by a schedule
  function). That design exists in the spec but is **not implemented** here.
- **Accuracy was measured on synthetic data written by the author.** 12/12 is a
  sanity check, not a production accuracy claim.
- **Email freshness revalidation is not implemented in v1.** The email
  generator records which summary it was based on, but approval does not
  re-validate that summary's freshness. Because email is never sent, the worst
  case is a stale draft.
- **Inputs are filtered at field level only.** Free-text content is sent as
  data; content-level scanning is out of scope.
- The category labels, prompts, and email signature are written for a Japanese
  Zoho consultancy context. Localize them together with the closed list in
  `aiGenClassify`.

## Files

- `deluge/` — the five Deluge functions (sanitized: connection names and API
  domain are placeholders, no org or record IDs)
- `docs/fields.md` — every field to create, with API names and types
- `docs/fixtures.md` — the 12 synthetic test leads and their expected results
