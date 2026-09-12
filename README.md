# Building a card-inventory system for a government vehicle-inspection unit

A four-month case study: designing, building and shipping a stock-tracking system that
replaced a paper-and-spreadsheet process at DINAS PERHUBUNGAN in Jakarta.

Built solo, in production, on a runtime I could not choose.

> **Note on the code.** This repository contains the write-up only. The application
> itself belongs to the agency and handles records for real vehicle-inspection cards,
> so it is not public. Everything below describes decisions and trade-offs, not
> deployable code.

---

## The problem

A government unit issues **BLUe cards** — roadworthiness certificates for commercial
vehicles. Cards arrive from the provincial government in sealed cartons, move through a
warehouse, then an office, then to service counters, and are finally handed to vehicle
owners.

Every step of that was tracked on paper and in spreadsheets. Nobody could answer "where
is card number 41042 right now" without walking to a filing cabinet. Stock counts drifted.
Reconciliation was manual.

The job was to make the system answer that question, for every card, at any time.

---

## The constraints, and why they mattered

None of these were chosen. All of them shaped the design.

| Constraint | Consequence |
|---|---|
| **PHP 5.6, CodeIgniter 3** | No modern language features. No Composer in practice. Every dependency vendored by hand. |
| **MySQL 5.5** on the server | A 2010 release. No `DEFAULT` on TEXT columns, no auto-timestamp on `DATETIME`, **one** auto-timestamp column per table. |
| **No admin access to the server** | Every database change had to be a portable SQL file someone else would run. I could not log in and fix it. |
| **VPN-only access, manual file upload** | No git remote, no CI, no rsync. Deployment was copying files one at a time over SFTP. |
| **Live users from day one** | The system went live before it was finished. Every later change had to be backward-compatible with data already in it. |

The MySQL 5.5 constraint is the one I underestimated. It did not surface until a
stakeholder meeting, two months in, when the database import failed on the server and
nobody could say why.

---

## Three decisions worth explaining

### 1. Anchoring everything to the physical box

**First attempt:** treat stock as a pool of loose cards. A request for 300 cards would
draw from whatever was available.

That modelled the database well and reality badly. Cards do not move individually
through a warehouse — they move in sealed boxes of 100, and a clerk carrying a box to
the office carries all 100 or none. The pool model let the system record states that
could not physically exist, like 40 cards of one box in the office and 60 still in the
warehouse.

**Rebuilt it box-anchored.** A request is counted in *boxes*. A box moves intact or not
at all. The rule became: a box is "in location L" only if *every* card in it is in L.

That one rule eliminated an entire category of impossible state, and it made later
features simpler rather than harder — because the invariant held, code downstream never
had to ask "but what if the box is split?"

**What I learned:** the schema should model the physical process, not the other way
round. Reversing this cost a week. Getting it right first would have cost a
conversation.

### 2. The two-tier box, discovered by asking

Boxes arrive as a **master carton** containing three inner boxes of 100 cards each. The
first version of intake let a clerk register one box at a time, which meant three
separate entries for one physical delivery, and no record that those three belonged
together.

This did not come from a requirements document. It came from asking how the boxes
actually arrive.

Intake now registers a whole carton in one action, creating three linked inner-box
records. Each inner box can then be tracked independently once it is physically opened.

### 3. Scanning instead of typing

Cards carry official codes issued by the provincial government, in a fixed structured
format encoding a serial range, a quantity, and whether the code names an outer carton
or an inner box.

Originally an operator typed a placeholder code by hand. That is a data-entry error
waiting to happen on a record with legal weight.

I wrote a parser for the official format and rebuilt three flows — intake, warehouse
fulfilment, and counter distribution — around scanning the physical box. The operator
scans; the system identifies exactly which box that is and moves exactly those cards.

The subtle part: a carton code and an inner-box code can describe *the same physical
cards*. No uniqueness check catches that, because the two code strings differ. The only
thing that catches it is testing whether the **serial ranges overlap** — which is now
the single guard preventing the same cards being registered twice.

---

## A mistake I made, and how it was caught

The production server was misconfigured in a way that would show PHP stack traces —
file paths, SQL, framework internals — to any visitor. I wrote a fix so the application
would detect its own environment and default to production mode unless it could prove it
was running on a developer's machine.

**The fix introduced a worse bug than the one it solved.**

I detected "am I local?" by checking the HTTP `Host` header. But that header is supplied
by whoever makes the request. Anyone could send `Host: localhost` and flip the live
server into development mode — turning the stack traces *on*, remotely, at will.

An automated review flagged it. My first correction was also wrong: I tightened the
check to an exact string match, which an attacker defeats by sending the exact string.
I proved that to myself with a one-line `curl` before going further.

The working fix uses only signals a client cannot set — the PHP server API, and the
address the web server itself bound to — and it is covered by tests that specifically
attempt the bypass.

**What I learned:** request data is not a fact about your server. And when you are
wrong twice in a row about the same thing, stop reasoning and go reproduce it.

---

## What the system does now

- **Intake** — scan an official carton or box code; the system parses it, checks the
  serial range does not overlap anything already registered, and generates one record
  per individual card.
- **Warehouse → office** — a request-and-approval flow, fulfilled by scanning the boxes
  physically handed over rather than by the system picking for you.
- **Office → counter** — the same scan-driven model.
- **Returns** — per-card, for damaged or unused cards.
- **Card tracking** — search any serial and see its full history.
- **Corrections** — any movement can be undone, with the reason and the person recorded.
- **Role-based permissions** — editable in the UI, not hardcoded.
- **Dashboard** — stock position by location, movement trends.

---

## Engineering practices I held to

**A decision log.** Every working session recorded: what changed, why, what was rejected
and for what reason. Four months and roughly 1,200 lines of it. When a decision was
questioned two months later, the reasoning was still there — including the wrong turns.

**Tests where a bug is expensive.** No framework, no CI — just standalone scripts run
from the command line. They cover the code-format parser, the stock-movement guards, and
the environment detection described above: the places where a bug silently corrupts a
record or exposes the server. Not the places where a bug is merely visible.

**A risk register.** Known problems written down with cause, consequence and mitigation,
reviewed at the start of each session. Some are still open. Writing them down is what
stops "we should fix that sometime" from evaporating.

**Backward-compatible migrations.** Widening a column is safe; code expecting a column
that does not exist yet is not. Schema changes ran before the code that needed them,
every time.

---

## What I would do differently

**Initialise version control on day one.** Git came late. Weeks of early work exist only
as a narrative in the decision log.

**Ask how the physical process works before designing the schema.** Both the box
anchoring and the two-tier carton were rebuilds that a single conversation would have
prevented.

**Verify the target environment early.** The MySQL 5.5 incompatibility could have been
found in week one with one import test. It was found in month three, in a meeting.

**Separate configuration from code from the start.** Production settings and a real
encryption key ended up committed to version control before that was corrected — the
sort of thing that is trivial to prevent and tedious to undo.

---

## Stack

PHP 5.6 · CodeIgniter 3 · MySQL 5.5 / MariaDB · Bootstrap 5 (vendored) · Chart.js ·
plain JavaScript, no build step

---

*Written up from a working decision log kept throughout the project. Implementation
details, infrastructure and schema are omitted deliberately: the system is live and
holds government records.*
