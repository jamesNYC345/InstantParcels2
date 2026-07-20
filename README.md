# Carrier Tracking Number Formats

**A developer reference for recognizing, parsing, and validating shipping and postal tracking numbers.**

Every courier encodes its tracking numbers differently. Some follow a global postal standard, some use a proprietary layout with a built-in check digit, and some are just a run of digits with no structure you can rely on. If you have ever tried to answer the deceptively simple question *"which carrier does this tracking number belong to?"*, you already know there is no single, authoritative, machine-readable table for it. This repository is an attempt at one.

It documents the formats used by the major global carriers and national postal operators, gives you a tested regular expression for each, and includes working check-digit algorithms for the formats that have them. Use it to build a carrier detector, to validate user input before you hit a tracking API, or just as a reference when a tracking number lands on your desk and you have no idea who to ask.

If you would rather skip the parsing entirely and just track a parcel across any of these carriers in one place, that is exactly what [InstantParcels](https://instantparcels.com) does: it aggregates tracking for USPS, UPS, FedEx, DHL and 600+ carriers and national postal operators behind a single lookup. This document is the open reference that sits underneath that kind of problem.

---

## Contents

- [How to read this document](#how-to-read-this-document)
- [Quick reference table](#quick-reference-table)
- [The UPU S10 international standard](#the-upu-s10-international-standard)
- [USPS](#usps)
- [UPS](#ups)
- [FedEx](#fedex)
- [DHL](#dhl)
- [Royal Mail and the S10 postal family](#royal-mail-and-the-s10-postal-family)
- [European parcel carriers](#european-parcel-carriers)
- [Detecting the carrier from a number](#detecting-the-carrier-from-a-number)
- [Validating check digits](#validating-check-digits)
- [Caveats and limitations](#caveats-and-limitations)
- [Contributing](#contributing)
- [Resources](#resources)

---

## How to read this document

Two properties of a tracking number matter when you are trying to identify it programmatically: its **shape** and its **check digit**.

The *shape* is the pattern of letters and digits: how many characters, where the letters sit, and whether it ends in a two-letter country code. Shape is what a regular expression captures, and it is usually enough to narrow a number down to a small set of candidate carriers. The problem is that shapes overlap. A bare string of twelve digits could be a FedEx number, a Deutsche Post number, or something else entirely, because a plain numeric run carries no identifying marks.

That is where the *check digit* comes in. Several formats reserve one position for a value computed from the others, so that a typo or a random string can be rejected without ever contacting the carrier. When a format has a check digit, validating it turns a *maybe* into a near-certainty. When it does not, shape is all you have, and you should treat detection as a best guess rather than a fact.

Throughout this document, regular expressions assume the input has been **uppercased and stripped of spaces and hyphens** first. Carriers and shipping labels love to print tracking numbers with spaces (`9400 1118 9922 ...`) or in lowercase, and normalizing before you match saves you a lot of pattern noise.

```
normalized = raw.upper().replace(" ", "").replace("-", "")
```

---

## Quick reference table

| Carrier | Typical shape | Example | Regex (normalized input) | Check digit |
|---|---|---|---|---|
| **USPS** | 22 digits (IMpb) | `9400111899223197428497` | `^(94\|93\|92\|95)\d{20}$` or `^\d{20,22}$` | MOD 10 |
| **UPS** | `1Z` + 16 chars | `1Z999AA10123456784` | `^1Z[0-9A-Z]{16}$` | UPS 1Z |
| **FedEx** | 12 / 15 / 20 digits | `020207021381215` | `^\d{12}$\|^\d{15}$\|^\d{20}$` | none (varies) |
| **DHL Express** | 10 digits | `1234567890` | `^\d{10}$` | none |
| **DHL eCommerce** | `GM` + digits | `GM6000000000000000` | `^GM\d{16,20}$` | none |
| **Royal Mail** | S10, `…GB` | `AB123456785GB` | `^[A-Z]{2}\d{9}GB$` | S10 (MOD 11) |
| **Canada Post** | 16 digits | `1234567890123456` | `^\d{16}$` | none |
| **Australia Post** | S10 or 10–12 digits | `AB123456785AU` | `^[A-Z]{2}\d{9}AU$` | S10 (MOD 11) |
| **China Post / EMS** | S10, `…CN` | `RB123456785CN` | `^[A-Z]{2}\d{9}CN$` | S10 (MOD 11) |
| **India Post** | S10, `…IN` | `EE123456785IN` | `^[A-Z]{2}\d{9}IN$` | S10 (MOD 11) |
| **Deutsche Post / DHL DE** | 12 / 16 / 20 digits | `RR123456785DE` | `^\d{12,20}$` or S10 | none / S10 |
| **La Poste / Colissimo** | 13 chars | `6A12345678901` | `^[0-9A-Z]{2}\d{9}[0-9A-Z]{2}$` | none |
| **PostNL** | `3S…` or S10 | `3SABCD1234567` | `^3S[A-Z0-9]{9,11}$` | none / S10 |
| **DPD** | 14 digits | `01234567890123` | `^\d{14}$` | none |
| **GLS** | 11–14 digits | `12345678901` | `^\d{11,14}$` | none |
| **Evri (Hermes)** | 16 digits | `1234567890123456` | `^\d{16}$` | none |
| **International (any)** | S10 | `RB123456785DE` | `^[A-Z]{2}\d{9}[A-Z]{2}$` | S10 (MOD 11) |

The examples above are structurally valid and, where a check digit exists, they pass it. They are constructed for documentation and do not correspond to real shipments.

---

## The UPU S10 international standard

By far the most useful format to understand is **S10**, the Universal Postal Union standard for identifying international postal items. It is used by essentially every national postal operator for registered mail, EMS, insured parcels, and tracked cross-border items, which means a single pattern covers a huge share of the world's tracked mail.

An S10 identifier is always thirteen characters, in three parts:

```
 R B 1 2 3 4 5 6 7 8 5 D E
 └┬┘ └────┬────┘ │ └┬┘
  │       │      │  └── ISO 3166 country code (2 letters)
  │       │      └───── check digit (1 digit)
  │       └──────────── serial number (8 digits)
  └──────────────────── service indicator (2 letters)
```

The **service indicator** is two letters. The first letter denotes the class of service (for example `E` for EMS, `R` for registered, `C` for insured parcels, `L` for tracked letters, `U` for unregistered), and the second letter is assigned by the originating operator. The **serial number** is eight digits assigned by that operator. The **check digit** is computed from the serial. The final **country code** is the ISO two-letter code of the country that originated the item, which is what lets you route `…US` to the US, `…DE` to Germany, `…CN` to China, and so on.

A single regex matches all of them:

```
^[A-Z]{2}\d{9}[A-Z]{2}$
```

Note that the nine digits in the middle are the eight-digit serial plus the one check digit.

### S10 check digit

The check digit is a weighted modulo-11 calculation over the eight serial digits. Weights are applied left to right:

```
weights = 8, 6, 4, 2, 3, 5, 9, 7
```

Multiply each serial digit by its weight, sum the products, then:

```
C = 11 − (sum mod 11)
if C == 10 → C = 0
if C == 11 → C = 5
```

**Worked example** using serial `47312482` (from the UPU specification):

```
4×8 + 7×6 + 3×4 + 1×2 + 2×3 + 4×5 + 8×9 + 2×7 = 200
C = 11 − (200 mod 11) = 11 − 2 = 9
```

So `RA473124829US` is a well-formed S10 number: service `RA`, serial `47312482`, check digit `9`, origin `US`. The two special cases (result 10 becomes 0, result 11 becomes 5) exist so the check digit is always a single decimal digit.

Because the country code is literally embedded in the number, S10 is the one format where you can identify the *originating country* with certainty even when you cannot pin down the exact operator, since several operators in one country can share the same `…XX` suffix.

---

## USPS

The United States Postal Service uses the **Intelligent Mail package barcode (IMpb)**, which encodes into a tracking number that is typically **22 digits** long, though 20-digit and older variants appear in the wild. Domestic USPS numbers usually begin with a recognizable prefix (`92`, `93`, `94`, or `95`), and numbers printed from certain label systems carry a leading `420` followed by the destination ZIP code, which you strip before reading the tracking portion.

```
^(94|93|92|95)\d{20}$      # common 22-digit IMpb
^\d{20}$                    # 20-digit variant
^420\d{5}(94|93|92|95)\d{20}$   # with leading ZIP routing block
```

USPS also carries international inbound items under the S10 standard, so a number ending in `US` and matching the S10 pattern is very likely a USPS-handled item.

The IMpb tracking number ends in a **MOD 10 check digit**, the same scheme used by GS1 barcodes and credit cards' outer layer. Working from the rightmost digit of the number *excluding* the check digit, weight digits alternately by 3 and 1, sum them, and the check digit is whatever brings the total up to the next multiple of ten:

```
check = (10 − (weighted_sum mod 10)) mod 10
```

For a deeper walk-through of how to read a USPS number, decode its status messages, and spot common tracking scams, InstantParcels has a full guide at [Master US Post Tracking Numbers](https://instantparcels.com/us-post-tracking-numbers) and a live lookup at [instantparcels.com/couriers/usps](https://instantparcels.com/couriers/usps).

---

## UPS

UPS tracking numbers are the easiest of all to recognize: they begin with `1Z` and are exactly eighteen characters long.

```
^1Z[0-9A-Z]{16}$
```

The structure is `1Z`, a six-character shipper account number, a two-digit service level, a seven-digit package identifier, and a final check digit. UPS also issues a handful of other formats, such as a nine-digit "InfoNotice"-style number, numbers beginning with `T` followed by ten digits, and long Mail Innovations numbers that hand off to USPS for final delivery, but the `1Z` form is what you will see the overwhelming majority of the time.

### UPS 1Z check digit

The `1Z` number carries its own check digit over the sixteen characters that follow the `1Z` prefix (fifteen significant characters plus the check digit itself). Letters are converted to digits on a repeating cycle (`A`=2, `B`=3, up to `H`=9, then `I`=0, `J`=1, and so on), which is the same as `(ASCII_code minus 63) mod 10` for an uppercase letter. Then the fifteen significant characters are summed with odd positions weighted once and even positions weighted twice, and the check digit brings the total to the next multiple of ten:

```
value(c)   = c if digit else (ord(c) − 63) mod 10
total      = Σ odd-position values + 2 × Σ even-position values
check      = (10 − total mod 10) mod 10
```

The canonical UPS example `1Z999AA10123456784` passes this check, which makes it a convenient test vector when you implement the algorithm yourself. Track UPS at [instantparcels.com/couriers/ups](https://instantparcels.com/couriers/ups).

---

## FedEx

FedEx is the format most likely to trip up a naive detector, because its numbers are **plain digits with no letters and no reliable prefix**, and they come in several lengths: **12 digits** for many ground and express shipments, **15 digits** for FedEx Ground, and **20 or 22 digits** for numbers derived from a longer barcode. Some FedEx SmartPost numbers begin with `96` and run 20 or more digits, which overlaps with other carriers' ranges.

```
^\d{12}$
^\d{15}$
^\d{20}$
^\d{22}$
```

Because a 12-digit run is not unique to FedEx, treat a bare numeric match as a *candidate* rather than a confirmed identification. When a number could plausibly be FedEx or a postal operator, the honest answer is to present both and let the tracking lookup resolve it, which is precisely the kind of ambiguity a multi-carrier tracker is built to absorb. Track FedEx at [instantparcels.com/couriers/fedex](https://instantparcels.com/couriers/fedex).

---

## DHL

"DHL" is really several different products with different number formats, so it pays to distinguish them.

**DHL Express**, the international air courier arm, uses a **10-digit air waybill number** (occasionally 11). It is purely numeric with no check digit you can rely on externally.

```
^\d{10}$
```

**DHL eCommerce** and DHL Parcel, the lower-cost e-commerce and domestic products, use longer numbers that frequently begin with `GM`, or with `LX`/`RX` on S10-style international pieces, or a long numeric string.

```
^GM\d{16,20}$
^(LX|RX)\d{9}[A-Z]{2}$
```

**Deutsche Post / DHL Paket** in Germany issues 12-, 16-, or 20-digit domestic numbers, and handles inbound international mail under S10 with a `…DE` suffix. Because the DHL family spans air freight, e-commerce, and a national post all at once, a DHL match should always record *which* DHL product you think it is. Track the DHL family at [instantparcels.com/couriers/dhl](https://instantparcels.com/couriers/dhl).

---

## Royal Mail and the S10 postal family

Most national postal operators do not have a bespoke tracked-item format of their own for international mail; they use S10, and you tell them apart by the country-code suffix. This is the single most powerful idea in carrier detection, because one pattern plus a two-letter lookup covers dozens of operators.

| Suffix | Operator |
|---|---|
| `GB` | Royal Mail / Parcelforce |
| `CA` | Canada Post |
| `AU` | Australia Post |
| `CN` | China Post / China EMS |
| `IN` | India Post |
| `JP` | Japan Post |
| `ES` | Correos |
| `IT` | Poste Italiane |
| `FR` | La Poste |
| `NL` | PostNL |
| `DE` | Deutsche Post |
| `SG` | Singapore Post |
| `BR` | Correios |

For any of these, the pattern is the same `^[A-Z]{2}\d{9}XX$` with `XX` swapped for the suffix, and the S10 check digit described above applies. That means for the entire postal family you get *free validation*: if the MOD 11 check digit passes, you can be confident the number is at least well-formed and correctly transcribed, even before you query anything. Browse all supported operators at [instantparcels.com/couriers](https://instantparcels.com/couriers).

Note that many operators *also* run purely domestic formats that are not S10, such as Canada Post's 16-digit domestic number and Australia Post's domestic article numbers, so the S10 suffix identifies international items specifically.

---

## European parcel carriers

Beyond the national posts, Europe has a dense field of private parcel carriers whose formats are mostly plain numeric and therefore mostly ambiguous:

- **DPD**: commonly a **14-digit** number; longer 28-digit forms exist for certain services.
- **GLS**: an **11- to 14-digit** parcel number, and S10-style numbers for cross-border tracked mail.
- **Evri (formerly Hermes UK)**: a **16-digit** number.
- **Colissimo (La Poste)**: a **13-character** number that may mix letters and digits, e.g. beginning `6A`.
- **PostNL**: 3S-prefixed barcodes (`3S` + 9 to 11 characters) alongside S10 for international.

For these, expect shape-based detection to produce *sets* of candidates rather than a single answer. A 16-digit number in a European context could be Evri or Canada Post; without a check digit there is no way to be sure from the number alone. This is not a flaw in your code; it is a genuine property of the numbering schemes, and the correct engineering response is to carry a candidate list forward until a tracking query disambiguates it.

---

## Detecting the carrier from a number

Putting it together, a robust detector runs in three stages: normalize, match by shape in priority order, then confirm with a check digit where one exists.

**Order matters.** Evaluate the specific, unambiguous patterns first (a `1Z…` is unmistakably UPS; an S10 suffix pins the origin country) and fall back to generic numeric patterns last. If you test `^\d{12}$` before you test the structured formats, you will misclassify numbers that merely happen to be twelve digits long.

```python
import re

PATTERNS = [
    ("ups",           r"^1Z[0-9A-Z]{16}$"),
    ("usps",          r"^(94|93|92|95)\d{20}$"),
    ("royal-mail",    r"^[A-Z]{2}\d{9}GB$"),
    ("china-post",    r"^[A-Z]{2}\d{9}CN$"),
    ("dhl-ecommerce", r"^GM\d{16,20}$"),
    ("postnl",        r"^3S[A-Z0-9]{9,11}$"),
    ("s10",           r"^[A-Z]{2}\d{9}[A-Z]{2}$"),  # any international postal item
    ("dpd",           r"^\d{14}$"),
    ("fedex",         r"^\d{12}$|^\d{15}$"),
    ("dhl-express",   r"^\d{10}$"),
    # ... extend as needed
]

def candidates(raw):
    s = raw.upper().replace(" ", "").replace("-", "")
    hits = [name for name, pat in PATTERNS if re.match(pat, s)]
    return hits or ["unknown"]
```

The function returns a *list* on purpose. For structured formats it will usually be a single entry; for bare numeric formats it may return several, and that is the truthful output. If you want a single best guess, prefer the match with a passing check digit, then the most specific pattern.

---

## Validating check digits

Three of the formats here have check digits worth implementing. All three are short, and all three let you reject a mistyped number instantly, before you spend an API call on it.

```python
import re

def s10_valid(tn):
    m = re.match(r"^[A-Z]{2}(\d{8})(\d)[A-Z]{2}$", tn)
    if not m:
        return False
    serial, chk = m.group(1), int(m.group(2))
    weights = [8, 6, 4, 2, 3, 5, 9, 7]
    total = sum(int(d) * w for d, w in zip(serial, weights))
    c = 11 - (total % 11)
    c = 0 if c == 10 else 5 if c == 11 else c
    return c == chk

def usps_valid(tn):
    if not tn.isdigit():
        return False
    body, chk = tn[:-1], int(tn[-1])
    total = sum(int(d) * (3 if i % 2 == 0 else 1)
               for i, d in enumerate(reversed(body)))
    return (10 - total % 10) % 10 == chk

def ups_valid(tn):
    m = re.match(r"^1Z([0-9A-Z]{15})([0-9A-Z])$", tn)
    if not m:
        return False
    body, chk = m.group(1), m.group(2)
    val = lambda c: int(c) if c.isdigit() else (ord(c) - 63) % 10
    total = (sum(val(c) for i, c in enumerate(body) if i % 2 == 0)
             + 2 * sum(val(c) for i, c in enumerate(body) if i % 2 == 1))
    return str((10 - total % 10) % 10) == chk
```

These implementations pass the reference vectors used throughout this document: `s10_valid("RA473124829US")`, `ups_valid("1Z999AA10123456784")`, and `usps_valid("9400111899223197428497")` all return `True`, and flipping any single digit makes them return `False`.

---

## Caveats and limitations

A tracking number tells you less than you would hope, and it is worth being honest about the limits:

A passing check digit proves the number is *well-formed*, not that a shipment exists. Anyone can construct a syntactically valid S10 or UPS number; only the carrier's system knows whether it was ever assigned to a real parcel.

Shape overlaps are real and unavoidable. Plain numeric formats such as FedEx, DPD, Evri, and Canada Post domestic share length ranges, so a purely offline detector cannot always give a single answer. Treat detection as narrowing, not deciding.

Carriers change and extend their formats. New services, acquisitions, and label-system upgrades introduce new prefixes and lengths over time. The patterns here reflect the common formats in circulation but are not a frozen specification, which is why this repository is open to corrections.

Reseller and marketplace numbers may be re-encoded. Items moving through consolidators, marketplaces, or cross-border partners are sometimes re-labeled, so the number a buyer sees may belong to a different carrier than the one that first shipped it.

When you actually need the delivery status rather than just the format, resolve the number against the carrier, or against an aggregator like [InstantParcels](https://instantparcels.com) that queries the right carrier for you regardless of which format the number turns out to be.

---

## Contributing

Corrections and additions are welcome. If you have a format that is missing, a prefix that is wrong, or a check-digit algorithm that should be documented, please open an issue or a pull request with:

- the carrier and, where relevant, the specific product or service,
- at least one structurally valid example (please do not post real tracking numbers tied to real shipments),
- a source or reference for the format where possible.

Accuracy beats coverage here. A confidently wrong pattern is worse than a missing one, because it makes a detector *look* certain while being wrong.

---

## Resources

- [InstantParcels, multi-carrier tracking](https://instantparcels.com): track USPS, UPS, FedEx, DHL and 600+ carriers and postal operators in one place.
- [All supported couriers](https://instantparcels.com/couriers): the full carrier directory, one page per carrier.
- [USPS tracking guide](https://instantparcels.com/us-post-tracking-numbers): reading, decoding, and troubleshooting USPS numbers.
- Per-carrier pages: [USPS](https://instantparcels.com/couriers/usps) · [UPS](https://instantparcels.com/couriers/ups) · [FedEx](https://instantparcels.com/couriers/fedex) · [DHL](https://instantparcels.com/couriers/dhl)
- [S10 (UPU standard) on Wikipedia](https://en.wikipedia.org/wiki/S10_(UPU_standard)): the international postal item numbering standard.

---

*Maintained as an open reference. If it saved you an afternoon of reverse-engineering barcode formats, a star is appreciated.*
