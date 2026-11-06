# SourceCheck

Does this page really say that? SourceCheck lets anyone submit a claim and a source link. GenLayer validators read the live page, each ask a model, and must agree on one verdict: **SUPPORTED**, **CONTRADICTED** or **NOT_FOUND**. The verdict is stored on-chain together with a quote copied from the page, and any contract can read it.

Claims that cite a source are everywhere (news, docs, reports, social posts). A normal smart contract cannot read the web, and a single AI answer is not trustworthy. SourceCheck turns "the page says so" into a verdict that many independent validators had to agree on, with a quote anyone can verify.

- **Live app:** https://obitotobi1.github.io/sourcecheck/
- **Contract (GenLayer Studio):** `0xab7F99bBfC2F25eDdfAfe2ad26D866075CD1e9e4`
- **Contract source:** [`sourcecheck.py`](./sourcecheck.py)
- **Tests:** [`tests/test_sourcecheck.py`](./tests/test_sourcecheck.py)

## How it works

```
submit(claim, url) --> PENDING --run--> CHECKED (verdict + quote)
                          |
                          +--> UNREACHABLE (retry, up to 3 attempts)
```

1. `submit(claim, url)` stores a check (public https page only).
2. `run(check_id)` makes every validator fetch the page (`gl.nondet.web.render`, first 6,000 characters), ask its model, and reach consensus with `gl.eq_principle.prompt_comparative`: the `verdict` must be identical, the quote wording may differ.
3. `is_supported(owner, claim, url)` lets anyone, including other contracts, read the result.

## What it protects against

| Risk | How SourceCheck handles it |
|---|---|
| The model invents a quote | `SUPPORTED` and `CONTRADICTED` are only kept if the quote really appears in the fetched page. Otherwise the verdict becomes `NOT_FOUND`. |
| Prompt injection from the claim or the page | Each is fenced in its own tag with `<` and `>` escaped, and the prompt says to treat both as untrusted data. |
| Validators fetching internal addresses | Only `https` links to public hosts. `localhost`, private and link-local IPs, `.local`/`.internal`, credentials and IPv6 literals are refused. |
| Garbled model output | Anything that is not a valid verdict becomes `NOT_FOUND`. |
| Spam and retries | Only the owner can run a check, a verdict is final, up to 3 attempts for unreachable pages, 30 checks per wallet, duplicate claim + URL refused. |
| Huge pages | Only the first 6,000 characters are read. |

## Read API (composable)

- `get_check(check_id)` returns the full record as JSON (claim, url, status, verdict, quote)
- `get_checks_by_owner(owner)` returns that wallet's checks
- `is_supported(owner, claim, url)` is `true` only for a stored `SUPPORTED` verdict

## Live demo

The wallet `0xb2B565655Dd79A3b8aF76e846C22161bC47D044D` checked the claim "ProofLink proves that a wallet controls a public web page" against `https://raw.githubusercontent.com/Obitotobi1/prooflink/main/README.md`. The validators agreed on `SUPPORTED` with the quote "Prove that a wallet controls a public web page, with GenLayer validators as the witnesses." Open the app's **Look up a verdict** tab (it loads this example).

## Try it alone in two minutes

The app creates two demo accounts in your browser (no wallet, free on Studio).

1. Open the app, go to **New check**, enter a claim and a public https page that backs it up, and submit.
2. Open **My checks**, select the check and click **Run check**. Wait for consensus (a few minutes).
3. Read the verdict and the quote. Try a false claim on the same page to see `NOT_FOUND` or `CONTRADICTED`.

Every write shows the transaction lifecycle: signed, submitted, accepted by consensus, finalized.

## Tests

`tests/test_sourcecheck.py` runs the contract's real logic against a small stand-in for the `genlayer` library, with the web fetch replaced by a dictionary of pages and the model by a scripted reply. No node is needed:

```
python3 tests/test_sourcecheck.py
```

It covers URL refusal, owner-only runs, invented quotes being downgraded, garbage model output, unreachable pages and the attempt cap, prompt fencing, page truncation and the read API (11 tests).

## Limits

- It checks what a page says, not whether the page is true.
- Pages that need a login or are mostly images are not supported.
- Studio test network only. No payments or enforcement.

## Run the app locally

It is a single static file. Serve the folder (for example `python3 -m http.server`) and open `http://localhost:8000`. The `genlayer-js` library loads from a CDN at runtime, and demo keys are kept in your browser's local storage for the Studio test network only.
