# NTAK RMS compliance overview

Review of `agocsvince/ntak-php` against:

- **NTAK VENDÉGLÁTÁS RMS interface description v1.07** (2025-09-24, MTÜ)
- **NTAK RMS test-connection questionnaire v1.01**

Date of review: 2026-10-15. Code reviewed: `master` @ `e86c587`.

---

## Verdict

This repository **is a PHP client SDK for the NTAK Vendéglátás RMS interface**. It is **not** a complete RMS (restaurant / hospitality POS).

| Question | Answer |
| --- | --- |
| Does it implement the official RMS REST interface? | **Yes** — the three v1.07 endpoints and the documented JSON schema |
| Does it compile? | **Yes as a PHP 8.1+ library** (syntax-clean, Composer installs, offline pricing tests mostly pass) |
| Is it itself an RMS product that can be registered on the questionnaire? | **No** — manufacturers still need a real hospitality system around this library |
| Can it pass MTÜ validation tests as-is? | **No** — missing certs, send cadence, retry, CSR, and several client-side guards |

The README title (“NTAK RMS PHP API / SDK”) is accurate. The package helps an RMS **talk to NTAK**. It does not replace an RMS.

---

## What was compiled and tested

Environment: PHP 8.3.6, Composer 2.7.1, PHPUnit 10.5.45. No NTAK client certificate was present (`auth/` is gitignored).

| Check | Result |
| --- | --- |
| `composer install` | Success |
| `php -l` on every file under `src/` | No syntax errors (25 files) |
| Offline unit tests (`PricingTests`, `CategoryTest`) | **28 passed / 4 errors / 153 assertions** |
| Live / cert tests (`StoreOrder`, `CloseDay`, `Verify`, `EnumTest`) | All failed — no `auth/pem.pem` |
| Live swagger enum sync (`GET /v3/api-docs/rms`) | Not run — cert required |

The 4 offline errors are a **Linux PSR-4 case mismatch**, not a pricing bug:

- class `Kiralyta\Ntak\Tests\FakeNTAKClient`
- file `src/Tests/FakeNtakClient.php`

PHPUnit only hits this when `PricingTests::log()` instantiates the fake client (4 cases). The remaining 26 pricing cases, which actually assert `tetelOsszesito` / discount / service-fee / DRS maths, **passed**.

So: **the library compiles and the money-rounding engine works**. End-to-end RMS submission was **not** proven here.

---

## What this package actually is

```
NTAKClient  --TLS client cert + detached JWS-->  https://rms.ntak.hu  (prod)
                                                 https://rms.tesztntak.hu (test)
    POST /rms/rendeles-osszesito
    POST /rms/napi-zaras
    POST /rms/ellenorzes
```

Public API:

- `NTAK::message($client, $when)->handleOrder(...$orders)` → process id
- `->closeDay($start, $end, $dayType, $tips)` → process id
- `->verify($processId)` / `->verifyAll($ids)` → `NTAKVerifyResponse`

Convenience the spec does **not** define, but that still emit legal line items:

- order-level and item-level `%` discounts → `EGYEB` / `KEDVEZMENY` (negative)
- `%` service fee → `EGYEB` / `SZERVIZDIJ`, split by VAT
- DRS deposit (hardcoded 50 Ft) → `EGYEB` / `KORNYEZETBARAT_CSOMAGOLAS` / `E_0`
- takeaway VAT override: `ALKMENTESITAL_HELYBEN` forced to `C_27`
- HUF cash rounding to 5 Ft + `KEREKITES` payment row

---

## Spec coverage (v1.07)

### Implemented and matching

| Spec item | SDK |
| --- | --- |
| Three endpoints only: order summary, daily close, verify | `/rms/rendeles-osszesito`, `/rms/napi-zaras`, `/rms/ellenorzes` |
| Header: `szolgaltatoAdatok`, `uzenetAdatok`, `kuldoRendszerAdatok` | Exact field names, including `vendeglatoUzletRegSzam` |
| Order fields (`rendelesBesorolasa`, UUID id, `targynap`, times, `helybenFogyasztott`, `osszesitett`, payments, items) | Present |
| Enums: order type, day type, VAT, payment, amount (incl. `ADAG`), verify status | All names match appendix 6.3 / chapter 4.6 |
| Category ↔ subcategory matrix (6.3) | Encoded in `NTAKCategory::subcategories()` |
| Detached JWS: `x-jws-signature` = `header..signature`, payload empty | `NTAKClient::requestHeaders()` |
| Algorithm RS256 + `x-certificate` (base64 PEM) | Present |
| Cash rounding: paid − due, remainder as `KEREKITES` | `NTAKPayment` |
| Discount / service fee split by ÁFA, discount negative | `NTAKOrder` builders |
| Storno omits items, payments, times | `handleOrder()` |
| Closed-day close omits open/close timestamps | `closeDay()` |
| Verify response: status, successful / unsuccessful / header errors | `NTAKVerifyResponse` |
| Timezone `Europe/Budapest`, RFC 3339-ish ISO-8601 | Carbon `toIso8601String()` |
| Official example host path shape | Same `/rms/...` paths |

Sample payload produced by the SDK (header + one discounted order) matches the v1.07 curl examples in chapter 6.2, including `fizetesiInformaciok` (unaccented — the spec table on p.27 is inconsistently accented; the official examples use the unaccented name the SDK sends).

### Missing or incomplete vs v1.07

| Severity | Gap | Why it matters |
| --- | --- | --- |
| High | Not a full RMS | Questionnaire + chapters 2–3 require CSR generation, cert install, 15-minute send cadence, validation-test protocol, screenshots, retry of failed / `UJRA_KULDENDO` messages |
| High | Almost no client-side schema validation | NTAK will reject asynchronously: UUID v4, 11-digit tax number, 10-char shop id, ≤20-char RMS id/version, `megnevezes` ≤255, `mennyiseg` ≠ 0, `rendelesVege − rendelesKezdete` ≤ 24 h, `zaras − nyitas` ≤ 24 h, 14-month `targynap` window, ≤500 items / message, `∑ tetelOsszesito = rendelesVegosszegeHUF`, `∑ fizetettOsszegHUF = rendelesVegosszegeHUF` |
| High | Category/subcategory pairing is not enforced on send | `hasSubcategory()` exists but is unused. `PricingTests` even builds `ALKMENTESITAL_HELYBEN` + `ROSTOS_UDITO`, which appendix 6.3 forbids (async `Conflict`) |
| High | TLS `verify => false` | Spec requires the NTAK RMS cert for **mutual** TLS; disabling server verification is unsafe in production |
| Medium | `osszesitett` is hard-coded `false` | No `osszesitettIndoklasa` support (outage / cert-expiry aggregated days) |
| Medium | `UJRA_KULDENDO` exact resend | Spec: resend the **identical** body, including original `uzenetKuldesIdeje`. Callers who pass `Carbon::now()` will mint a new timestamp |
| Medium | `KESZPENZEUR` also rounded to 5 HUF | Spec only requires `0` or `5` ending for `KESZPENZHUF` |
| Medium | `closeDay($start, null, NORMAL_NAP)` fatals | `Call to a member function timezone() on null`. Only closed-day (`ADOTT_NAPON_ZARVA`) is null-safe |
| Medium | Service-fee line shape differs from the official example | Example: `tetelszam: 0.01` × unit price. SDK: `tetelszam: 1` × fee amount. Both can satisfy `tetelOsszesito = round(bruttoEgysegar * tetelszam)`, but integrators copying the example will not match this SDK |
| Medium | No HELYESBITO / SZTORNO integration test | `test_destroy_order` is commented out |
| Medium | README vs code | README amount table omits `ADAG` (present in enum since changelog 2024-02-06). README is otherwise ahead of changelog (item-level discounts) |
| Low | Test host in PDF examples is `https://rms.ntaktst.hu` | SDK uses `https://rms.tesztntak.hu`. Hosts are handed out in the MTÜ info pack and have changed over versions; confirm before go-live |
| Low | Changelog last dated 2024-02-06 | Spec v1.07 (14-month `targynap` window, extra error keys) is 2025-09-24 — no SDK change for those rules |
| Low | No `phpunit.xml` | `vendor/bin/phpunit src/Tests` works only if Composer autoload is bootstrapped |
| Low | Extra fields always sent | `hivatkozottRendelesOsszesito: null` on NORMAL; `osszesBorravalo: 0` on closed days (required only for `NORMAL_NAP`) |

### Questionnaire (v1.01) — not implemented

The one-page form is an **MTÜ registration request**, not an API. This repo cannot fill it:

- company / tax / address / representative
- public listing (vendor name, **RMS name + version**, website)
- public test IP
- planned test start
- own-use vs commercial declaration

`NTAKClient` *stores* `softwareRegNumber` and `version` (`rmsRendszerNTAKAzonosito`, `rmsRendszerVerzioszam`) once MTÜ issues them. It does not produce the form, a CSR, or an RMS listing on info.ntak.hu.

---

## Compile / runtime notes

1. **PHP `^8.1` enums** — required, as documented.
2. **No `declare(strict_types=1)`** — `Ramsey\Uuid\Uuid` objects are coerced to string for `orderId`. Official type is UUID v4; `"not-a-uuid"` is also accepted.
3. **`Carbon ^2.66` only** — Carbon 3 is not allowed.
4. **Guzzle 30 s / 20 s timeouts** — present (commit `f674910`).
5. **Fake client does not merge the header** into `lastRequest()`, so debug dumps miss `szolgaltatoAdatok` / `uzenetAdatok` / `kuldoRendszerAdatok`.
6. Integration tests embed dummy credentials (`taxNumber=11223344122`, `regNumber=ET23002480`, `softwareRegNumber=TABTENDER`) and hit the **live test host**. They are not hermetic.

---

## What an integrator still has to build

To be a certifiable RMS around this SDK (chapters 2–4):

1. Generate PKCS#10 CSR (RSA 4096, SHA-256, CN = service-location id) — spec 6.1; not in this repo.
2. Concatenate issued `.cer` + private `.key` into the `.pem` `NTAKClient` expects.
3. Queue order summaries at a **configurable ≤15 minute** interval; never more than **500 line items** per HTTP call.
4. Enforce send order: NORMAL → HELYESBITO → SZTORNO → napi zárás.
5. After each accepted process id, poll `/rms/ellenorzes` (wait ≥60 s as the README says; must happen within 24 h, data gone after 1 month).
6. On `TELJESEN_HIBAS` / `RESZBEN_SIKERES`, correct and resend with a **new** `rmsRendelesAzonosito`.
7. On `UJRA_KULDENDO`, resend the **byte-identical** original message.
8. Send a daily close **every tárgynap**, including closed and zero-traffic days; open→close ≤ 24 h.
9. Keep `targynap` = currently open business day’s opening date; reject >14-month-old days (v1.07).
10. Produce the MTÜ validation protocol + screenshots. This library only returns process ids.

---

## Bottom line

Treat `kiralyta/ntak-php` / `agocsvince/ntak-php` as a **well-aimed v1.07 message builder**, not as an RMS and not as a finished compliance product.

- **Schema alignment is good.** Endpoints, headers, enums, JWS shape, cash rounding, and VAT-split discount/fee lines match the official document and examples.
- **It compiles as a library.** Syntax is clean; Composer works; pricing tests (the valuable ones) pass.
- **It does not compile into, or substitute for, an RMS.** Registration, certificates, cadence, retries, and most v1.07 validations are the host application’s job.
- **Do not ship to production with `verify => false`, unchecked category pairings, or non-UUID order ids.**

Recommended next steps if this fork is meant to stay v1.07-clean:

1. Rename `FakeNtakClient.php` → `FakeNTAKClient.php` so Linux CI can autoload it.
2. Validate UUID / lengths / 24 h windows / 500-item cap / category pairing before POST.
3. Add `osszesitettIndoklasa` and a first-class “resend exact message” path.
4. Turn TLS verification on; pin or bundle the NTAK CA.
5. Add hermetic payload-contract tests (no live host) against the chapter 6.2 examples.
6. Confirm the current test/prod base URLs with `dev.support@ntak.hu` before any validation window.
