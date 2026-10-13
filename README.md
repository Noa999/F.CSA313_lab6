# Lab 6: Category-Partition and PICT

**Name:** Nomunzaya
**Student code:** B232270804

## Part A: Category-Partition (POST /transfer)

### Test assumptions (setup used for all values below)

- Source account A: currency USD, balance 1000, daily limit 500, no transfers made today (unless stated).
- Fixed test exchange rate: 1 EUR = 1.2 USD.
- Balance and daily limit are checked **after** converting the amount to the account currency.

### Priority order for multiple errors

The spec does not say which error wins when several apply, so I chose: **from -> to -> amount**, and inside amount: **BAD_AMOUNT -> INSUFFICIENT -> LIMIT**. I chose it because an account must exist and be usable before a transfer to or from it means anything, and the amount checks follow the order the rules are listed in. An amount over the balance also exceeds the daily limit here (limit 500 < balance 1000), so INSUFFICIENT must take priority over LIMIT.

### A1. Choices and representative values

| Parameter | Choice | Representative value |
|---|---|---|
| from | valid account | existing, not frozen (account A) |
| from | frozen | existing account, frozen |
| from | missing | id that does not exist |
| to | valid account | existing account B, different from A |
| to | missing | id that does not exist |
| to | same as from | to = account A |
| amount | normal | 100 |
| amount | exactly the daily limit | 500 |
| amount | zero | 0 |
| amount | negative | -50 |
| amount | over balance | 2000 |
| amount | over daily limit (within balance) | 600 |
| currency | same as account | USD |
| currency | different (conversion needed) | EUR |
| daily limit used (hidden environment) | empty | no transfers today |
| daily limit used (hidden environment) | almost full | 450 already transferred today |

### A2. Constraints

| Value | Tag | Justification |
|---|---|---|
| from = frozen | [ERROR] | Returns ERROR_FROZEN; combining it with other values would hide their errors. |
| from = missing | [ERROR] | Returns ERROR_NO_FROM, which wins over every later check. |
| to = missing | [ERROR] | Returns ERROR_NO_TO; tested once with everything else valid. |
| to = same as from | [ERROR], [IF from exists] | Returns ERROR_SAME_ACCOUNT; "same as from" is meaningless if from does not exist. |
| amount = zero | [ERROR] | Returns ERROR_BAD_AMOUNT (amount <= 0). |
| amount = negative | [ERROR] | Same error class as zero; kept as its own value to test the sign. |
| amount = over balance | [ERROR] | Returns ERROR_INSUFFICIENT. |
| amount = over daily limit | [ERROR] | Returns ERROR_LIMIT. |
| daily limit = almost full | [ERROR], [IF amount is positive and within balance] | Returns ERROR_LIMIT even for a normal amount; it only matters if the amount itself is valid. |
| amount = exactly the daily limit | [SINGLE] | Boundary of "does not exceed the limit"; one test is enough, combining it with other values adds nothing. |

**Count ("N -> M"):**

- Full set: from 3 x to 3 x amount 6 x currency 2 x daily limit 2 = **216**
- After constraints: 2 happy combinations (currency same / different) + 1 SINGLE (exact limit) + 9 ERROR values tested once each = **12**
- **216 -> 12**

### A3. Specification table

| # | from | to | amount | currency | daily limit used | Expected result | Note |
|---|---|---|---|---|---|---|---|
| 1 | valid | valid | normal (100) | same | empty | 201 OK | happy path |
| 2 | valid | valid | normal (100 EUR) | different | empty | 201 OK | conversion: 120 USD, within balance and limit |
| 3 | valid | valid | exact limit (500) | same | empty | 201 OK | SINGLE boundary |
| 4 | missing | valid | normal | same | empty | ERROR_NO_FROM | |
| 5 | frozen | valid | normal | same | empty | ERROR_FROZEN | |
| 6 | valid | missing | normal | same | empty | ERROR_NO_TO | |
| 7 | valid | same as from | normal | same | empty | ERROR_SAME_ACCOUNT | |
| 8 | valid | valid | zero | same | empty | ERROR_BAD_AMOUNT | |
| 9 | valid | valid | negative (-50) | same | empty | ERROR_BAD_AMOUNT | |
| 10 | valid | valid | over balance (2000) | same | empty | ERROR_INSUFFICIENT | also exceeds the limit; INSUFFICIENT wins by my priority order |
| 11 | valid | valid | over limit (600) | same | empty | ERROR_LIMIT | within balance (1000) |
| 12 | valid | valid | normal (100) | same | almost full (450 used) | ERROR_LIMIT | environment: 450 + 100 > 500 |
| 13 | valid | valid | 450 EUR | different | empty | ERROR_LIMIT | interesting: 450 < 500 as written, but 450 EUR = 540 USD after conversion |
| 14 | frozen | missing | normal | same | empty | ERROR_FROZEN | double condition: from is checked before to |
| 15 | valid | missing | zero | same | empty | ERROR_NO_TO | double condition: to is checked before amount |

Rows 1-12 are the 12 tests from the count above. Rows 13-15 are extra interesting cases (conversion effect and double conditions).

## Part B: PICT

### B1. Browser model

Model: `browser.model`. Row counts (header line excluded, counted with `tail -n +2 ... | wc -l`):

| Strength | File | Rows |
|---|---|---|
| 2-way | results/browser-2way.txt | 10 |
| 3-way | results/browser-3way.txt | 20 |

**2-way:** PICT gave 10 rows, one more than the 9 rows found by hand in the lecture. 9 is the minimum possible, because Content and Cookies have 3 values each, so there are 9 distinct pairs and one row covers only one of them. PICT uses a greedy algorithm: at each step it adds the row that covers the most uncovered pairs. That is fast and gives a small set, but a locally best choice does not always lead to the smallest overall set, so PICT does not guarantee the minimum.

**3-way:** 20 rows, compared with the full set of 3 x 3 x 2 x 2 x 2 x 2 = 144 combinations, which is about 14% of the full set.

### B2. Transfer models

Models: `transfer-nc.model` (no constraints), `transfer.model` (A2 constraints written as IF ... THEN), `transfer-neg.model` (`~` on the [ERROR] values). Counts come from `results/transfer-nc.txt`, `results/transfer.txt` and `results/transfer-neg.txt` (header excluded). An error is counted per row when from is not Valid, to is not Valid, amount is Zero/Negative/OverBalance/OverLimit, or the daily limit is AlmostFull.

| Version | File | Rows | Happy-path rows | Rows with 1 / 2 / 3 / 4 errors | Max errors per row |
|---|---|---|---|---|---|
| No constraints | results/transfer-nc.txt | 18 | 0 | 3 / 6 / 7 / 2 | 4 |
| A2 constraints | results/transfer.txt | 19 | 1 | 18 / 0 / 0 / 0 | 1 |
| `~` negative values | results/transfer-neg.txt | 22 | 3 | 19 / 0 / 0 / 0 | 1 |

**Reading the output.** Without constraints there is no happy-path row, and 15 of 18 rows contain two or more errors at once. In those rows the first error in my priority order (from -> to -> amount) hides the others, so a row such as Missing + Zero + OverLimit checks only ERROR_NO_FROM and says nothing about the amount rules. Both the constraint version and the `~` version give at most one error per row, so every error is tested alone.

**Are the A2 constraints still needed with `~`?** Partly. `~` already solves error masking, which was the main job of my [ERROR] constraints, and it also removes combinations such as from = Missing with to = SameAsFrom. It does not remove combinations that are valid in PICT's eyes but meaningless in my design. `ExactLimit` (500) is a boundary value only with the same currency and an empty daily limit: 500 EUR is 600 USD, which is over the limit, and 500 plus 450 already used is also over the limit. So I kept one constraint in `transfer-neg.model` for ExactLimit. `~` replaces the [ERROR] tag, but not the [SINGLE] and [IF] tags.

**Real test cases.** Common setup: account A (USD, balance 1000, daily limit 500, nothing transferred today), account B (USD, balance 0, active), account F (USD, frozen). Fixed rate 1 EUR = 1.2 USD. Input errors return HTTP 200 with an ERROR result.

| # | Source row | Setup | Input body | Oracle |
|---|---|---|---|---|
| T1 (happy path) | `transfer-neg.txt` line 2: Valid, Valid, Normal, Same, Empty | A and B as above | `{"from":"A","to":"B","amount":100,"currency":"USD"}` | 201, `{"result":"OK"}` |
| T2 (one error) | `transfer-neg.txt` line 7: Valid, Valid, ~OverBalance, Same, Empty | A and B as above | `{"from":"A","to":"B","amount":2000,"currency":"USD"}` | 200, `{"result":"ERROR_INSUFFICIENT"}` (2000 > balance 1000) |
| T3 (one error) | `transfer-neg.txt` line 13: Valid, ~SameAsFrom, Normal, Different, Empty | A as above | `{"from":"A","to":"A","amount":100,"currency":"EUR"}` | 200, `{"result":"ERROR_SAME_ACCOUNT"}` |
| T4 (multi-error) | `transfer-nc.txt` line 7: Frozen, SameAsFrom, Zero, Same, AlmostFull | F is frozen, 450 already transferred today | `{"from":"F","to":"F","amount":0,"currency":"USD"}` | 200, `{"result":"ERROR_FROZEN"}` |

T4 has four errors at once (frozen, same account, bad amount, and a limit problem). By my priority order from -> to -> amount, the from check comes first, so the oracle is ERROR_FROZEN. This row only checks that ordering and says nothing about the other three errors.

### B3. Conclusion

Manual category-partition produced 15 specifications (216 -> 12 plus 3 extra cases), and every row has an expected result written next to it. PICT produced 19 rows with hand-written constraints and 22 rows with `~`, but its output has no expected results, so I had to add the oracle for every row myself. The manual table captures error semantics better: each error class has its own row, and my priority order decides the double-condition rows. The `~` model comes close, because it limits each row to one error, but PICT still does not know which ERROR_ code should be returned. PICT guarantees pairwise coverage for the values I give it, and my manual table does not: for example, I never checked every pair of currency and from. With `~`, the pair coverage of error values with each other is deliberately dropped, so the guarantee holds only for pairs of normal values and for each error with the normal values. The unconstrained model showed why constraints matter: it had no happy-path row and 15 of 18 rows had two or more errors. PICT is also greedy, so it gave 10 rows for the browser model where 9 are enough, and its sets are not always minimal. The manual work and PICT complement each other: PICT is faster and systematic, the manual design gives meaning to the tests. The step that needed the most thinking was A2, deciding which values are [ERROR], [SINGLE] or [IF] and the priority order, because the PICT models and the oracles all depend on it.
