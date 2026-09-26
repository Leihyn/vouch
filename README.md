# Vouch

**Permission to spend, without the party granting it learning what for.**

Before an agent is allowed to spend, something usually has to approve it: a credit check, an
allowlist, a compliance desk. Every one of those learns what the agent is doing, because the
approval and the job travel together. They do not have to.

Live: https://vouch-ten-henna.vercel.app

## The mechanism

A macaroon's signature is an HMAC chain, so appending a caveat needs only the current signature.
A **third-party caveat** exploits that to split approval from disclosure.

The service picks a fresh key `cK`, seals `(cK, predicate)` under a key it shares with the
voucher, and appends the sealed blob as a caveat. The agent holding the credential cannot read
it, cannot edit it without breaking the chain, and cannot satisfy it alone.

To spend, the agent takes the sealed blob to the voucher. The voucher opens it, sees a question
and a key, answers the question, and mints a **discharge macaroon** rooted at `cK`. That
discharge is then *bound* to the credential it was issued against:

```
bound = HMAC(root_macaroon.sig, discharge.sig)
```

The service verifies by replaying its own chain, opening the sealed caveat with the key it
already had, checking the discharge under `cK`, and checking the binding. It never contacts the
voucher.

Three consequences, all demonstrated on the page:

- **The voucher never learns the job.** It sees a predicate and a key, and nothing else — not the
  job id, not the budget, not the service's root key, not which service asked.
- **A discharge cannot be recycled.** Binding ties it to one credential's signature, so attaching
  it to another job fails even though that job's own chain is valid.
- **A discharge cannot be forged.** It must verify under the key that was sealed for the voucher,
  which the forger never had. Binding alone is not the check — the page forges a discharge that
  *does* bind correctly, and it is still refused.

## Running it

Static. No build, no server, no node, no wallet, and no dependencies at all.

```
python3 -m http.server 8000
```

Then open `http://localhost:8000`. Append `?demo` to run the whole flow automatically.

## What is implemented

Macaroon minting and verification over an HMAC-SHA256 chain, third-party caveats with an AES-GCM
sealed caveat identifier, discharge issuance, and cryptographic binding of a discharge to one
root credential.

Not implemented: payment, chained third-party caveats where a discharge itself carries one,
caveat expiry, and key distribution between service and voucher, which is assumed to have already
happened. This proves the authorization mechanism; it moves no money and contacts no network.

## Verification

The page's own script is loaded into Node behind a DOM shim, so the tests drive the shipped code
rather than a reimplementation. 23 assertions, including the three that matter:

```
PASS  the voucher does not see the job identifier
PASS  reusing the discharge elsewhere is refused
PASS  and the reason is the binding
PASS  a forged discharge is refused
PASS  and the reason is the signature, not the binding
PASS  the forgery did bind correctly, proving binding alone is insufficient
```

That last one is the interesting case: the two abuses are caught by two different checks, so
neither check is redundant.

## Dependencies

None. WebCrypto provides HMAC-SHA256 and AES-GCM. No framework, no build step, no vendored
library.

## Licence

MIT.
