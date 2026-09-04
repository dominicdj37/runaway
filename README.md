# Runway

Debt and reserve tracker. One HTML file, no build step, GitHub Pages + Firebase.

Tick a debt to model foreclosing it and watch the month your money runs out move.

## Privacy

**No personal figures live in this repository.** `index.html` starts empty. Your
numbers sit in `seed.json`, which `.gitignore` excludes, and in Firestore behind
your auth rules.

Run `git status` before your first commit and confirm `seed.json` is not listed.

## Setup

**1. Firebase**

- console.firebase.google.com → Add project
- Authentication → Get started → enable **Google**
- Authentication → Settings → Authorised domains → add `<you>.github.io`
- Firestore Database → Create database → **production mode**

**2. Rules — do this before pushing**

Firestore → Rules → paste `firestore.rules` → Publish. The repo is public, so
these rules are the only thing protecting your data.

**3. Config**

Project settings → Your apps → Web (`</>`) → register → copy the config object
into the `firebaseConfig` block near the top of `index.html`.

Those values are safe in public. The API key identifies the project; it grants
nothing. The rules do the protecting.

**4. Deploy**

```bash
git init
git add .
git status                 # confirm seed.json is absent
git commit -m "Runway"
git branch -M main
git remote add origin git@github-personal:<you>/runway.git
git push -u origin main
```

Settings → Pages → Source `main`, folder `/ (root)`.

**5. Load your figures**

Open the site, sign in, Settings → **Import from file** → pick `seed.json`.
Firestore holds it from then on. Use **Export backup** regularly — it is your
only backup.

**Inviting someone** — Settings → Members → Invite someone, enter their
Google email. When they sign in, they'll see an accept/decline screen; once
accepted they have full view and edit access to the same data. Only the
original owner can invite or cancel a pending invite.

## Using it

**Debts** — tick *Clear* to model foreclosing. EMIs stop, the payoff comes off
the reserve. *Net saved* is interest avoided minus the foreclosure charge.

**Monthly commitments** — everything that isn't a loan. Mark a row *variable* if
monthly actuals should override it.

**Monthly spends** — pick a month and work down it: commitments, card bills,
loan EMIs, extras.

- Every amount starts at what you planned. Changing one here changes that month
  only, not the commitment behind it.
- Variable budgets (groceries, utilities) take individual entries and show what
  is left. A budget still costs its full amount until you overspend it, so a
  half-logged month never reads as a cheap one.
- Card EMIs are billed under their card, with a box for whatever went on the
  card on top. That figure should be zero every month.
- *Balance checkpoint* re-anchors the forecast to your real bank balance. Use it
  quarterly; without it small errors compound over 60 months.

## Still to confirm

- **Cred Aditya Birla** — rate unknown, tenure estimated at 18 months. The CRED
  app or their customer care will give the rate in two minutes.
- **SBI Education Loan** — rate unknown, tenure estimated at 37 months.
- **Finnable** — the sanction letter says no interest rebate on early repayment,
  contradicting its own foreclosure-charge table. Get a written quote before paying.
- **HSBC** — new ₹33,308.77 KGR Visionary charge from 17 Aug; total loan
  outstanding is ~₹9,700 above what the two known plans account for.
- **SBI FlexiPay** — says 11 of 36, but the balance implies about 21 months left.
- **Car loan and the ₹13L personal loan** — both tenures assumed, not confirmed.

## Notes

- Axis PL, IDFC PL and Axis CC are held in Nissi R's name. Foreclosure paperwork
  normally needs the account holder's signature.
- Nothing here is financial advice. It is arithmetic on figures you entered.
