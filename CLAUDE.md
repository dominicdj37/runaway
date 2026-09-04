# Runway — context for Claude Code

## What this is

A personal debt and reserve tracker for one household (Daniel, and whoever he
invites — currently just one other person). It answers one question: **which
month does the money run out**, and how does that move if he forecloses a
given debt.

Not a budgeting app, not a product. One shared dataset per household, viewed
and edited by every member of it — see the sharing model below.

## Hard constraints — do not change without asking

1. **No personal figures in the repository.** `index.html` must ship empty.
   Real numbers live in `seed.json` (git-ignored) and in Firestore. Never
   hardcode a balance, salary, or loan into committed source. The repo is public
   because GitHub Pages requires it on the free plan.
2. **No build step.** One HTML file, Firebase v10 modular SDK from CDN, vanilla
   JS. No npm, no bundler, no framework. Committing and enabling Pages is the
   entire deploy.
3. **Firestore rules are the only access control.** The Firebase API key is
   public by design and grants nothing; `firestore.rules` restricts reads and
   writes on `accounts/{accountId}` to its current `members`, and gates joining
   on a verified-email invite. Don't loosen this, and don't hand-roll access
   checks in `index.html` — the rules are the real boundary, the UI is not.
4. **Money is arithmetic, not vibes.** Any change to projection logic gets
   checked against a worked example before shipping. Rounding errors here cost
   the user real money.

## Architecture

- `index.html` — everything: markup, styles, logic. Google sign-in, no backend
  beyond Firestore (no Cloud Functions — that would break the no-build-step
  constraint).
- `seed.json` — the real dataset, imported through the UI once.
- Debounced autosave (700ms) on every edit.
- An offline mode ("Explore without saving") runs with no Firebase configured.

### Sharing model

Three Firestore collections:

- `accounts/{accountId}` — the actual data (`settings`, `debts`, `house`,
  `log`), plus `owner`, `members` (uids), `memberInfo` (display info),
  `invitedEmails`. `accountId` is always the creating owner's uid.
- `users/{uid}` — a pointer doc, `{accountId}`. Tells a signed-in user which
  account's data to load. Written once, at account-creation or invite-accept
  time; never changes after.
- `invites/{email}` — one doc per pending invite, keyed by the *lowercased
  invitee email*, not an auto-id. That's deliberate: the invitee looks up
  their own invite with a single `getDoc` on a known path, so no Firestore
  `list`/query permission is ever needed on the collection. Only the account
  owner can create or delete these; only the matching signed-in email can flip
  `status` from `pending` to `accepted`/`declined`.

Join sequence (`acceptInvite` in `index.html`): accept the invite doc first,
*then* add yourself to the account's `members`/`memberInfo`. The second step's
rule is only satisfied if an `accepted` invite for your email already points
at that account — so order matters, and both steps run from the client with
no server code in between.

Currently: full read/write for every member once joined (no read-only tier),
invites are owner-only (a member can't invite further members).

### Projection

For each month `i` of the horizon, `monthSpend(i)` produces the outgo and
`project()` turns it into a balance:

```
emi     = Σ debts where !foreclosed && i < months   (override ?? emi)
fixed   = Σ commitments where !variable && i < months   (override ?? amt)
varUsed = Σ commitments where variable && i < months: max(spent, cap)
          (or log[month].actual, if legacy data still sets it)
cardTop = Σ per-card spend logged on top of that card's EMI bill
extras  = Σ that month's extra spends
outgo   = emi + fixed + varUsed + cardTop + extras
cashIn  = Σ cash where (type=='oneoff' && i==monthIndex(from))
              || (type=='recurring' && i >= monthIndex(from))
balance += (salary + cashIn + extraIncome) − outgo
if log[month].checkpoint is set: balance = checkpoint   // re-anchor
```

`monthSpend(i)` is the single source of truth: the Monthly spends UI and
`project()` both read it, so the card and the forecast cannot disagree. A
month nobody has touched has no overrides and no logged entries, so every
term collapses to the budgeted figure and the arithmetic is identical to
what it was before that section existed — there are node checks for exactly
this in the scratchpad pattern described under "Money is arithmetic".

`settings.salary` (labelled "Salary 1" in the UI) is the one fixed recurring
income. Everything else — a second salary, a spouse's income, a one-time
top-up — lives in `cash[]`, each entry `{n, amt, type, from, to}`: `type` is
`'recurring'` (counts every month from `from` onward, through and including
`to` if set, or with no end date if `to` is null/absent) or `'oneoff'`
(counts exactly once, in the month named by `from`; `to` is unused).
`monthIndex(from)` converts a `YYYY-MM` string to a horizon-relative index
the same way `mKey`/`mLabel` go the other direction. A `spouse` field used to
live directly on `settings`; `migrateSpouseIncome()` carries any real figure
found there into a `cash` row (tagged `migratedSpouse: true`) the first time
older data loads, so a schema change never silently drops real income — see
the hard constraint above about money being arithmetic, not vibes.

The Cash In section (right after the overview) has its `h2`/`.sub` heading
flush left like every other section on the page — not inside the card,
which would indent it relative to the rest of the page. Below that, one
glass card (`.cashcard`) is split into two tracks by a vertical divider
(`.cashdivider`): left lists Reserve, Salary 1, and every `cash[]` entry as
glass cards
(`.srccard`) — clicking one opens a glass-themed `<dialog>` (`#cashdialog`)
to edit it; Reserve/Salary only expose an amount, cash entries expose label,
amount, type, start and optional end. Right is a month-by-month list
(`#cashflow`) showing, per month, a 100%-stacked bar of that month's money
in hand split into reserve carried over (`r.open`, floored at 0), Salary 1,
and other cash (`cashIn + extraIncome`, floored at 0), with the actual
rupee amount per segment spelled out underneath (`.mflowbreak`) and on
hover (`title` on each `.seg`) — it never shows negative, since it conveys
inflow composition, not the runway balance. The current month (`r.i===0`)
gets an outline. `project()` exposes `open` (opening balance before that
month's flow), `salary`, `cashIn` and `extraIncome` per row specifically so
this chart can be built without recomputing the breakdown.

The right list's height is capped to match the left column's own
(unstretched) height, not the other way round — `syncCashHeights()`
measures `.cashleft`'s `offsetHeight` and sets `#cashflow`'s `max-height`
to it (minus the right side's own header height) after every `render()`
and on window resize. This exists because CSS Grid's `auto` row sizing has
no clean way to say "size this track to my sibling, then let me scroll
internally" — an unbounded `overflow:auto` list still contributes its full
content height to the row, which stretched the *shorter* column and left
blank space under it instead. Below 760px the two tracks stack, the divider
hides, and `#cashflow` falls back to a fixed CSS `max-height` instead (JS
resets its inline height to nothing at that width). See
[[arch-glass-dialog-centering]] for a related CSS-vs-JS lesson from the
same section.

### Monthly spends

The section right after Cash In. A scrollable tab bar (`#monthtabs`, one
`.mtab` per horizon month, a dot where that month has data) selects
`spendMonth`; `#spendcard` is one glass card holding four blocks in the
order the money is actually spent — Commitments, Card spends, Debts, Extra
spends — plus a balance checkpoint and note at the foot.

A month record lives at `D.log[key].spend` and stores only the *differences*
from the plan: `fixed{rowId:amt}` and `caps{rowId:amt}` (per-month amount
overrides), `items{rowId:[{n,amt}]}` (the individual spends logged against a
variable budget), `emis{rowId:amt}`, `cards{cardLabel:onTop}` and
`extras[{n,amt}]`. Anything absent falls back to the budget. Overrides apply
to that month only — the `.ovr` tag shows the planned figure and resets on
click.

Three rules that carry meaning:

- A variable budget counts `max(spent, cap)`, not `spent`. A half-logged
  month must not read as a cheap month; overspending pushes outgo up.
- Card EMIs are billed under the card, not under Debts. `d.card` holds the
  card's name (`''` = loan), classified once from the debt name by
  `cardLabel()` (`"HSBC CC — Amazon Pay"` → `HSBC`); the Card column in the
  Debts table is the escape hatch when that guess is wrong. A card keeps its
  row after its EMIs finish — you can still spend on it. **Spend on top
  should be zero every month**; the block says so and colours it red when it
  isn't.
- The old "What actually happened" table is gone. Its `oneoff` column
  migrates into `extras` (`migrateLogOneoffs()`), `checkpoint` and `note`
  moved into the card's foot, `extraIncome` is now a one-time Cash In entry,
  and a legacy `actual` still overrides that month's variable budgets until
  the banner's button clears it — money already entered is never dropped.

Month records key their overrides by row id, so `normalize()` stamps a
stable `id` on every debt and commitment before anything else runs; index
positions shift the moment a row is deleted and would silently repoint a
month's figures at a different row. `normalize()` is called from `load()`
and from import, and owns all four migrations.

The seed's `Card spend on top` commitment is superseded by the per-card
entries — `deadCardTop()` drops it from both the UI and the arithmetic, but
only while it is ₹0, so a real figure can never vanish.

`renderSpend()` rebuilds the card and only runs for structural changes (a
different month, a row added or removed). Typing goes through `setSpend()`
and then `updateSpendNumbers()`, which rewrites the derived figures
(`[data-calc]`, `[data-bar]`, `[data-ovr]`) in place — same split, and same
reason, as [[arch-render-split]].

The runway strip (`#strip`, hero section) is hover-interactive: a custom
glass tooltip (`#striptip`, positioned in JS via `mousemove`, no native
`title`) shows that month's label, balance, income, outgo and net on
hover, and a `click` handler does the same on tap for touch devices where
hover doesn't fire. The current month (`i===0`) gets a permanent outline
and a small tick mark (`.cell.cur`) so it reads without hovering. Rows are
matched back to their full data via `data-i` on each `.cell` and a
module-scope `lastR` (the most recent `project()` output, set at the end of
`renderLive()`) — the tooltip never recomputes the projection itself.

Opening balance is `settings.reserve` minus the payoff of every debt marked
cleared. `payoff = outstanding × (1 + fee% × 1.18)` — the 1.18 is 18% GST on the
foreclosure charge, which Indian lenders levy. `netSaved = (emi × months −
outstanding) − (payoff − outstanding)`, i.e. interest avoided minus the charge.

Checkpoints exist because a 60-month forecast drifts. They pin it to the real
bank balance.

## Domain notes that matter

- **Foreclosing is not automatically good.** Spending reserve to kill a
  long-tenure low-rate loan *shortens* runway — cash leaves now, savings arrive
  over years. It only wins when the rate is high enough. Finnable at 21.99% is
  worth clearing; ICICI PL at 13.25% is marginal; Axis PL with 4 months left is
  pointless. Any feature that ranks debts must reflect this, not just sort by
  interest saved.
- **The structural problem is the monthly gap, not the debt.** Fixed
  commitments exceed salary before any EMI is paid. No foreclosure fixes that.
  Don't build UI that implies otherwise.
- **Several debts are held in another person's name** (marked `who: "Nissi"`).
  They are treated as the user's obligation in every calculation. The field
  exists only because foreclosure paperwork needs the account holder's
  signature.
- **Credit card EMIs carry GST on the interest portion**, so the true monthly
  cost is above the headline EMI. Stored `emi` values already include it.

## Unresolved — don't treat these as settled

- Cred Aditya Birla: rate unknown, tenure estimated at 18 months.
- SBI Education Loan: rate unknown, tenure estimated at 37 months.
- Finnable's sanction letter denies an interest rebate on early repayment,
  contradicting its own foreclosure-charge table. Needs a written quote.
- HSBC's total loan outstanding is ~₹9,700 above the two known plans; a new
  ₹33,308.77 charge posted 17 Aug may be starting a third.
- SBI FlexiPay says 11 of 36 remaining, but the balance implies about 21 months.
- Car loan and the ₹13L personal loan tenures are assumed, not confirmed.

## Style

Dark UI, `Archivo`, tabular numerals throughout. Signal colours carry meaning:
green safe, amber thin, red gone. Don't add colour that doesn't mean something.
Amounts render through `rs()` as `₹1,23,456` (Indian grouping). No em-dash-heavy
microcopy, no exclamation marks — this is someone checking whether they make it
to 2029.

## Good next things

- Reserve-over-time chart (SVG, no chart library).
- Compare two foreclosure scenarios side by side instead of toggling.
- Carry an overspent variable budget forward as next month's extra spend,
  instead of leaving it to be re-entered by hand.
