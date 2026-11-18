# LockSight architecture

## Entities

- **Prefix**: an https URL prefix with a label, added by the contract owner. Pages can only be registered under a current prefix. When a URL matches more than one prefix, the longest (most specific) one is used.
- **Evidence binding**: at registration the matched prefix and its label are copied onto the item (`prefix`, `label`). This binding is immutable and is the item's challenge and supersede authority: counter-evidence and newer versions must sit under the stored `prefix`, never under the owner's current prefix list. Removing, re-adding or adding prefixes later affects only new registrations.
- **Item**: id `t1`, `t2`, ... with the canonical page URL, registrant, state, optional assurance, rating, challenge data and version links. One URL maps to one item; a URL whose item ended UNRATED may be registered again.
- **Project**: the chain of versions that starts at an item registered with `register_page` (`project` is that item's id). It stores the ordered list of item ids and `head`, the latest FINAL version.
- **Rating**: risk level, score, the rule hits (`why`), the derived fields (band and number per field), the points per field (`parts`) and the grounded quotes (`raw`) they came from. The whole record is rebuilt by the contract from the quotes; nothing else a leader sends is stored.
- **Drift**: stored on a FINAL item that has a predecessor: direction, scores, risk levels and the list of changed fields.

## Item state machine

```
register_page / supersede -> REGISTERED
REGISTERED --assess RATED--> RATED
REGISTERED --assess not rated (attempts+1)--> REGISTERED, or UNRATED on the 3rd attempt (assurance refunded)
REGISTERED --stale, expire_item--> UNRATED (assurance refunded)
RATED --challenge, before challenge_end--> CHALLENGED
RATED --finalize at or after challenge_end--> FINAL (assurance refunded)
CHALLENGED --resolve, merged risk differs--> FINAL (challenger: bond + assurance), rating replaced, original_risk kept
CHALLENGED --resolve, merged risk equal--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve, counter page lacks the item URL--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve, undetermined--> RATED, bond refunded, window reopened
CHALLENGED --stale, cancel_challenge--> RATED, bond refunded, window reopened
```

Every state has a path to FINAL or UNRATED that needs no particular party. A consensus failure leaves storage untouched, and the stale timers (`stale_after_seconds`) then open `expire_item` or `cancel_challenge`.

**Reopening.** An undetermined resolution (for example the counter page could not be fetched) or a stale cancellation refunds the bond and sets the item back to RATED. The window is restored to exactly the time that was left when the challenge was filed (`challenge_end - challenge_filed`), so the clock stands still while a challenge is open and the window never closes as a side effect of a refund. The challenge slot is not consumed, so the challenger or someone else can try again. This restoration happens at most `MAX_REOPENS` (2) times per item; after that, a refunded challenge leaves the original deadline in place, so time spent in it counts. The total delay a challenger can cause is therefore bounded by two stale periods.

## Versions

`supersede(item_id, url)` is accepted only if all of these hold, otherwise the attached value goes to the caller's balance:

1. the previous item exists and is FINAL;
2. it has no newer version in progress and has not been superseded;
3. the new URL is clean and under the previous item's stored prefix;
4. the new URL is not already registered (unless that item ended UNRATED);
5. the assurance is a whole number of GEN from 0 to 100 and the item limit is not reached.

The new item copies the project, label and prefix, gets `version = previous + 1` and `prev`, and the previous item gets `successor`. During `assess`, a page with `prev` must cite the previous page's full URL as a whole token (case-insensitive); otherwise the attempt ends UNLINKED with no model call. This is a code-checked link between versions, so a different page that merely sits under the same prefix cannot be pushed into the chain without naming the page it replaces.

When the new item becomes FINAL (by `finalize` or by a resolved challenge, so the final, possibly revised rating is used), `_close` does three things in one step: sets the project's `head`, stores the drift against the previous final rating, and marks the previous item `superseded_by` (clearing `successor`). When the new item ends UNRATED, `successor` is cleared and another version can be submitted.

Drift direction is decided by the score alone: higher is WORSE, lower is BETTER, equal is SAME. Because unstated fields cost a point, a version that stops disclosing a term reads as worse. `changes` lists every field whose band or number moved, including moves inside one band that leave the score unchanged.

Registry reads: `get_risk` serves only an item that is FINAL and not superseded, so a stale id yields an empty string. `get_project_risk` returns the head's risk.

## Extraction and rating pipeline

`_assess_single` (and, for the counter page, `_assess_challenge`) runs identically on the leader and on every validator:

1. Fetch the page (first 12,000 characters). A failed fetch is UNDETERMINED.
2. For a new version, check the citation of the previous URL (UNLINKED if missing).
3. Ask the model for a fixed JSON object: an exact quote for the team share, cliff, vesting, initial unlock and lock, plus a YES / NO / UNSTATED flag for cliff and vesting and a CONTRACT / THIRD_PARTY / NONE / UNSTATED label for the lock. The prompt declares the page untrusted and forbids rating.
4. `_ground`: values are coerced to the expected types (a non-string such as `null` becomes an empty value, never the text "None"); each quote is kept only if it is 3 to 200 characters (a longer one is dropped, not cut) and, after whitespace and case normalisation, occurs in the page with word boundaries at both ends, so a digit or a decimal part cannot be cut off (`5%` does not match inside `25%` or `2.5%`); invalid enumerations become UNSTATED.
5. `_derive_fields`: the code reads the quotes. Percentages are the first number followed by `%` or `percent` (number words up to ninety-nine are accepted). Durations are the number next to a day, week, month, quarter or year unit, choosing the one nearest a word starting with "cliff" or "vest" (so "investors" is not an anchor) when a quote holds several; they are converted to days with integer arithmetic. A flag without a surviving quote is UNSTATED. A quote that negates that same term ("no cliff", "does not vest", "cliff: none") and gives no duration means NONE whatever the flag says; a NO flag is accepted only with such a negation or a zero duration, otherwise it is UNSTATED. A zero initial unlock is read only when a zero word sits right before a release word ("no tokens are unlocked"), so "no lockup, everything unlocked" is not read as 0%. A lock label is accepted only if the quote supports it: CONTRACT needs contract words, no negation of the lock and no multisig; THIRD_PARTY needs custody or escrow words, no negation and no self-custody; NONE needs a negated lock or words such as "fully unlocked" or "unrestricted".
6. `_rate_raw`: points are summed (see README) and mapped to LOW / MEDIUM / HIGH. Fewer than three stated fields gives UNDETERMINED.

Conservative defaults: a fact that cannot be grounded or parsed is treated as not stated, and an unstated field adds a point rather than removing one. Text the model invents is dropped by the quote check, so it cannot lower a score.

## Consensus

Both consensus calls use `gl.vm.run_nondet_unsafe(leader, validator)` with positional arguments, from module-level functions that capture only plain strings, never the contract.

- The leader returns its result as canonical JSON. Errors inside the closure become an ordinary UNDETERMINED result.
- A validator re-runs the whole pipeline on its own copy of the page and requires an exact match of the consensus key: the status, the band of each of the five fields, the score and the risk level. Only these values can change a rule, so the key is as coarse as the rules are. Raw numbers are not compared; because the risk level and every band that feeds it are compared exactly, two validators cannot take different decisions near a band edge, and harmless wording or number differences inside a band do not block consensus.
- The validator also checks the leader's output deterministically: every leader quote occurs in the validator's own page, and re-deriving and re-scoring the leader's quotes reproduces the leader's fields, points, score and risk. A leader therefore cannot get a fabricated quote or a number that does not follow from its quotes accepted. A tampered or malformed leader result is rejected.
- The contract does not store the leader's rating object. After consensus it rebuilds the rating from the leader's quotes with the same code (`_clean_raw`, `_rate_raw`), so extra keys, invented rule hits or oversized text sent by a leader never reach storage, and failure reasons come from a fixed list.
- A challenge does not re-extract the original page. It starts from the grounded quotes stored with the consensus-accepted rating, so a challenger cannot win just because a second extraction of the same page came out differently. Only the counter page is fetched and extracted. It must contain the full stored item URL as a whole token (case-insensitive, within the first 12,000 characters; a longer URL that merely starts with it does not count); without it the result is NO_ANCHOR and no model call is made. A counter field replaces the stored field only if its own quote parses to a stated value, so unreadable text cannot erase a stored field. The merged fields are scored by the same rules and compared with the stored risk level.

Pages should be static text, ideally pinned to a commit. If the page changes between validators' fetches, validators disagree and the transaction does not complete; the stale paths cover that.

## URL rules

Page and counter-evidence URLs must be https, ASCII with no whitespace or control characters, no credentials, no non-443 port, no `..` or `.` path segments, no `//` in the path, no `%2e`, `%2f`, `%5c` or `%00`, no query and no fragment, must not end in `/`, and, after lower-casing the host and dropping `:443`, start with one of the owner's current prefixes (registration), with the previous item's stored prefix (supersede) or with the item's stored prefix (counter-evidence). Prefixes themselves must end with `/`, have a non-empty path and no query. Look-alike hosts and `host@evil` forms fail the parsed-URL checks.

## Money

All amounts are wei internally; users enter whole GEN. Payouts are pull-based: every credit goes to `balances` and `withdraw` transfers after zeroing the balance. The three payable methods (`register_page`, `supersede`, `challenge`) never raise on bad input; they credit the attached value back to the caller's balance. Invariant checked in tests after every call:

```
contract balance == sum(balances) + sum(open challenge bonds) + sum(open assurances)
sum of per-address balances == recorded total
```

`get_accounting` computes the first right-hand side by scanning every item and also reports whether it matches the running totals.

## Trust assumptions

- The owner is trusted to choose prefixes under content the project controls. The owner cannot change ratings, windows, bonds or any balance, and changing the prefix list does not affect existing items: their assessment, challenge, resolution, cancellation, finalization and versions all use the prefix stored at registration. A prefix on a place where anyone can publish lets anyone author a page; consumers of the registry should check the item's `label` and URL.
- Anyone can submit a new version of a FINAL page, but only a page under the stored prefix that cites the previous URL. The registrant of an item has no special control over its versions.
- A counter-evidence page is treated as an authoritative amendment for the fields it states. Challenging therefore requires a page under the item's own stored prefix that cites the item URL.
- The risk rules are a transparent summary of stated facts. They do not read token contracts, do not verify that a lock exists on chain and say nothing about facts the page leaves out beyond the not-stated points.
- Validators are trusted only to locate quoted facts. They never produce the risk level, and the code checks their quotes.
- A registrant who attaches assurance accepts that a successful challenge pays it to the challenger, including when the cause is an ambiguous page.

## Known limits

- One rating per URL. A changed page needs a new URL (a dated or versioned file, or an archive snapshot) and `supersede`.
- Percentages are read from digits or number words; figures expressed only in token counts are not converted.
- Quotes are capped at 200 characters and pages at 12,000 characters.
- At most 500 items and 10 prefixes; list views are paginated (50 per page).
- Time comes from `datetime.datetime.now()` as seen by GenVM.
