---
name: wazirx-postman-docs-sync
description: Add, update, or remove a request in this Postman collection, or audit existing requests against the official WazirX API docs (docs.wazirx.com) for drift. Use whenever the user asks to add/wire up/fix a WazirX API request in this collection, or to check the collection against the live API docs.
---

# WazirX Postman collection ↔ docs sync

This repo publishes the request definitions third parties (including SDK
generators) treat as ground truth for the WazirX API. The **source of
truth for this skill is the official API reference at
https://docs.wazirx.com/ — not this collection's existing contents.**
This is the opposite trust direction from SDK-side sync skills that treat
this collection as their source of truth; that only works if this
collection is itself kept correct against the real docs, which is what
this skill is for.

The two files that matter:
- `collections/wazirx_spot_api_v1.postman_collection.json` — main spec
- `collections/wazirx_spot_api_v1_tests.postman_collection.json` — edge cases/examples

`environments/wazirx_com_spot_api.postman_environment.json` is reference
only for env var meanings (`{{wazirx_api_domain}}`, `{{api_key}}`, etc.)
— don't edit it unless a request change genuinely requires a new env var.

## 0. Known past drift (calibration, not a shortcut)

The "Withdraw" request under "Crypto SAPIs" was found pointing at
`POST /sapi/v1/crypto/withdraws` (plural) with params in the query
string, while the live docs specify `POST /sapi/v1/crypto/withdraw`
(singular) with params in a url-encoded body, plus an `addressBookId`
(from a separate address-book endpoint not in the collection at all) and
a required `extra` field. Fixed in WazirX/wazirx-api-postman#7. This is
the shape of bug to expect elsewhere: **verb and pluralization don't
reliably tell you the real path or transport — always check the docs'
own curl example.**

## 1. Fetch the real spec from docs.wazirx.com

For the request(s) in scope, fetch the relevant section of
https://docs.wazirx.com/ (WebFetch, not memory — the API changes over
time and training knowledge goes stale). Read the docs' own example curl
command directly rather than inferring from the prose table alone; the
curl example is ground truth for:
- **HTTP method** and **exact path** (don't assume `GET` when the prose
  says "Get X" — check the code block's `GET`/`POST`/`DELETE`).
- **Where params actually go** — query string vs url-encoded body.
  Every doc example is explicit about this (`-d`/`--data-urlencode` means
  body; params appended to the URL means query string). Do not assume
  the HTTP verb implies the location.
- **Each param's type and required/optional status**, from the docs'
  parameter table (`STRING`/`INT`/`LONG`/`DECIMAL`/`ENUM`) — not from
  guessing based on the param name.
- **Headers required** (`X-Api-Key` for signed endpoints).

## 2. Compare against the existing collection entry

Find the corresponding request in the collection (by folder + name, or
by URL if renamed). Check method, path, query vs body param placement,
param names, and the pre-request script's signing target
(`pm.request.url.query.toObject(...)` vs `pm.request.body.urlencoded.toObject(...)`
— these must match wherever the params actually live, or signatures
generated from this collection will be wrong against the real API).

- **If it matches:** leave it untouched. Do not "improve" or reformat a
  correct request.
- **If it differs:** fix the request definition — url, method, query/body
  params (add/remove/rename, set correct `disabled` flags), and the
  pre-request script's signing target if transport changed.
- **If a documented endpoint has no request anywhere in either file:**
  add one, mirroring the closest existing sibling request's conventions
  (headers, pre-request script pattern, `protocolProfileBehavior`,
  response array left empty `[]`).
- **If a request in the collection has no corresponding docs entry:**
  do not delete it on that basis alone — it may be real but
  undocumented, or intentionally marked unsupported (see the existing
  "Binance Transfer (Not supported)" request for the established pattern
  of keeping-but-labeling rather than removing). Flag it in your summary
  instead of guessing.

## 3. When uncertain, don't guess

If the docs are ambiguous, silent on a param, or you can't confidently
map a collection request to a docs section, leave the existing request
as-is and note it in your final summary rather than making a change you
can't verify.

## 4. Verify

Both collection files must remain valid JSON:
```bash
python3 -m json.tool collections/wazirx_spot_api_v1.postman_collection.json > /dev/null
python3 -m json.tool collections/wazirx_spot_api_v1_tests.postman_collection.json > /dev/null
```
Fix any parse errors before finishing. There's no build/test suite in
this repo beyond that — the JSON validity check plus your own re-read of
each changed request against the docs is the verification step.

## 5. Preserve style

Keep existing JSON formatting (indentation, key ordering) as close to
untouched as possible outside of what you're actually changing — this
file is diffed by hand by maintainers and other consumers.

## 6. Output

Summarize: for every request changed, what was wrong and what docs
section confirmed the fix (link the doc heading, e.g. "Wallet Endpoints
> Crypto Withdraw (USER_DATA)"). List anything left alone due to
uncertainty, with why. This summary is meant to be usable directly as a
PR description.