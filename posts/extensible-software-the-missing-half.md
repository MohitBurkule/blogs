# Extensible software: the missing half

### LLMs made extensions cheap to write and sandboxes made them safe to run. That is the middle of the problem. The beginning is that users don't know what to ask for, and the end is that everything they write lands in your database.

---

Jeremy Morrell's [*Extensible Software in the age of LLMs*](https://jeremymorrell.dev/blog/extensible-software-in-the-age-of-llms/)
makes an argument I think is correct and under-appreciated: LLMs have collapsed the cost of *writing*
an extension, modern sandboxes have collapsed the cost of *running* one safely, and together those
two collapses reopen a design space the web mostly abandoned. Build a solid, accountable core; let
people extend it in directions you never anticipated.

> "We can build our app as a solid, accountable core, and allow users to safely extend it in many
> directions by having LLMs fill in the missing pieces. We can give our users super powers."

I want to take the thesis seriously enough to argue with it. Three things I think are missing, and
one place where I'd put the sandbox somewhere else entirely.

First, though, it's worth being precise about who "users" means here, because the whole argument
changes depending on the answer.

## Who is actually extending the software

Morrell is explicit that he does not mean developers. His examples are accountants, doctors, lawyers
— people with deep domain knowledge and no intention of becoming programmers. The point is not to
turn users into engineers; it is, in his framing, to make the software fit their needs.

That is the ambitious version of the claim, and it is the one worth engaging with. There is a much
safer version — "let *businesses* buy a platform and hire consultants to extend it" — which is
already a solved and enormous market. Salesforce, ServiceNow, SAP, and Workday all sell exactly that,
and have for decades. The interesting claim is the harder one: **an individual end user, with no
engineering support, changing how their software behaves.**

Hold onto that distinction, because two of the three problems below only appear in the hard version.

## The three-decade-old problem underneath

There is a rich research literature on this, and it is not optimistic in the way the LLM framing is.

End-user programming has been studied since at least Bonnie Nardi's *A Small Matter of Programming*
(1993), and the canonical success story is the spreadsheet: tens of millions of people writing what
are unambiguously programs, without ever calling themselves programmers. Andrew Ko, Brad Myers, and
Margaret Burnett's [survey of end-user software engineering](http://www.cs.cmu.edu/~Compose/Ko2009EndUserSoftwareEngineering.pdf)
counts more end-user programmers than professional ones by a wide margin.

But the same literature identifies the barrier that matters most here. Ko's taxonomy names a
**selection barrier**: the difficulty of discovering *what the system can do* and *which capability
achieves the outcome you want*. Not syntax. Not logic. Knowing what to ask for.

This is the classical requirements problem wearing different clothes. Users do not arrive with a
specification. They arrive with a frustration. The gap between "this report takes me an hour every
Monday" and "add a scheduled job that joins these two tables, filters by region, and emails a PDF"
is precisely the gap that software engineers are paid to close, and it is the part LLMs are *least*
obviously good at, because it requires knowing what is possible in *this* system, not in general.

Morrell's argument implicitly assumes the user shows up with an extension in mind. The literature
says that assumption is where most end-user programming efforts die. A recent line of work
[argues that requirements, not code, are now the final frontier for end-user software engineering](https://arxiv.org/pdf/2405.13708)
— which is exactly right, and exactly the half that the "LLMs write the code" framing skips.

### What that implies for the design

If the selection barrier is the real barrier, then the interesting AI surface is not the code
generator. It is the thing that watches what you do and proposes the extension you didn't know to
ask for.

Concretely: the system observes that you export the same view every Monday, reshape it the same way,
and send it to the same three people. It offers: *"Want this to happen automatically? Here's what it
would do."* You see a preview built from your real data, not a description. You accept, edit, or
decline.

That is a different product than "a prompt box that writes plugins". The prompt box serves the 1% who
already know what they want. Nielsen's [participation inequality](https://www.nngroup.com/articles/participation-inequality/)
— which Morrell nods to — says that is roughly the fraction who will ever author anything.
**Proposal, not authoring, is what reaches the other 99%.** The LLM's job is to convert observed
behaviour into a candidate specification, and then to make the specification legible enough to
approve or reject.

There's a real hazard in this design that deserves stating: a system that proposes features based on
your behaviour is a system that is watching your behaviour, and one that can propose confidently
wrong things. A preview built from real data is the mitigation — you are approving an *outcome*, not
a *description* — but the trust cost is not zero and should not be waved through.

## Where the sandbox should live

Morrell's extensions run server-side, on backend workers — his best-in-class example being
Cloudflare's Dynamic Workers, with capability-based security passing in narrow operations rather than
credentials, in the style of IFTTT handing you `twitter.post_new_tweet()` instead of an API key.

That is the right architecture for a large class of extensions: webhooks, scheduled jobs, data
transforms on ingestion, anything that must run when the user's browser is closed.

But there is a second architecture he explicitly sets aside — he footnotes client-side execution as
needing separate treatment — and I think it deserves more than a footnote, because for a large class
of extensions it is *strictly better*.

**Put the sandbox in the browser. Store the script on the server. Expose the backend as an API.**

The user's extension is a script, versioned server-side like any other user data. When the app loads,
it fetches that script and runs it inside a WebAssembly-hosted interpreter in the page — a JS engine
compiled to WASM, so the guest has its own heap, its own object graph, and no reference to the host's
globals. The extension calls a narrow API surface you define. Network access is whatever your
capability layer grants and nothing else.

It is, structurally, cross-site scripting — with the polarity reversed. XSS is an attacker injecting
code into your page that runs with your users' authority. This is *your user* deliberately injecting
code into their own page, running with strictly less authority than the page itself. The mechanism
that makes XSS catastrophic — code executing in the user's session context — is exactly the mechanism
that makes this useful, once the code is chosen by the user and the authority is bounded.

This is not hypothetical. It is what Figma does. Their
[plugin security post](https://www.figma.com/blog/an-update-on-plugin-security/) describes moving
from a Realms-based shim to **QuickJS compiled to WebAssembly**, and the reasoning is the single most
useful paragraph in the whole discussion: the Realms approach shared one JavaScript VM between host
and guest, so objects from inside and outside the sandbox could be confused with one another. Once
the guest runs in a *different VM* with a *different object representation*, that entire vulnerability
class stops existing. Figma runs plugin logic in QuickJS-on-WASM and plugin UI in an iframe, with a
message boundary between them.

The payoffs of moving execution client-side:

- **Latency.** No round trip. An extension that reshapes what you see runs at interaction speed,
  which is the difference between a feature and a nuisance.
- **Cost.** Compute is the user's. Morrell notes cost models as an open question; this dissolves a
  chunk of it. A `while(true)` burns one tab, on one machine, and your platform does not care.
- **Blast radius.** The extension touches data already in that user's session. It cannot reach
  another tenant's rows because it was never on a machine that could see them. Multi-tenant isolation
  stops being something your sandbox must enforce and becomes something your architecture makes
  meaningless.
- **Data residency.** For regulated data, computing in the client can be the difference between a
  compliance conversation and no conversation at all.

And the honest costs, because they are real:

- **The client is not a trust boundary.** Anything the extension can see, the user can already see —
  fine — but any rule you enforce *only* in the sandbox is advisory. Every write still has to be
  validated server-side as though it came from a hostile client, because it did.
- **Spectre-class leakage** across the WASM boundary is a live concern in a shared address space, and
  it is why cross-origin isolation and the usual browser mitigations are prerequisites, not extras.
- **You now support N browsers** rather than one runtime you control.
- **Long-running and scheduled work still needs a server.** This is a complement to Morrell's
  architecture, not a replacement for it.

The natural design is both: **client-side WASM sandbox for anything in the interaction path, backend
workers for anything that must outlive the tab.** Same extension language, same capability model, two
execution hosts.

## The problem nobody wants to talk about: what this does to your database

Here is the failure mode I think is most likely to actually bite, and it gets almost no attention in
the extensibility discourse.

Extensions that only *read* are easy. Extensions that *write* are where platforms go to die.

The moment users can create arbitrary features, they create arbitrary state. Someone adds a field.
Someone stores a status enum with values you have never seen. Someone encodes a workflow in a JSON
blob. Individually, all reasonable. Collectively, your database stops having a schema in any useful
sense — it has a schema *plus an unbounded set of user-defined extensions to that schema*, and every
analytical question you want to ask now has to account for the second part.

The industry already knows what this looks like, because the industry already built it. The
[entity-attribute-value pattern](https://cedanet.com.au/antipatterns/eav.php) is the standard way to
let users add fields without `ALTER TABLE`, and it is one of the most reliably criticised patterns in
data modelling. Attributes become dynamically typed and need casting to compare. Filtering on *n*
attributes means *n* joins. Aggregation needs pivots. The database's type system, constraints, and
query planner — the things you were paying for — stop applying to precisely the data your users care
most about. There are
[documented multi-million-dollar failures](https://novicksoftware.com/wp-content/uploads/2016/09/Entity-Attribute-Value-EAV-The-Antipattern-Too-Great-to-Give-Up-Andy-Novick-2016-03-19.pdf)
attributed to it.

Modern platforms mitigate rather than solve. Typed JSON columns keep values in one row and let you
index specific paths. But you have moved the problem, not removed it: you still cannot answer "how
many customers are in state X" across tenants when every tenant defined "state" differently.

Salesforce is the honest case study, because it has run this experiment at scale for twenty years.
Its answer is not "let users write whatever they want". Its answer is
[governor limits](https://www.salesforceben.com/what-are-salesforce-governor-limits-best-practices-examples/):
hard, non-negotiable, runtime-enforced ceilings — 100 queries per transaction, 150 DML statements, 10
seconds of CPU, 6 MB of heap. Exceed one and your code is killed, in production, with no appeal. That
is what shared-substrate extensibility actually costs, and Morrell's resource-limit discussion is the
same insight applied to compute. **The unsolved half is applying the same discipline to *schema*.**

What I think that looks like:

- **Extensions declare their state, and the declaration is the contract.** Not "here is a blob" but
  "this extension owns a table with these typed columns". The platform provisions it, migrates it,
  and — critically — can *see* it. A declared shape is analysable; an opaque blob is not.
- **Writes to core entities go through the same validation as first-party code.** If a field is
  constrained for your code, it is constrained for extension code. The sandbox restricts what the
  extension *can run*; only the server can restrict what it is *allowed to persist*.
- **Namespacing that survives analytics.** Extension state lives somewhere a query can find it,
  labelled by extension and version, so "what did users build?" is answerable — and so is "which
  extension wrote this row?" when something looks wrong.
- **Semantic reconciliation as a first-class feature.** When two hundred tenants each invent a
  "priority" field, something has to map them onto a shared vocabulary for cross-tenant analysis.
  This is a genuinely good use of an LLM — fuzzy, judgement-heavy, human-reviewable, and exactly the
  work nobody has time to do by hand.

The last one is the interesting inversion. The AI's job is not only to help users *create*
extensions. It is to continuously make sense of what they created, so that user-authored state stays
legible to the platform rather than becoming a growing blind spot.

## The uncomfortable part

There is a tension between the two halves of this argument that I don't think resolves cleanly, and
I'd rather name it than paper over it.

The case for extensibility is that you cannot anticipate what users need. The case for schema
governance is that you must constrain what they can store. Every constraint you add to keep the data
analysable narrows the space of extensions people can build — and the valuable extensions are
disproportionately the ones you didn't anticipate, which are disproportionately the ones your
constraints will reject.

Governor limits are a real cost, not a free lunch. Ask anyone who has fought them. Salesforce's
extensibility is genuinely bounded by them, and the bound is sometimes exactly where the useful thing
lived.

I don't think there is a general answer. I think the honest position is that **read-extensibility and
write-extensibility are different products with different risk profiles**, and most platforms should
ship the first long before the second. Let people transform, visualise, filter, summarise, and
route — all of which are pure functions over data they can already see, all of which are safe in a
client-side sandbox, and none of which threaten your ability to understand your own database. Then
add narrow, declared, typed write capabilities where the demand is loud enough to justify the
governance work.

## Where I land

Morrell's thesis holds. The cost curves genuinely moved, and there is a real opportunity here that
was not available five years ago.

But "LLMs write the extension, a sandbox runs it safely" is the middle of the problem, not the whole
of it. The beginning is that **users do not know what to ask for**, which is a thirty-year-old
finding and the reason most end-user programming stays in spreadsheets. The end is that **anything
users write, you have to live with**, in a database you still need to be able to reason about.

The sandbox is the easy part. It is the part with vendors, benchmarks, and a clear success criterion.
The requirements problem and the schema problem have none of those, which is exactly why they're
where the work is.

---

*Written after reading Jeremy Morrell's post, which is worth reading in full — it is more careful than
most writing in this space and it is explicit about its own gaps, which is what made it worth
arguing with.*
