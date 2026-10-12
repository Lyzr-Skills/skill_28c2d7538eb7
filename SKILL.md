---
name: bucket-products-entitlements
description: Fact bucket for WHAT is sold and WHAT the customer may use - every distinct product on the order (name, SKU, type, unit, bundle and components), entitlement terms that modify standard product definitions (workspace size and allocation, use-case limitations, user-type allocations and exchange rights, usage limits, unlimited / enterprise licences) and promised deliverables outside the product lines. Joins to Pricing on product_key. Feeds SKU-based flags, performance-obligation (services) detection, MERA notes and the workspace, use-case, user-exchange and ELA flags.
metadata:
  bucket_id: products_entitlements
  schema_version: 2.0.0
  output_model: products_entitlements_models.py (ProductsEntitlementsOutput)
  brd_refs: Sec 2.2, 3.2, 6.2, 6.4.1, 6.5
  tracker_rows: 22, 23, 24, 29, 37, 38, 39, 43, 44, 45, 46, 49, 51, 53, 54, 55, 56
---

# Bucket: Products & Entitlements

This skill is self-contained: Part A is how to extract (shared by every bucket), Part B is what
this bucket extracts, Part C is the exact output structure.

# Part A - How to extract

You extract facts for **one bucket** (Products & Entitlements) from **one document**. Part B says which facts,
Part C gives the exact output structure, and the output is validated by `products_entitlements_models.py`
(`ProductsEntitlementsOutput`). Your output feeds three kinds of downstream checks:

| Output section | Downstream consumer | Your job |
|---|---|---|
| `values` | Fact rules in code (exact match, tolerance, date difference) | Transcribe the single value exactly + light parse |
| `records` | Calculation rules in code (sums, joins, period comparisons) | Transcribe every row of a repeating structure, completely, with stable keys |
| `evidence` | Contextual agent (policy overrides, deviations, judgements) | Capture verbatim clauses and signals; give non-binding hints only |

**Golden rule: you transcribe; code and the contextual agent decide.**

## A1. Input
- An OCR reconstruction of the PDF in **markdown**. Pages are separated by page markers
  `<<<PAGE n>>>`, where `n` is the 1-based physical page of the PDF.
- Markdown formatting in the input (`#` headings, `**bold**`, `_italic_`, `|` table cells, `-` bullets,
  `>` quotes, backticks, `<br>`, `\` escapes) is layout produced by the OCR step, not document text.

Use nothing else: not Salesforce, not Anaplan, not other documents, not general knowledge about the
customer. If a fact is not in this document, it is `not_found`.

## A2. Applicability
Read Part B section 1 and the document.
- Bucket does not apply -> `applicable: false`, a one-sentence `applicability_reason`, and
  `values`, `records`, `evidence` set to `null`. `observations` and `notes` are `[]`.
- Bucket applies -> `applicable: true` and fill **every** field Part B defines.

"Applies" means the document is the kind that could contain these facts, not that it does.
Silence in an applicable document is evidence (no opt-out clause, no budget cap), so it is reported
as `not_found`, never skipped.

## A3. Citations
Every `found`, `inferred` or `unreadable` fact carries `sources`, a list of
`{"page_number": int, "quote": [string, ...]}`. The quotes are used to **highlight the fact in the
original PDF**, so they must match the OCR text exactly.

**Rules for each string in `quote`:**
1. It is a verbatim span from **one single line** of the page. Never join two lines in one string.
2. It contains **plain text only**: no markdown syntax (`#`, `**`, `__`, backticks, `>`, leading
   `-` / `*` / `+` bullets, `<br>`, HTML tags, `\` escapes) and **never a `|` character**. Drop the
   formatting characters and keep the words between them: `**Order Schedule Number:** ACM-1`
   is quoted as `Order Schedule Number: ACM-1`.
3. Copy the characters exactly as the OCR text has them, including OCR typos, casing, punctuation,
   currency symbols and number formatting. No ellipses, no paraphrase, no translation.
4. Make it long enough to identify the fact unambiguously on the page (label + value, or the
   sentence that states it), but no longer than the line.

**When a fact spans several lines or cells, use one string per line or cell, in reading order:**
- Address block on 5 lines -> 5 strings.
- Label on one line, value on the next -> 2 strings.
- Table row -> one string per relevant cell (`| Solution Architect | 200 |` ->
  `["Solution Architect", "200"]`).
- A printed value that itself contains `|` (e.g. DocuSign `10-Apr-26 | 5:35:23 PM EDT`) -> split at
  the pipe: `["10-Apr-26", "5:35:23 PM EDT"]`. The `raw` field still keeps the full printed value.
- A fact on two pages -> two source objects, one per page.

**By status:**
- `inferred`: quote the text you inferred from. Never invent text for something that is absent.
- `unreadable`: quote the partially legible text as the OCR gives it; if nothing is legible, use
  `["[illegible]"]` with the page number.
- `not_found` / `not_applicable`: `sources` is `[]`.

## A4. Status and confidence
| Status | Use when | raw / parsed |
|---|---|---|
| `found` | Stated in the document | raw = as printed (formatting removed); parsed = light parse |
| `inferred` | Not stated directly but follows from specific text | raw = the text relied on; parsed = your reading; confidence at most `medium` |
| `not_found` | Bucket applies, document is silent | both null, no sources, no candidates |
| `not_applicable` | This field cannot exist in this document type | both null, no sources, no candidates |
| `unreadable` | Text is there but illegible | raw = what is legible; parsed = null |

`confidence`: `high` = explicit and unambiguous; `medium` = explicit but wording or formatting
leaves some doubt, or inferred; `low` = conflicting or barely legible.

## A5. Light parsing
`raw` is the truth (as printed, markdown removed); `parsed` is a convenience that code re-checks.
Put `parsed: null` rather than guessing.

| Type in Part C | parsed format | Rules |
|---|---|---|
| Text | string | Trim whitespace and line breaks; do not rewrite. |
| Number | number | Remove thousands separators; respect decimal commas (`1.234,56` -> 1234.56). Percentages in percent units (`99.5%` -> 99.5). |
| Boolean | true / false | Only from explicit wording; otherwise null. |
| Date | `YYYY-MM-DD` | Only if unambiguous. `03/04/2026` with no locale evidence -> null + `normalization` note. |
| Time | `HH:MM:SS` (24 h) | Seconds `00` if not printed. |
| Timestamp | `YYYY-MM-DDTHH:MM:SS` | Local time as printed; **no conversion, no offset**. |
| UtcOffset | `+HH:MM` / `-HH:MM` | Only when printed as an offset or an unambiguous zone listed in Part B. |
| Money | `{"amount": number, "currency_iso": "USD"}` | ISO 4217. `$` alone is ambiguous -> `currency_iso: null` unless the document names the currency. |
| CountryCode | ISO 3166-1 alpha-2 (`US`, `GB`) | |
| SubdivisionCode | ISO 3166-2 (`US-IL`, `CA-ON`) | |
| LanguageCode | ISO 639-1 (`en`, `fr`) | |
| CurrencyCode | ISO 4217 (`USD`, `EUR`) | Only when the document names the currency or uses an unambiguous symbol (`€`, `£`). |
| enum (`"a" \| "b"`) | one listed value | If nothing fits, use `other` where listed and keep the wording in raw. Never force a fit. |

**Never compute.** No totals, sums, differences, term lengths, execution dates, discounts, FX or
time-zone conversions. If the document prints a total, transcribe it. Code does the arithmetic.

## A6. Conflicts -> candidates
If the document states the same fact differently in two places, put the reading you judge primary
in the value, every other reading in `candidates` (each with its own sources), set
`confidence: low`, and add a `conflict` note. Do not reconcile.

## A7. Records (repeating structures)
- One item per row / entry **as printed**. Do not merge, split, de-duplicate or re-order.
- `key`: follow the key rule in Part B. Keys must be unique within the collection.
- Leaves are `{"raw", "parsed"}`; `status` and `sources` sit on the item (quote the row's line or cells).
- A column that is blank for a row -> `{"raw": null, "parsed": null}`.
- `completeness`: `items_extracted` = number of items you output; `items_declared` = a count the
  document itself prints, else null; `continues_beyond_extraction` = true if the table visibly
  continues somewhere not in this document; `status` = `complete` / `partial` / `unknown`.
- `declared_totals`: every total / subtotal printed for the collection, transcribed.
- Empty collection: `items: []`, `items_extracted: 0`, status `complete`.

## A8. Evidence (for the contextual agent)
**Clauses**: capture each clause matching one of Part B's clause categories.
- `clause_id`: `C1`, `C2`, … in document order.
- `text_verbatim`: the clause text with markdown formatting removed and line breaks kept.
- Length: full text for categories marked ★ (full-verbatim). Otherwise up to 1,500 characters; if
  cut, set `text_truncated: true` and stop at a sentence end.
- `sources`: one string per line of the clause (and one source object per page).
- `overrides_hint`: fill it when the clause says it modifies, replaces, supplements or conflicts with
  another document or clause; otherwise null.

**Signals**: short factual pointers explaining why something may matter.
- `signal_id`: `S1`, `S2`, … Link `clause_id` when the signal comes from a captured clause.
- Record weak signals too. `description` states what is printed, not a conclusion.

**Hints**: one entry for every hint Part B defines, every time the bucket applies.
- `likely` must cite at least one `signal_id`; `unlikely` / `unclear` may have an empty list.
- Hints are leanings, not verdicts.

## A9. Observations and notes
- `observations`: useful facts this bucket does not define but that matter to contract review
  (cited). Do not repeat facts already in a field.
- `notes`: `conflict`, `normalization`, `missing` (expected but absent), `multi_document`,
  `truncation`, `other`, with `field_path` when about a specific field.

## A10. Fact paths
Downstream checks, agent findings and human overrides address facts by path:
`products_entitlements.values.<field>`, `products_entitlements.records.<collection>[<key>].<leaf>`,
`products_entitlements.evidence.clauses[<clause_id>]`, `products_entitlements.evidence.signals[<signal_id>]`.

## A11. What the validator checks
`products_entitlements_models.py` rejects the output if any of these fail; it is retried once with the errors,
then routed to a human.
1. Structure, field names, types and enum values match Part C exactly (no extra fields).
2. Every quote string is plain text (no `|`, no markdown syntax, no line break) and is found on a
   single line or table cell of its page.
3. Status rules: `found` / `inferred` / `unreadable` have sources; `not_found` / `not_applicable` have
   none and null raw / parsed; `inferred` is never `high` confidence.
4. `values` / `records` / `evidence` are null exactly when `applicable` is false.
5. `items_extracted` equals the number of items; record keys are unique; every item has a source.
6. Clause and signal IDs are unique and every reference to them exists.
7. Every hint appears exactly once; `likely` hints cite a signal; ★ clauses are not truncated.

# Part B - Products & Entitlements

**Core rule: list what is sold and what modifies it; leave classification to code.** Most SKU flags
(Subscription Fee, PAF Upgrade, AI Training, Hyperconnect, BYOK, Optimizer, Anaplan Apps, CPX,
HyperCare, AnaplanHelps, Summit Success) are set by code by looking up `sku` / `product_name` in the
product mapping (SYS07). The model's job is to transcribe every product exactly, so the lookup works.

## 1. Applicability
**Applies** to any document that lists products or services sold, or changes what a product gives:
OS, SOW (services lines), change order, amendment, quote, marketplace private offer, PO with line
items, and an SSA / customer paper that defines entitlements (e.g. workspace or use-case terms).

**Does not apply** to documents with no products and no entitlement terms (e.g. a Deal Desk email,
a DPA, an SLA exhibit).

## 2. Ownership boundaries
| Fact | Owner bucket | Join |
|---|---|---|
| Line dates, quantities per period, prices, totals | Pricing | `product_key` |
| How services are delivered, accepted, billed (milestones, T&M, training delivery) | Services Delivery | services products appear here as products; delivery mechanics there |
| Renewal price caps, future discounts, options to buy more at a fixed price | Special Terms | |
| Affiliates as organisations | Parties & Geography | the **right** of affiliates to use the service is an entitlement here (`affiliate_use`) |

## 3. product_key (shared with Pricing)
Every product has a `product_key`, built with the same rule in this bucket and in Pricing:
- SKU printed -> the SKU upper-cased with spaces removed: `ANP-EPU`.
- No SKU -> the product name lower-cased, every run of non-alphanumeric characters replaced by `_`,
  trimmed: `Anaplan Enterprise Planner User` -> `anaplan_enterprise_planner_user`.
- Use the product name exactly as it appears in the order-line table, so both buckets build the same key.

## 4. values (fact rules)
| Field | Type | Extraction notes |
|---|---|---|
| `product_table_present` | Boolean | true if the document has a product table or product list; false (found) if it clearly has none (quote the title or a heading). |
| `unlimited_usage_stated` | Boolean | true only if the text grants unlimited users / usage, an enterprise-wide licence, or quantity "Unlimited". Silence = `not_found`. |

## 5. records (calculation rules)

### products: one item per distinct product (not per period)
| Leaf | Type | Notes |
|---|---|---|
| `product_name` | Text | Exactly as printed in the order-line table (OCR typos kept). |
| `sku` | Text | Product code as printed, if any. |
| `item_type` | enum `subscription`, `capacity`, `platform_addon`, `connector`, `consumption`, `professional_services`, `training`, `support`, `bundle`, `fee_placeholder`, `other` | From the name / description / section. `capacity` = workspace / data volume; `connector` = HyperConnect / integration connectors; `fee_placeholder` = a generic "Subscription Fee" or similar line with no product identity; `professional_services` includes Delivery Assurance and AI Training lines on an OS. Unsure -> `other`. |
| `unit_of_measure` | Text | "Users", "GB", "Workspaces", "Connectors", "Hours". |
| `is_bundle` | Boolean | The line is a bundle / suite / package of other products. |
| `bundle_parent_key` | Text | For a component listed under a bundle: the bundle's `product_key`. |
| `description_raw` | Text | Product description printed with the line, if any. |

The same product on several period rows -> **one** item here (the rows are in Pricing).

### entitlements: one item per term that adds to, limits or modifies a product
| Leaf | Type | Notes |
|---|---|---|
| `entitlement_type` | enum `workspace_size`, `workspace_count`, `workspace_allocation`, `user_type_allocation`, `user_exchange_right`, `use_case_limitation`, `usage_limit`, `environment`, `affiliate_use`, `other` | |
| `product_key` | Text | The product it modifies, if identifiable. |
| `quantity` / `unit` | Number / Text | e.g. 30 / GB, 10 / users. |
| `at_no_charge` | Boolean | Given free / "at no additional charge". |
| `description_raw` | Text | Short wording of the term. |

Key: `<entitlement_type>:<n>`. Record entitlements from product tables, product descriptions and special terms.

### deliverables: promised items that are NOT product lines
Free services, custom builds, onsite days, credits toward services, training promised in special
terms. Key `D<n>`. Leaves: `description_raw`, `deliverable_type` (`service`, `training`,
`custom_development`, `onsite_days`, `service_credit`, `other`), `quantity`, `unit`, `at_no_charge`.
A services product that is a priced line belongs in `products`, not here.

## 6. evidence (contextual agent)
**Clause categories** (★ = full-verbatim): `entitlement_definition`, `workspace_terms` ★,
`use_case_limitation` ★, `user_exchange` ★, `free_or_bundled_items` ★, `enterprise_license` ★,
`product_substitution`, `other_products`.

**Signal types**:
| Type | Record when |
|---|---|
| `unlimited_usage_language` | "unlimited", "enterprise-wide", "all employees", quantity "Unlimited" |
| `workspace_modification` | Extra / free workspace, custom size, workspace per user or business unit |
| `use_case_restriction` | Licences limited to named use cases, or separate entitlements per use case |
| `user_exchange_language` | Swap, exchange, convert, reallocate, substitute, true-down licences between types |
| `free_item` | Any product, entitlement or deliverable given at no charge |
| `bundle_presented` | A bundle line whose components are listed separately elsewhere, or a bundle with no components shown |
| `unrecognised_product_name` | A custom or generic product name (e.g. "Subscription Fee", "Platform Fee") that may stand for a placeholder SKU |
| `services_in_order` | A services, training or support line on a subscription order |
| `new_product` | Wording that the product is new to the customer or a new tier |
| `other` | Anything else a reviewer should see |

**Hints** (each exactly once):
| Hint | Lean on… | Feeds |
|---|---|---|
| `enterprise_license` | The order is an enterprise / unlimited licence | #51 ELA Agreement |
| `special_workspace_allocation` | Workspace terms go beyond the standard SKU definition | #39 Special Workspace Allocation, MERA note (Rule 6.4.1) |
| `multiple_use_case` | Licences are tied to or limited by named use cases | #55 Multiple Use Case, MERA note |
| `user_exchange_rights` | The customer may swap or convert licences between types | #49 User Exchange Rights |
| `separate_deliverable` | Anything other than subscription products is promised (services, training, deliverables) | #29 No other separate deliverable (likely -> 'N') |

## 7. Validations fed (rule registry)
| Rule | Type | Check | Fact paths |
|---|---|---|---|
| SKU flags #22-24, #37-38, #43-46, #54, #56 | fact | Look up each `sku` / `product_name` in SYS07; flag TRUE if any product maps to the category; compare with the tracker / SF value. Gate G1 (BOM match, Pricing) must pass first | `records.products[*].sku`, `.product_name` |
| #22 Subscription Fee variance | context | A placeholder SKU on the quote but a custom name on the OS at the same amount -> human confirmation, not a fail | signal `unrecognised_product_name` |
| #46 Anaplan Apps bundles | calc | Bundle on the OS vs components on the quote: match totals, flag for human | `records.products[is_bundle]`, `bundle_parent_key` |
| #29 No other separate deliverable (Rule 2.2 / 6.2) | fact | Only subscription / capacity / add-on products -> 'Y'; any services / training / support product or any deliverable -> 'N' + list | `records.products[*].item_type`, `records.deliverables`, hint `separate_deliverable` |
| #24 AI Training cross-check | fact | AI Training = TRUE -> #29 must be 'N' | products + #29 |
| #38 Hyperconnect treatment | fact | Connector products -> point-in-time, excluded from SSP allocation (BRD 3.2, 6.5) | `records.products[item_type=connector]` |
| BRD 3.2 Consumption SKUs | fact | Consumption products -> flag for review (recognise on usage) | `records.products[item_type=consumption]` |
| #39 Special Workspace Allocation | context | TRUE + MERA note (adjust list price for modified entitlements) | `records.entitlements[workspace_*]`, hint `special_workspace_allocation` |
| #55 Multiple Use Case | context | TRUE + use cases listed + MERA note | `records.entitlements[use_case_limitation]`, hint `multiple_use_case` |
| #49 User Exchange Rights | context | TRUE + clause | `records.entitlements[user_exchange_right]`, hint `user_exchange_rights` |
| #51 ELA Agreement | context | TRUE + clause (definition still open with the client) | `values.unlimited_usage_stated`, hint `enterprise_license` |
| #53 Additional Units | calc | Products already held under an active order for the account (document graph) -> TRUE | `records.products` + prior orders |
| Rule 6.3.1 New product tier | context | Net-new product tier -> Commissions / Bookings review | signal `new_product` |

# Part C - Output structure

Return **one JSON object** exactly in this shape. Every key shown is required (use `null` /
`[]` as the rules allow); no other keys are allowed.

## C1. Shared shapes

```jsonc
Source        = { "page_number": int >= 1, "quote": [string, ...] }        // >= 1 plain single-line string
Value<T>      = { "raw": string | null, "parsed": T | null,
                  "status": "found" | "not_found" | "not_applicable" | "inferred" | "unreadable",
                  "confidence": "high" | "medium" | "low",
                  "sources": [Source], "candidates": [Candidate] }
Candidate     = { "raw": string, "parsed": string | number | boolean | Money | null, "sources": [Source] }
Leaf<T>       = { "raw": string | null, "parsed": T | null }
Money         = { "amount": number | null, "currency_iso": string | null }   // ISO 4217, 3 letters
Item          = { "key": string, "status": "found" | "inferred" | "unreadable", "sources": [Source] (>= 1), ...leaves }
Collection<I> = { "completeness": Completeness, "declared_totals": [DeclaredTotal], "items": [I] }
Completeness  = { "items_extracted": int, "items_declared": int | null,
                  "continues_beyond_extraction": boolean, "status": "complete" | "partial" | "unknown" }
DeclaredTotal = { "label_raw": string, "value": Leaf<Money>, "sources": [Source] }
Clause        = { "clause_id": string, "clause_number_raw": string | null, "heading_raw": string | null,
                  "text_verbatim": string, "text_truncated": boolean, "category_hints": [category],
                  "overrides_hint": OverridesHint | null, "sources": [Source] }
OverridesHint = { "target_document_raw": string | null, "target_clause_raw": string | null,
                  "relationship": "modifies" | "replaces" | "supplements" | "conflicts" | "unclear" }
Signal        = { "signal_id": string, "signal_type": signal_type, "description": string, "entity_raw": string | null,
                  "strength": "strong" | "moderate" | "weak", "clause_id": string | null, "sources": [Source] }
Hint          = { "hint": hint_name, "value": "likely" | "unlikely" | "unclear", "signal_ids": [string] }
Observation   = { "label": string, "value_raw": string, "sources": [Source] }
Note          = { "type": "conflict" | "normalization" | "missing" | "multi_document" | "truncation" | "other",
                  "field_path": string | null, "message": string }

Scalar types: Text = string, Number = number, Boolean = boolean, Date = "YYYY-MM-DD",
Time = "HH:MM:SS", Timestamp = "YYYY-MM-DDTHH:MM:SS", UtcOffset = "+HH:MM" | "-HH:MM",
CountryCode = "US", SubdivisionCode = "CA-ON", LanguageCode = "en", CurrencyCode = "USD", Money as above.
```

## C2. `products_entitlements` output

```jsonc
{
  "bucket": "products_entitlements",
  "schema_version": "2.0.0",
  "applicable": true,                // false -> values, records, evidence are null
  "applicability_reason": string,

  "values": {
    "product_table_present": Value<Boolean>,
    "unlimited_usage_stated": Value<Boolean>
  },

  "records": {
    "products": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "product_name": Leaf<Text>,
      "sku": Leaf<Text>,
      "item_type": Leaf<"subscription" | "capacity" | "platform_addon" | "connector" | "consumption" | "professional_services" | "training" | "support" | "bundle" | "fee_placeholder" | "other">,
      "unit_of_measure": Leaf<Text>,
      "is_bundle": Leaf<Boolean>,
      "bundle_parent_key": Leaf<Text>,
      "description_raw": Leaf<Text>
    }>,
    "entitlements": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "entitlement_type": Leaf<"workspace_size" | "workspace_count" | "workspace_allocation" | "user_type_allocation" | "user_exchange_right" | "use_case_limitation" | "usage_limit" | "environment" | "affiliate_use" | "other">,
      "product_key": Leaf<Text>,
      "quantity": Leaf<Number>,
      "unit": Leaf<Text>,
      "at_no_charge": Leaf<Boolean>,
      "description_raw": Leaf<Text>
    }>,
    "deliverables": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "description_raw": Leaf<Text>,
      "deliverable_type": Leaf<"service" | "training" | "custom_development" | "onsite_days" | "service_credit" | "other">,
      "quantity": Leaf<Number>,
      "unit": Leaf<Text>,
      "at_no_charge": Leaf<Boolean>
    }>
  },

  "evidence": {
    "clauses": [Clause],
    "signals": [Signal],
    "hints": [Hint]
  },

  "observations": [Observation],
  "notes": [Note]
}
```

Allowed lists for this bucket:

- `category` (clause `category_hints`): `entitlement_definition`, `workspace_terms` ★, `use_case_limitation` ★, `user_exchange` ★, `free_or_bundled_items` ★, `enterprise_license` ★, `product_substitution`, `other_products`
  (★ = full-verbatim, never truncated)
- `signal_type`: `unlimited_usage_language`, `workspace_modification`, `use_case_restriction`, `user_exchange_language`, `free_item`, `bundle_presented`, `unrecognised_product_name`, `services_in_order`, `new_product`, `other`
- `hint_name` (each exactly once): `enterprise_license`, `special_workspace_allocation`, `multiple_use_case`, `user_exchange_rights`, `separate_deliverable`
