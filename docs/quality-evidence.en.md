# Quality & security evidence — Margin Keeper 1.0.5

A condensed, English summary of the evidence behind the claims in the README. The underlying reports (test logs, red-team reports, the acceptance report) live in the private repository and can be shown on request.

## 1. Automated test matrix (every release build)

| Stack | Tests | Result |
|---|---|---|
| PHP 8.3 · WordPress 7.1.3 · WooCommerce 11.2.0 · HPOS | 276 | pass |
| PHP 8.3 · WordPress 7.1.3 · WooCommerce 11.2.0 · legacy order storage | 276 | pass |
| PHP 7.4 · WordPress 6.7.9 · WooCommerce 10.3.0 · HPOS | 276 | pass |
| PHP 7.4 · WordPress 6.7.9 · WooCommerce 10.3.0 · legacy order storage | 276 | pass |
| Each stack, WooCommerce inactive | 5 | pass |

Tests run against a real WordPress and WooCommerce, never mocks of the behaviour under test. They include:

- **Golden-data integration tests:** a seeded store of 16 sellable items and 15 orders with hand-computed expected values (e.g. the two dashboard totals 1,123,395 and 1,072,245 Toman), asserted exactly.
- **Write-scope proofs:** a fingerprint of the whole database before and after every write flow (bulk update by percentage, by dollar, from a file, migration, revert). Only cost values and the plugin's own tables may differ.
- **Red-team regression tests:** one targeted test per finding of the two adversarial reviews (idempotent retries, stale previews, inheriting variations, inconsistent stock data, per-product permissions, category scope, cancel and stale jobs, settings-changed notice).
- **Upload hardening:** oversize, too many rows, renamed type, zip bomb, XXE, formula cells, huge shared strings, nothing kept on disk.
- **Access:** subscriber, customer, a user with `manage_woocommerce` only, shop manager and administrator, against every page and every write.
- **Jalali calendar:** every day from 1350/01/01 to 1450/12/end compared with ICU's Persian calendar, in both PHP versions.

## 2. Static analysis

PHPCS (WordPress Coding Standards + PHPCompatibility 7.4+), PHPStan, WordPress Plugin Check (0 errors; 2 warnings reviewed as false positives), Psalm taint analysis (no tainted flow from request input to output; a planted flaw was detected to validate the tool).

## 3. Performance (10,000 products, 300 orders)

| Request | WooCommerce 10.3 | WooCommerce 11.2 |
|---|---|---|
| Dashboard calculation, slowest step | 10.0 s | 10.2 s |
| Bulk preview of 5,000 rows | 7.0 s | 8.1 s |
| Supplier file of 5,000 rows: read, match, preview | 10.1 s | 10.3 s |
| Migration preview of 10,000 rows | 13.6 s | 14.6 s |

Limit: 20 s per request. All steps are resumable, so a lost connection never loses progress.

## 4. Security

| Review | Scope | Outcome |
|---|---|---|
| Red team, Slice 1 (2026-09-26) | Read-only report | 0 critical, 2 important, 14 minor; all fixed |
| Red team, v1 (2026-10-06) | Whole product incl. write flows | 0 critical, 5 important, 9 minor; all fixed except one deferred performance note |
| Pre-sale review (2026-10-08) | Independent adversarial reviewer, 95 live attacks on a disposable site, Psalm taint analysis | 0 critical/high/medium; 2 low (availability only) fixed; 4 hardening notes documented |

Live attacks included anonymous and low-privilege access to every page and AJAX action, CSRF without and with forged nonces, GET-based writes, a user with the view capability but no edit rights using an admin's nonce, stored XSS in product names, SKUs and category names, CSV formula injection, a PHP file renamed `.csv`, an XXE xlsx, a 600,000-string zip bomb and a zip-slip entry. All were refused; the store's data fingerprint was identical before and after.

## 5. Release process

`bin/build.sh` on a clean, committed tree: all checks and tests → package from `git archive` of the tagged commit minus development files → scan for forbidden files → fresh WordPress + WooCommerce install (oldest and newest stack): install, activate, open the page with COGS off and on, deactivate, uninstall with data kept, reinstall, uninstall with data deleted (no options or tables left) → Plugin Check on the zip → local tag `v<version>-clean`. Every package's SHA-256 is recorded.

## 6. Acceptance

- Screenshot-based acceptance against the specification's 8 criteria (all passed), checked by the product owner.
- A full functional pass in the browser as a store owner, on a fresh Persian WordPress and on the seeded test store: first run, settings validation, dashboard numbers, simulator, product box, all four bulk methods with apply and revert, a test order, and the revert's order list. No functional defect was found.
