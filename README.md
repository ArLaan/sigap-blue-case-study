# Building a card-inventory system for a government vehicle-inspection unit

A four-month case study: designing, building and shipping a stock-tracking system that
replaced a paper-and-spreadsheet process at Dinas Perhubungan (Dishub) — a provincial transport agency in Jakarta.

Built solo, in production, on a runtime I could not choose.

> **Note on the code.** This repository contains the write-up only. The application
> itself belongs to Dishub and handles records for real vehicle-inspection cards,
> so it is not public. Everything below describes decisions and trade-offs, not
> deployable code.

---

## The problem

Dishub issues **BLUe cards** — roadworthiness certificates for commercial
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

## Four decisions worth explaining

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

Cards carry official codes issued by the provincial government (Dishub), in a fixed structured
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

### 4. Undoing an intake by deleting data on purpose

Every other correction in this system is append-only. Sending a box to the wrong counter
is reversed by writing a second, opposite movement — the ledger then shows both the
mistake and the correction, which is what an audit trail is for.

Cancelling an *intake* could not work that way, and it took me a while to accept why.

When a clerk mis-scans a delivery, the system registers a range of serial numbers as
received. A guard prevents the same physical cards ever being registered twice — which is
correct, and which also means a mis-scan is **permanent**. Those cards can never be
entered correctly. The guard cannot tell a duplicate from a correction.

So the fix has to actually release the serial range, and that means destroying the
records the bad scan created. There is no version of this that only adds rows.

What I kept is the receipt: the original record survives, marked cancelled, carrying who
cancelled it, when, and why — and the reason is mandatory, because once the card records
are gone that receipt is the only evidence left. Two guards keep it narrow: a box can only
be cancelled while every card is still exactly where the intake put it, *and* while it has
no movement history at all. A box that went out to an office and came back looks untouched
by the first test and fails the second — it has been in circulation, so its intake was
never the mistake.

**What I learned:** "never delete" is a good default, not a law. The question is what the
record is *for*. Here the audit trail needed to survive; the operational rows did not, and
keeping them would have preserved evidence of something that never validly happened.

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

## The bug that seven reviews missed

The cancellation feature went through a task-by-task review — seven pieces, each one read
and checked before the next was written. Every piece passed. A final review of the whole
branch, read as one change, found something none of them could have.

The design document contained an audit: every place in the code that reads the intake
records, and what each one needed to do about a cancelled one. It was thorough. It was
also, entirely, an audit of the **application code** — and I had carried that framing into
every piece of the work.

The blocker was in the database. Two columns on that table carried uniqueness constraints
dating from the original build. A cancelled record keeps its box code and its first serial
number, because that record is the receipt. So the corrected re-scan would pass every check
I had written, reach the database, and be rejected there for duplicating the receipt.

The clerk would see *"Failed to save, please try again"* — advice that could never succeed.
And by that point the cancellation had already deleted the card records. The cards would be
**gone and still un-enterable**: worse than if the feature had never been built.

Fixing it meant relaxing a database constraint, which is the sort of change that should
make you uncomfortable. What made it acceptable is that the application now enforces the
same rule more strictly than the constraint ever did — it compares whole serial *ranges*,
catching overlaps a single-column constraint cannot see. What it does not do is survive two
clerks submitting at the same instant, so that gap is written down in the project's risk
register rather than left to be rediscovered.

I confirmed the constraints on the live database before changing anything, rather than
trusting the schema file in the repository.

**What I learned:** an audit is only as wide as its framing. Mine said "find every query"
when it should have said "find everything that can reject this write." Reviewing pieces
catches piece-sized bugs; something has to read the whole thing.

---

## What the system does now

- **Intake** — scan an official carton or box code; the system parses it, checks the
  serial range does not overlap anything already registered, and generates one record
  per individual card.
- **Correcting an intake** — a mis-scanned delivery can be retracted, releasing its serial
  range so the cards can be entered correctly. The retraction is permanent, requires a
  written reason, and is refused for any box that has already moved. A carton registered
  as one unit is retracted as one unit.
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
