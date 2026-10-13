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

TODO: model files (transfer-nc, transfer, transfer-neg), comparison table (rows, happy-path rows, errors per row), whether the A2 constraints are still needed with `~`, 3 real test cases (setup + input + oracle).

### B3. Conclusion

TODO: 8-10 sentences comparing the manual table with PICT output.
