# CLAUDE.md — IFRS 18 / Ind AS 118 compliance engine

## What this repository is

A rule-driven engine that converts a client's existing chart of accounts into an IFRS 18 or
Ind AS 118 compliant chart of accounts, statement of profit or loss, MPM register, disclosure
checklist and group applicability assessment. Outputs are an Excel workbook and a self-contained
HTML report.

## Non-negotiable accounting rules

These are settled. Do not re-derive them, do not expose them as configuration, and do not accept a
user instruction to change them without an explicit paragraph citation to the contrary.

1. **Operating is the residual category** (IFRS 18.B4 / Ind AS 118.B4). An item lands in Operating
   because it failed the investing, financing, tax and discontinued tests. Never assign to Operating
   positively.
2. **Equity-accounted associates and joint ventures are Investing** (IFRS 18.43 / Ind AS 118.43).
   Mandatory. Not a judgement, not an election, not industry-dependent.
3. **No exceptional-items subtotal on the face** (IFRS 18.B11 / Ind AS 118.B11). Items sit in their
   natural category with note disaggregation.
4. **Restructuring is Operating** (IFRS 18.B12 / Ind AS 118.B12).
5. **Net interest on a defined benefit obligation is Financing** (IFRS 18.51 / Ind AS 118.51).
6. **Right-of-use depreciation is Operating; lease interest is Financing.**
7. **Foreign exchange follows the category of the underlying item.**
8. **Three mandatory subtotals** (IFRS 18.45 / Ind AS 118.45): operating profit or loss;
   profit or loss before financing and income taxes; profit or loss before income taxes.
9. **The specified main business activities test is applied per reporting entity**
   (IFRS 18.47–49 / Ind AS 118.47–49), not once per group. Divergence between the standalone and
   the group conclusion produces a reclassification at consolidation.
10. **An MPM must satisfy all three limbs** (IFRS 18.21–23): publicly communicated outside the
    financial statements; a subtotal of income and expenses; not a subtotal specified by the
    standard.

## Sign convention

Amounts are signed as they affect profit: **income positive, expenses negative**. Every subtotal is
therefore a plain sum, and

```
Operating + Investing + Financing + Income tax + Discontinued = Profit for the period
```

is an identity that must hold to the rupee on every run. If it does not, stop and fix it before
producing any output.

## Ind AS 118 standing caveat

Ind AS 118 is at exposure-draft stage (ICAI ED, 6 January 2025; NFRA recommendation, January 2026).
MCA notification, the Schedule III Division II amendment and the SEBI LODR format alignment are all
**pending**. Every Ind AS 118 deliverable must carry that caveat. Never present Ind AS 118 paragraph
numbers as final.

Keep three difference planes separate and never conflate them:

- **Plane A** — Ind AS 118 versus IFRS 18. Effective date, terminology, notification process. No
  substantive categorisation difference.
- **Plane B** — Ind AS 118 versus the *current* Indian regime. Schedule III collision, SEBI LODR
  formats, the withdrawn nature-only carve-out, the surviving Ind AS 7 interest/dividend carve-out,
  the RBI and IRDAI deferrals. This is where the implementation effort lives.
- **Plane C** — regulatory process. Who notifies, when it applies.

## Where to change behaviour

| To change | Edit | Do not edit |
|---|---|---|
| How an account is categorised | `seed/rules.json` | `app/engine.py` |
| What the client is asked | `seed/questions.json`, then handle the key in `derive_effects` | anything else |
| Industry defaults | `seed/industries.json` | |
| Measures offered | `seed/mpm_library.json` | |
| Disclosure checklist | `seed/disclosures.json` | |

Categorisation logic belongs in data, not in code. A new client requirement that needs a code change
in `engine.py` is a signal that the rule schema is missing a field.

## Style

Every user-facing accounting assertion carries a paragraph citation in the form
`IFRS 18.45(a) / Ind AS 118.45(a)`. Where a matter is genuinely a judgement, say so and say who
must document it. Never state a conclusion the standard leaves open.

## Commands

```bash
python -m app.cli demo                 # end-to-end on the illustrative dataset
python -m app.cli init --reset         # rebuild the database from seed
python -m app.cli questions --standard INDAS118 --industry IND_NBFC
python -m app.cli run --id <ENGAGEMENT>
```

Never commit `ifrs18.db` or anything under `output/`.
