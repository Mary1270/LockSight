# LockSight

Structured risk assessment of a project's tokenomics and vesting claims on GenLayer, with a readable on-chain registry and version-to-version drift tracking.

Someone registers the URL of a project's tokenomics page. GenLayer validators fetch it and locate five facts, each copied from the page as an exact quote: the team share, the cliff, the vesting period, the initial unlock and how team tokens are locked. The contract code checks every quote against the page, parses the numbers itself, and computes the risk level (LOW / MEDIUM / HIGH) with a fixed point table. No model ever writes the risk level. When the project publishes a new version of the page, `supersede` registers it, and once it is final the code records the drift: whether the terms got worse, better or stayed the same, and which fields moved.

## What the model does and what the code does

| Step | Done by |
|---|---|
| Check the URL is https, clean, and under an owner-approved prefix | code |
| Fetch the page | each validator |
| Locate the five facts and copy exact quotes | model |
| Drop every quote that is not on the page as whole words (so "5%" cannot be cut out of "25%"), or is longer than 200 characters; drop flags that have no quote | code |
| Parse percentages and durations from the quotes, check lock labels against the quote text | code |
| Bucket the numbers, add the points, assign LOW / MEDIUM / HIGH | code |
| Agree on the bands that drive the rules, the score and the risk level | validators (exact match) |
| Re-check that the leader's quotes occur on the validator's own copy of the page | each validator, in code |
| Rebuild the stored rating from the verified quotes (nothing else the leader sent is stored) | code |
| Check that a new version cites the page it replaces | code |
| Compute drift between two final ratings | code |
| Challenge, bonds, payouts, windows, accounting | code |

## Risk rules

| Field | Reading of the quote | Points |
|---|---|---|
| Team share | up to 20% | 0 |
| Team share | above 20% up to 30% | 1 |
| Team share | above 30% | 2 |
| Team share | not stated | 1 |
| Cliff | 90 days (3 months) or longer | 0 |
| Cliff | shorter than 90 days, or a cliff without a readable length | 1 |
| Cliff | explicitly no cliff | 2 |
| Cliff | not stated | 1 |
| Vesting | 360 days (12 months) or longer | 0 |
| Vesting | shorter than 360 days, or vesting without a readable length | 1 |
| Vesting | explicitly no vesting | 2 |
| Vesting | not stated | 1 |
| Initial unlock | up to 10% | 0 |
| Initial unlock | above 10% up to 20% | 1 |
| Initial unlock | above 20% | 2 |
| Initial unlock | not stated | 1 |
| Lock | smart contract, or third-party custodian / escrow | 0 |
| Lock | not mentioned | 1 |
| Lock | explicitly not locked | 2 |

Score 0-2 is LOW, 3-4 is MEDIUM, 5 or more is HIGH. At least three of the five fields must be grounded in the page; otherwise the result is UNDETERMINED. Durations use 30-day months and 365-day years. `get_rules()` returns the same table from the contract.

Conservative behavior: a quote that cannot be found on the page, a number that cannot be parsed, a flag that its quote does not support (a NO cliff or vesting flag needs a quote that negates that same term or says 0), or a lock label the quote does not support (a negated lock, a multisig presented as a contract, or self-custody presented as a third party) is treated as not stated, which costs a point instead of earning one.

## Lifecycle of one page

```
register_page -> REGISTERED
REGISTERED --assess--> RATED (challenge window opens)
REGISTERED --assess fails 3 times--> UNRATED
REGISTERED --stale, expire_item--> UNRATED
RATED --challenge (2 GEN bond)--> CHALLENGED
RATED --finalize after the window--> FINAL
CHALLENGED --resolve_challenge, risk level changes--> FINAL (challenger gets bond + assurance)
CHALLENGED --resolve_challenge, risk level holds or no citation--> FINAL (registrant gets bond + assurance)
CHALLENGED --resolve_challenge undetermined--> RATED, bond refunded, a fresh window opens
CHALLENGED --stale, cancel_challenge--> RATED, bond refunded, a fresh window opens
```

Every state has an exit that needs no particular person. A refunded challenge does not use up the challenge slot and does not close the window: the time that was left when it was filed is given back, so the clock stands still while a challenge is open. This is done at most twice per page; after that, time spent in a refunded challenge is no longer given back and the original deadline applies, so a challenger cannot hold an item open without limit.

## Supersede and drift

`supersede(item_id, url)` registers a newer version of a page whose rating is FINAL.

- The new URL must sit under the prefix stored on the previous version, which is fixed at its registration.
- The new page must cite the previous page's full URL. A page that does not is reported as UNLINKED and uses up an assessment attempt without any model call.
- Only one newer version can be in progress per page. If it ends UNRATED or times out, the slot is free again.
- Anyone can submit the new version, including an optional assurance. The new version is assessed, can be challenged and finalized like any other.

When the new version becomes FINAL, the contract stores the drift against the previous final rating:

| Field | Meaning |
|---|---|
| `direction` | WORSE if the score went up, BETTER if it went down, SAME if equal |
| `score_from`, `score_to`, `score_delta`, `magnitude` | the scores and their difference |
| `risk_from`, `risk_to`, `risk_step` | the risk levels and how many steps they moved (+2 LOW to HIGH, -1 HIGH to MEDIUM) |
| `changes` | every field whose band or number moved, with old and new band, value and points |

The previous version then reports `superseded_by`, and `get_risk` no longer serves it, so nobody reads a stale rating as current. `get_project_risk(project_id)` always returns the latest final rating of a project.

## Challenges

A challenge is only about the evidence. The counter-evidence page must cite the item's URL and sit under the prefix the item was registered under. That prefix is stored on the item at registration and never changes, so later prefix edits by the owner cannot block or widen a challenge. The original page is not extracted again: fields the counter page states (with a grounded, parseable quote) replace the same fields in the stored rating, the result is scored by the same rules, and the challenge wins only if the risk level changes.

The registrant may attach an optional assurance of 0 to 100 whole GEN when registering. It returns to the registrant at the end of the item's life unless a challenge changes the risk level, in which case the challenger receives it with their bond.

## Contract

`contracts/locksight.py` (single file, deploy as is). Constructor: `challenge_window_seconds`, `stale_after_seconds` (each 60 to 604800). Suggested values: `900, 1800` for testing and `172800, 259200` for production.

| Method | Who | Purpose |
|---|---|---|
| `add_prefix(prefix, label)` / `remove_prefix(prefix)` | owner | manage the https URL prefixes new pages may be registered under; existing items keep their stored prefix |
| `register_page(url)` payable, 0-100 whole GEN | anyone | register a tokenomics page; invalid input is refunded to the caller's balance, not reverted |
| `supersede(item_id, url)` payable, 0-100 whole GEN | anyone | register a newer version of a FINAL page |
| `assess(item_id)` | anyone | consensus extraction and rating |
| `challenge(item_id, counter_evidence_url)` payable, exactly 2 GEN | anyone but the registrant | dispute the rating |
| `resolve_challenge(item_id)` | anyone | consensus extraction of the counter evidence, merged with the stored rating |
| `cancel_challenge(item_id)` | anyone | refund a challenge that stayed unresolved past the stale limit |
| `finalize(item_id)` | anyone | close an unchallenged rating after the window |
| `expire_item(item_id)` | anyone | release an item stuck before its first rating |
| `withdraw()` | anyone | pull payment of the caller's balance |

Views: `get_risk(item_id)`, `meets_max_risk(item_id, max_risk)`, `get_drift(item_id)`, `get_project_risk(project_id)`, `get_risk_by_url(url)`, `get_item`, `get_project`, `find_item(url)`, `list_items(offset, limit)`, `list_prefixes`, `get_params`, `get_rules`, `get_stats`, `get_balance`, `get_accounting`.

`get_drift` returns a JSON object whose `direction` is one of BASELINE (first version), PENDING (new version not final yet), UNKNOWN (new version ended UNRATED), WORSE, BETTER, SAME or NOT_FOUND.

## Using the registry from another contract

Only FINAL, current ratings are exposed through `get_risk`, `get_risk_by_url`, `get_project_risk` and `meets_max_risk`. An empty string or `False` means "no current final rating".

```python
registry = gl.get_contract_at(Address("0xLOCKSIGHT_ADDRESS"))
if not registry.view().meets_max_risk("t1", "MEDIUM"):
    raise Exception("no current final rating of MEDIUM or lower")
```

## Deployment

Studio deployment: ADDRESS (constructor `900, 1800`).

The contract is submitted on its own; there is no frontend in this repository. `evidence/` holds sample tokenomics pages used for live testing. The sample prefix is `https://raw.githubusercontent.com/Mary1270/LockSight/main/evidence/`.

| File | Use |
|---|---|
| `aurora-v1.txt` | clean terms, rates LOW |
| `aurora-v2.txt` | supersedes v1 with worse terms, rates HIGH, drift WORSE |
| `aurora-v3.txt` | supersedes v2 with better terms, rates LOW, drift BETTER |
| `borealis-v1.txt`, `borealis-v1-amendment.txt` | LOW page and a counter page that moves it to MEDIUM (challenge upheld) |
| `cinder-v1.txt`, `cinder-v1-note.txt` | MEDIUM page and a counter page that does not change the level (challenge rejected) |
| `cinder-unlinked.txt` | a newer version that does not cite the old page (UNLINKED) |

## Tests

```
python3 -m unittest discover -s tests -p "test_*.py"
```

No pip install is needed. The suite runs against a small offline stand-in for the GenLayer SDK (`tests/genlayer_stub.py`) with a scripted stand-in for the model. After every call it checks that the contract's real GEN balance equals its recorded obligations and that per-address balances add up to the recorded total. It covers the URL rules, each parser, every rule in the table (all 243 point combinations), the validator checks against a tampered leader, supersede and drift, all challenge paths, stuck-state exits, a randomized sequence test, the shipped evidence pages, and file hygiene. A mutation check rewrites the contract with about 100 small faults (flipped comparisons, shifted thresholds, removed checks) and fails unless the suite catches every one. The stand-in model cannot show how a real model behaves; that is covered by the live tests on Studio.

See `ARCHITECTURE.md` for the state machine, consensus design, accounting and trust assumptions.

## License

MIT
