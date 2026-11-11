# Law Firm Case Management System — Data Design - To Be Reviewed

**CSCI 377 · Shafin Awal, Adam Mohamed**

This document explains how we plan to structure the data for our system and why. The goal is a dataset of **1,000,000 cases** that is realistic enough to demonstrate a law firm's operations, consistent enough never to contradict itself, and compact enough to sort, search and store in trees on an ordinary laptop.

---

## 1. Overview

The data is made of five record types. Two are **shared** (one object used by many cases), and two **belong to a single case**:

```
Attorney ──1:many──┐
                   ├──► Case ──1:many──► Party        (victim, opposing party, opposing counsel)
Client ────1:many──┘         └──1:0-or-1──► Settlement
```

| Record | Holds | Shared or per case | Planned count |
|---|---|---|---|
| `Attorney` | name, practice area, hourly rate | Shared | 200 (25 per practice area) |
| `Client` | name, job title, client since | Shared | about 1 per 5 cases |
| `Case` | type, our client's role, filing date, status, amount billed, outcome, closing date | — | 1,000,000 |
| `Party` | name, role, kind (person, company or government) | Per case | 0–3 per case, set by case type |
| `Settlement` | amount, date settled | Per case, often absent | only on settled cases |

All data is synthetic, generated with the Faker library from a fixed random seed, so everyone running the project gets identical data, provided they use the same versions listed under Requirements below.

### Requirements
- **Python 3.10 or newer.** The design relies on `@dataclass(slots=True)` and the `int | None` type syntax, both introduced in Python 3.10.
- **Faker 40.41.0** (pinned in `requirements.txt` as `faker==40.41.0`). A fixed seed fixes the random choices, but Faker's word lists can change between versions, so a different version would produce different names from the same seed.

---

## 2. How the data is structured, compared with the alternatives

To show why we chose our structure, here is **one real case from our generated data**, case 100008, written four ways: the structure we chose, then three common alternatives.

### 2.1 Our structure: linked records

Each case is an object that **points to** shared Client and Attorney objects and **holds** its own list of parties:

```python
rachel = Client(client_id=501217, name="Lynch, Rachel",
                job_title="Pensions consultant", client_since=date(2008, 12, 15))
wagner = Attorney(attorney_id=1162, name="Wagner, Kristi",
                  practice_area="Securities Fraud", hourly_rate_cents=85_000)
SEC = Party("U.S. Securities and Exchange Commission", "Opposing Party", "Government")

case = Case(
    case_id=100008,
    client=rachel,                  # a reference to the ONE Client object
    attorney=wagner,                # a reference to the ONE Attorney object
    case_type="Securities Fraud",
    client_role="Defendant",        # our client is the accused side
    filed=date(2016, 4, 7),
    status="Closed",
    billed_cents=3_010_965,         # $30,109.65, exact
    parties=[SEC,                   # shared by every SEC case
             Party("Walls, Brett", "Government Attorney", "Person"),
             Party("Melendez, Anthony", "Victim", "Person")],
    outcome="Dismissed",
    closed_on=date(2017, 2, 24),
    settlement=None,                # an accused client never receives one
)
```

```
                  ┌──────────► Client: Lynch, Rachel      (shared by all of her cases)
Case 100008 ──────┼──────────► Attorney: Wagner, Kristi   (shared by all of her cases)
                  └── parties ─► SEC                      (shared by every SEC case)
                               ► Walls, Brett  (Government Attorney)   (this case only)
                               ► Melendez, Anthony (Victim)            (this case only)
```

### 2.2 Alternative A: one flat row per case (spreadsheet style)

| case_id | client_name | client_job | attorney_name | attorney_rate | opposing_party | opposing_counsel | victim | outcome | settlement |
|---|---|---|---|---|---|---|---|---|---|
| 100008 | Lynch, Rachel | Pensions consultant | Wagner, Kristi | 850.00 | U.S. Securities and Exchange Commission | Walls, Brett | Melendez, Anthony | Dismissed | |

This is the simplest layout, but:
- **The client's details are copied into every one of their cases.** If Rachel Lynch changes her name, every row must be found and edited; miss one and the data contradicts itself.
- **The columns fix the number of parties.** A case with two victims does not fit, and an estate plan leaves three party columns empty.
- **Nothing in the layout ties fields together**, so an accused client with a settlement, or an open case with an outcome, looks just as valid as a correct row.

### 2.3 Alternative B: nested dictionaries (JSON style)

```python
case = {"case_id": 100008,
        "client": {"name": "Lynch, Rachel", "job_title": "Pensions consultant"},
        "attorney": {"name": "Wagner, Kristi", "practice_area": "Securities Fraud"},
        "parties": [{"name": "Walls, Brett", "role": "Government Attorney"}, ...],
        "status": "Closed", "outcome": "Dismissed"}
```

This handles a varying number of parties, but:
- **Each case usually ends up with its own copy of the client**, with the same name-change problem as Alternative A.
- **Every dictionary carries its own table of field names**, which costs far more memory per record than a fixed-field class.
- **A misspelled field is silently accepted:** `case["stauts"] = "Closed"` creates a new field and leaves the real status unchanged.

### 2.4 Alternative C: separate tables linked only by IDs (database style)

```python
case   = {"case_id": 100008, "client_id": 501217, "attorney_id": 1162, ...}
client = {"client_id": 501217, "name": "Lynch, Rachel", ...}
```

Each fact is stored once, which fixes the copying problem, but:
- **Every use of a client's name needs a lookup first.** Sorting a million cases by client name compares names tens of millions of times. If clients are kept in a list, every one of those comparisons starts with a search through about 200,000 clients, and the sort would effectively never finish.

We do use this layout **on disk** (section 5), because a file cannot hold a reference to an object. When the data is loaded, each ID is converted into a reference once, so the algorithms never have to look anything up.

### 2.5 Side-by-side

| | A. Flat rows | B. Nested dicts | C. IDs only | **Linked records (ours)** |
|---|---|---|---|---|
| A client's details are stored | Once per case | Once per case | Once | **Once** |
| Changing a client's name | Edit every case | Edit every copy | Edit once | **Edit once** |
| Number of parties per case | Fixed by columns | Any | Any | **Any** |
| Reaching a client's name while sorting | One step | One step | A lookup every time | **One step** |
| A misspelled field name | Depends on format | Silently accepted | Silently accepted (if dicts) | **Rejected immediately** |
| Memory per record | Copies of shared data | Field names in every record | Compact | **Compact** |
| Where we use it | Not used | Not used | On disk | **In memory** |

Linked records combine the strengths of the alternatives: like a database, each fact is stored once; like a flat row, any field is one step away. Their only limitation, that they cannot be written to a file directly, is handled by storing IDs on disk and rebuilding the links on load.

---

## 3. Design decisions and why

### 3.1 Each record is a `@dataclass(slots=True)`
We considered plain dictionaries, tuples, named tuples and regular classes. A dictionary stores its field names inside every record, while `__slots__` fixes the fields in advance, so a slotted dataclass uses considerably less memory per record, which matters at a million records. It also turns a misspelled field (`case.stauts = "Closed"`) into an immediate error instead of a silently created extra field. Tuples and named tuples are compact but cannot be changed after creation, which would break the shared references described in 3.4.

### 3.2 Money is stored as whole cents (integers)
Decimal fractions such as 0.1 cannot be represented exactly in binary floating point; in Python, `0.1 + 0.2 == 0.3` is `False`. Over a million billing records those errors add up. Integer cents are exact and sort correctly; we convert to dollars only for display.

### 3.3 Dates are `date` objects, not text
Dates written as text in U.S. style sort incorrectly: `"12/01/2025"` sorts *after* `"11/05/2026"`. Because our trees are keyed by filing date, that would make every date range query wrong without raising an error. `date` objects compare chronologically by design, and on disk we write ISO 8601 (`2025-12-01`), which also sorts correctly as plain text.

### 3.4 Cases point to shared Client and Attorney objects
A client with five cases is stored **once**, and all five cases hold a reference to that same object. Copying the client's details into each case would waste memory and allow the copies to disagree: a name change would have to be repeated on every case. With shared references, a change is made once and every case sees it.

### 3.5 One rulebook decides what each case type contains
Different kinds of cases involve different people and results. A criminal defense case has a prosecutor and possibly a victim, but never a settlement; a personal injury case may settle; an estate plan has no opposing side at all. These rules live in **one table** (`CASE_RULES`) that the data generator reads, rather than being scattered through `if/elif` branches. To change a rule, we change one line.

A case type can have more than one **variant**, one for each side our client can be on. Securities Fraud has two: in half of these cases our client is accused by the SEC, and in the other half our client is an investor suing over fraud. Because the side can differ between cases of the same type, every case records its client's role, and the validation in 3.7 checks that role against the variants its type allows.

| Case type | Our client is the | Other side | Their lawyer | Victim | Can settle | Possible outcomes |
|---|---|---|---|---|---|---|
| Criminal Defense | Defendant | The People of the State of New York | Prosecutor | 75% of cases | No | Dismissed, Acquitted, Plea Deal, Convicted |
| Personal Injury | Plaintiff | Person or company | Opposing counsel | — | $10K–$2M | Settled, Won/Lost at Trial, Dismissed |
| Employment | Plaintiff | Company | Opposing counsel | — | $5K–$1M | Settled, Won/Lost at Trial, Dismissed |
| Corporate | Plaintiff | Company | Opposing counsel | — | $50K–$5M | Settled, Won/Lost at Trial, Dismissed |
| Real Estate | Plaintiff | Person or company | Opposing counsel | — | $5K–$750K | Settled, Judgment, Dismissed |
| Family | Petitioner | Person | Opposing counsel | — | $5K–$500K | Settled, Judgment |
| Securities Fraud (accused) | Defendant | U.S. Securities and Exchange Commission | Government attorney | 50% of cases | No | Dismissed, Consent Judgment, Won/Lost at Trial |
| Securities Fraud (investor) | Plaintiff | Company | Opposing counsel | — | $10K–$3M | Settled, Won/Lost at Trial, Dismissed |
| Estate Planning | Client | — | — | — | No | Completed, Withdrawn |

### 3.6 Everyone else in a case is a `Party` with a role
Rather than separate fields for victim, opposing party and opposing counsel (most of which would be empty on most cases), each case holds a **list of parties**, each tagged with a role. A case can have zero, one or several parties, and new roles can be added later without changing the record's shape. Government parties, such as "The People of the State of New York" or the SEC, appear on every case of their kind, so each is stored as one shared object rather than a new copy per case.

### 3.7 Each case is generated in order, so it cannot contradict itself
Each case is built step by step, with every choice limiting the next:

**case type → our client's side → its rules → parties → status → outcome → settlement**

This guarantees, for example, that an open case has no outcome, that only a "Settled" outcome carries a settlement, that an accused client in a securities case never receives one, and that a case always closes after it was filed. A validation function (`check_case`) re-checks every rule each time the data is loaded and rejects any case that breaks one.

### 3.8 Many cases share a filing date, so each tree node holds a list
Ten years of filings cover only about 3,900 distinct dates, so a million cases means roughly 250 cases per date. A tree storing one case per date would silently overwrite all but one of them. Each tree node therefore holds **every case filed that day**.

### 3.9 Repeated values are stored once
Values like "Open", "Opposing Counsel" or a given filing date repeat hundreds of thousands of times. When loading, we keep one shared object per distinct value instead of creating a new one for every row, which avoids storing the same text hundreds of thousands of times. The same lookups also act as validation: an unrecognized status or role in the data raises an error.

---

## 4. How the data is held in memory

The system keeps three structures that all point to the **same** case objects, so an update made through one is visible through the others:

| Structure | Key | Used for | Cost |
|---|---|---|---|
| Dictionary | Case ID | Finding, updating and closing a case | O(1) on average |
| AVL tree | Filing date | Cases in date order; billing totals for a date range | O(log n) |
| Dictionary of lists | Client ID | A client's full case history | O(1) to find the list |

We use a dictionary for ID lookups because scanning a list is O(n), while a dictionary lookup is O(1) on average. We use a balanced tree, rather than inserting into a sorted list, because inserting into a list shifts every element after the insertion point, which makes each insertion O(n) even though finding the position takes only O(log n).

**Rule:** every operation that adds or removes a case updates all three structures.

---

## 5. Storage on disk

The data is saved as four CSV files. Because a file cannot hold an object reference, links are stored as IDs and rebuilt into references, once, when the data is loaded:

| File | One row per | Links to |
|---|---|---|
| `attorneys.csv` | attorney | — |
| `clients.csv` | client | — |
| `cases.csv` | case | `client_id`, `attorney_id` |
| `parties.csv` | party | `case_id` |

The data is generated once and loaded before any experiment begins, so neither generating nor loading is ever included in a timing measurement.

---

## 6. Measurements

Memory use, generation and load times, and the record counts for the full dataset will be measured on our own machines and added here, together with the experimental results.

---

## AI disclosure

The data design was developed with AI assistance (Claude). All design decisions were reviewed by the team.

---

## Sources

- Python Software Foundation, [Time complexity of operations on built-in types](https://docs.python.org/3/library/time-complexity.html) — list containment and insertion are O(n); dictionary lookups are O(1) on average.
- Python Software Foundation, [Floating-Point Arithmetic: Issues and Limitations](https://docs.python.org/tutorial/floatingpoint.html) — why 0.1 cannot be stored exactly.
- Python Software Foundation, [bisect — Array bisection algorithm](https://docs.python.org/3.13/library/bisect.html) — "the O(log n) search is dominated by the slow O(n) insertion step."
- Python Software Foundation, [dataclasses — Data Classes](https://docs.python.org/3/_sources/library/dataclasses.rst.txt) — the `slots` option (Python 3.10+).
- Python Wiki, [Using Slots](https://wiki.python.org/python/archive/UsingSlots.html) — memory savings and prevention of undeclared attributes.
- Wikipedia, [ISO 8601](https://en.wikipedia.org/wiki/ISO8601) — lexicographic order of ISO dates matches chronological order.

---

## Professor's review

*To be completed by the professor. Please mark any boxes that apply and add notes where helpful; we will act on every item before the next progress meeting.*

**Overall decision on the data design**

- [ ] Approved as planned
- [ ] Approved with the changes noted below
- [ ] Please revise and resubmit

**Section-by-section notes**

| Section | Keep as is | Change | Notes |
|---|:---:|:---:|---|
| 1. Overview (records, planned counts, requirements) | ☐ | ☐ | |
| 2. Structure compared with the alternatives | ☐ | ☐ | |
| 3. Design decisions and why | ☐ | ☐ | |
| 4. How the data is held in memory | ☐ | ☐ | |
| 5. Storage on disk | ☐ | ☐ | |
| 6. Measurements (planned) | ☐ | ☐ | |

**Scope**

| Question | Yes | No | Notes |
|---|:---:|:---:|---|
| Is a dataset of 1,000,000 cases the right scale? | ☐ | ☐ | |
| Are the eight case types appropriate? | ☐ | ☐ | |
| Is the level of realism (parties, outcomes, settlements) appropriate for an algorithms course? | ☐ | ☐ | |
| Should any data be added? | ☐ | ☐ | |
| Should any data be removed or simplified? | ☐ | ☐ | |

**Specific changes requested**

| # | Change requested |
|---|---|
| 1 | |
| 2 | |
| 3 | |

**Anything else we should consider for the experiments, report or presentation?**

| Comments |
|---|
| |
| |

| Reviewed by | Date |
|---|---|
| | |
