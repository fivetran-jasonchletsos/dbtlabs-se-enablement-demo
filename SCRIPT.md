# dbt Labs SE Enablement — "It's Never Just Three Clicks"
Speaker: Jason Chletsos (Fivetran SE) | Audience: dbt Labs SEs | Time: 30 min
Follows Niraj's live database-connector demo.

Live in the Fivetran UI for ~80% of this. The leave-behind (`index.html`) is a
reference doc for you and something to forward afterward — not something you
present from.

---

## The setup (don't skip this, it's the whole point)

Niraj just showed a database connector: point at a host, give it credentials,
Fivetran reads the schema off the wire in seconds, done. That's real, and
it's genuinely one of Fivetran's strengths. The risk is that everyone in the
room now believes *every* connector works that way, and if ingestion is
"three clicks and done" for a database, the SaaS/API side must be trivial
too — and by extension, all the real engineering value sits downstream in
transformation, which is where dbt Labs SEs live.

That's the assumption to dismantle. Not "Fivetran is hard to use" — the UI
stays simple on purpose. What's underneath a SaaS or HR-system connector is
a different category of problem than reading a database's information
schema, and as NewCo, you're going to be selling the whole pipeline
(Connections -> Transformations -> Activations) as one story. If the room
believes Connections is a solved, boring problem, they'll undersell it and
overpromise on what a customer's data actually looks like by the time it
reaches dbt.

**Say this out loud near the top:** "A database exposes a schema. An API
exposes an opinion about how its product works. Those are not the same
problem, and the UI hides that on purpose — that's the point of good
engineering. I want to show you what's underneath it."

---

## 0:00–0:03 — Framing (slides/talk, no UI yet)

- Contrast the two mental models: **schema discovery** (database) vs.
  **API negotiation** (SaaS application).
- Preview the two examples: Salesforce (an API everyone in the room thinks
  they already understand) and Workday (an API where there usually isn't
  even a schema to discover until a customer builds one).
- One line to land before moving to the UI: "By the time you see a green
  checkmark on a Salesforce connector, Fivetran has already made about a
  dozen decisions on the customer's behalf. I'm going to open the hood on
  six or seven of them."

---

## 0:03–0:22 — Live in the Fivetran UI: Salesforce

Set up the connector live (or resume one already authorized so you don't
burn the whole slot on OAuth redirects) and narrate each of these as you
hit it. Suggested order follows the actual setup flow, not importance —
that's deliberate, it shows the complexity is front-loaded before a single
row of data moves.

### 1. Authorization isn't "a password"
A database connector wants a host, port, user, password. Salesforce wants
an OAuth flow against a connected app, and in real deployments that means
the customer's admin has usually pre-provisioned:
- A dedicated integration user with a specific **profile/permission set**
  (not an admin's personal login — that breaks the moment the admin
  leaves or changes roles).
- **IP allowlisting / login IP ranges**, sometimes conflicting with
  Fivetran's published IP ranges.
- API access enabled on that profile at all — plenty of orgs have API
  access turned off by default per profile.
Talking point: "In a database, the credential *is* the access model. In
Salesforce, the credential is a pointer into a permission system that's
already been configured — or hasn't."

### 2. There is no fixed schema — there's an object graph
Click into the schema/objects picker. Show how many standard objects
exist before a single custom object is added, then point out:
- **Custom objects and custom fields** — every org's schema is
  functionally unique. Fivetran can't ship a fixed schema for "Salesforce"
  the way it can for "Postgres."
- **Lookup and master-detail relationships** across dozens of objects,
  including junction objects that exist purely to model many-to-many
  relationships.
- **Record types** — the same object (say, Opportunity) can behave like
  several different logical entities depending on record type, with
  different page layouts and required fields upstream, but one physical
  table.
Talking point: "A `CREATE TABLE` statement is a contract. A Salesforce org
is a living configuration that a customer's admins reshape every release
cycle."

### 3. Formula and roll-up summary fields aren't "just columns"
Find one in the schema (Opportunity or Account usually has one). These are
computed server-side by Salesforce, not stored the way a normal field is,
and:
- They can't be filtered on on the API the same way as native fields in
  older API versions.
- Roll-up summary fields recalculate based on child-record changes, which
  historically has **not** reliably bumped the parent record's
  `SystemModstamp` — meaning an incremental sync keyed purely on
  modification timestamp can miss the fact that a roll-up changed.
Talking point: "This is the kind of bug that doesn't show up in a demo. It
shows up eight months into production when a customer asks why their
rolled-up pipeline number in the warehouse doesn't match what's on the
Opportunity page in Salesforce."

### 4. Deletes are a different code path entirely
Salesforce soft-deletes into a recycle bin (`IsDeleted` flag) rather than
actually removing the row, and standard queries silently exclude deleted
records. Capturing deletes correctly requires the connector to use
`queryAll`-style semantics or the dedicated deleted/updated endpoints —
whichever object type is in play. Miss this and a customer's warehouse
silently accumulates records that were deleted in Salesforce months ago.
Talking point: "A database DELETE is a DELETE. Salesforce has an entire
parallel universe of 'gone but not gone,' and every object type handles it
slightly differently."

### 5. Incremental sync is timestamp-chasing, not log-reading
No WAL, no binlog, no CDC stream. Incremental extraction is built on
querying by `SystemModstamp` windows (or object-specific change-data
endpoints where available), then reconciling. Show the sync history /
schema page and talk through:
- Historical objects like `Task`, `EmailMessage`, `LoginHistory`, or
  `EventLogFile` can run into the billions of rows for a large org —
  historical sync isn't a "few minutes," it's sized in days.
- The **Bulk API v2** vs. REST API decision Fivetran makes per object
  based on volume, and that this decision has to be re-evaluated as the
  object grows.
Talking point: "Databases replicate. Salesforce has to be *asked* for what
changed, over and over, forever, and the answer is different depending on
which of forty-something objects you're asking about."

### 6. API call budget is a shared, finite resource
Salesforce orgs get a **daily API call allotment** tied to license edition
and user count — and every other integration the customer runs (Marketing
Cloud, a CPQ tool, internal scripts) draws from the *same* budget. Point
out that Fivetran has to:
- Batch and backoff intelligently so it doesn't starve the customer's
  other integrations.
- Size sync frequency against a shared, capped resource that a DBA never
  has to think about with a database connection.
Talking point: "In a database you can hurt performance if you're careless.
In Salesforce you can get the customer's *other vendors* rate-limited if
you're careless. It's a shared commons problem, not a performance-tuning
problem."

### 7. The data itself doesn't map 1:1 into warehouse types
Show a multi-select picklist, a currency field in a multi-currency org, or
a compound field (geolocation, address) if the org has one:
- Multi-select picklists are semicolon-delimited strings that need
  decisions about how to land — flatten, junction table, array type.
- Multi-currency orgs carry a currency code per record plus historical
  conversion rates — "amount" isn't one number, it's a number plus context.
- Picklists can be renamed/reordered by admins without changing the
  underlying API name, which is its own schema-drift problem.
Talking point: "None of this is a database type-mapping problem. It's a
semantic modeling problem, and Fivetran is making that call so the
customer's dbt models don't have to start from a `VARCHAR` blob."

**Close the Salesforce section:** "Every one of those seven things happens
before dbt ever sees a row. That's not Fivetran hiding complexity from the
customer — it's Fivetran absorbing complexity so the customer's dbt
Wizard models start from something coherent instead of raw API sludge."

---

## 0:22–0:27 — Live in the Fivetran UI (or slides if no sandbox tenant): Workday

Pivot hard: "Salesforce at least *has* an API you can introspect. Workday
frequently doesn't even give you that much."

- **Reports-as-a-Service (RaaS)** — Workday's most common integration
  pattern isn't "call an endpoint for the Worker object." A Workday admin
  has to go into **Workday Studio**, build a **custom report**, define its
  fields and filters, and publish it as a web service before Fivetran (or
  anyone) has anything to connect to. Talking point: "With Salesforce,
  Fivetran discovers a schema. With Workday, someone has to *author* one
  first, by hand, inside Workday, before the word 'connector' means
  anything."
- **Authentication is tenant-specific and often SOAP-based** — Integration
  System Users (ISU), tenant URLs that differ by environment
  (implementation/sandbox vs. production), and credentials that are
  scoped per integration, not per user.
- **No real CDC** — most RaaS reports are full-snapshot pulls. "Incremental
  sync" downstream often means Fivetran/dbt has to do the change-detection
  work that Salesforce's `SystemModstamp` at least attempts to do natively.
- **The data is HR and compensation data** — sensitivity, field-level
  masking, and data residency constraints are part of the connector
  conversation from minute one, not an afterthought.
- **Every customer's "Workday connector" is bespoke** — it's really a
  contract against a report someone built. If that report definition
  changes in Workday, the pull can break silently with no schema-drift
  signal the way a new Salesforce field would show up.

Talking point to land it: "If Salesforce is 'the API has opinions,'
Workday is 'there is no API until a human builds one.' Two completely
different engineering problems, two completely different connectors, same
five-minute-looking setup screen."

---

## 0:27–0:30 — Close

- Tie back to NewCo: the reason this matters isn't to make Fivetran look
  hard to use — it's so the room stops mentally filing Connections as
  "the solved, boring part" and Transformations as "the real work." The
  pitch to a prospect is one pipeline: Connections absorbs this mess,
  Transformations (dbt Wizard) works because the mess was already
  absorbed, Activations sends it back out. If any SE in the room — Fivetran
  or dbt Labs side — undersells the ingestion layer, they're setting up
  their own transformation story to look worse than it is, because a
  customer will ask "why doesn't my dbt model look clean" and the honest
  answer is usually further upstream than dbt.
- Leave time for questions — this group will likely ask about NetSuite,
  Marketo, or another SaaS source they've hit in the field. You don't need
  the answer memorized; the Salesforce framework (auth model, object
  graph, deletes, incremental strategy, rate limits, type mapping) is a
  checklist that applies to basically any SaaS connector, so you can reason
  through whatever they name live.
- Forward `index.html` after the session as the reference doc.

---

## Pre-flight checklist (do this the night before, not in the room)

- [ ] Confirm you have a live Salesforce connector (or sandbox org) already
      authorized in Fivetran so you're not fighting an OAuth redirect live.
- [ ] Confirm the org has at least one custom object, one formula/roll-up
      field, and ideally a multi-currency or multi-select picklist field to
      point at.
- [ ] If you don't have a live Workday tenant, have 2-3 screenshots of the
      RaaS setup screens ready as a fallback — don't try to provision a
      Workday sandbox for a 5-minute segment.
- [ ] Time-box yourself: set a phone timer at 0:22 so the Workday section
      doesn't get cut for time. It's the section that lands the "totally
      different paradigm" point hardest.
