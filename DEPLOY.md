# Build, deploy and embed

The application in this folder is finished and tested. It is a static site with **no build step
and one vendored dependency** (SheetJS, for reading and writing xlsx). Everything below is
operational rather than architectural.

---

## What was built, and the one decision that shaped it

Browser-only, per your choice. Consequences worth being explicit about:

- The client's chart of accounts is read with `FileReader` and processed in memory. **No request
  is ever made carrying client data.** There is nothing to breach, retain or contract for.
- Session state persists to IndexedDB so a refresh does not lose an hour of answers. That state is
  per-browser and per-device — there is no cross-device resume, and clearing site data clears it.
- The rule library ships as JSON and is readable by anyone who opens devtools. If the 61 rules and
  72 questions are IP you intend to monetise, this is the price of the architecture. My view is
  that the rules are not the moat: the interpretation you put around them is.
- No build step means no dependency drift, no `npm audit` treadmill, and a deploy that cannot fail
  for a reason unrelated to your code.

---

## Phase 1 — Repository

```bash
mkdir ifrs18-diagnostic && cd ifrs18-diagnostic
# copy the contents of this folder in
git init && git add -A && git commit -m "IFRS 18 / Ind AS 118 diagnostic"
gh repo create ifrs18-diagnostic --private --source=. --push
```

Add `CLAUDE.md` from the earlier deliverable at the root. It is what stops a later session
re-litigating the settled accounting rules.

## Phase 2 — Local development in Claude Code

```bash
cd ifrs18-diagnostic
python3 -m http.server 8080     # ES modules need a server; file:// will not work
claude
```

The modules are separated so a change has one obvious home:

| File | Owns |
|---|---|
| `js/engine.js` | Categorisation, subtotals, reconciliation, MPMs, the cross-foot assertion |
| `js/parse.js` | Upload, column mapping, sign-convention detection, pre-flight checks |
| `js/charts.js` | Waterfall, Sankey, sunburst, donut, bar — all hand-rolled SVG |
| `js/outputs.js` | The nine output sections and the Excel workbook |
| `js/app.js` | The six-step wizard and IndexedDB persistence |
| `data/*.json` | Industries, questions, rules, MPM library, disclosures, references |

**The rule that matters:** categorisation logic lives in `data/rules.json`, never in
`engine.js`. A new client requirement that forces a change to `engine.js` means the rule schema is
missing a field. Fix the schema.

## Phase 3 — Netlify

**Via the dashboard.** Add new site → Import an existing project → pick the repo. Build command:
leave empty. Publish directory: `.`. Deploy.

**Or from the terminal:**

```bash
npm i -g netlify-cli
netlify login
netlify deploy --prod --dir .
```

`netlify.toml` and `_headers` are already written. Two things they do that matter:

- **`frame-ancestors`** in the CSP is what permits the embed. Netlify does not set
  `X-Frame-Options` by default, so nothing needs removing — but if you later add a security
  template that sets it to `DENY`, the iframe goes blank with no useful error. That header and
  `frame-ancestors` must agree.
- `/js/*` and `/data/*` are set to `must-revalidate` so a rule correction reaches users on their
  next visit rather than whenever their cache happens to expire. `/vendor/*` is immutable.

**Before going live, edit `_headers`** and replace the placeholder domains with your live domain.
Which brings me to the thing I still need from you.

**The domains are now set** in `_headers`. Confirmed live on 29 August 2026: the canonical is
`https://www.intelligentadaptivefinance.com/` and the apex redirects to it, so `frame-ancestors`
names both, plus `intelligenceandadaptivefinance.zohosites.in` because several of your blog posts
are served from that origin and an embed placed on one of those pages would otherwise be blocked.

Nothing further is needed on the domain. The only placeholder left is your Netlify subdomain, which
appears in `embed-snippet.html` in two places.

## Phase 4 — Embed

The site runs on **Zoho Sites**, which constrains how the embed goes in. `embed-snippet.html` now
carries three versions; work down the list until one holds.

**A — auto-height (preferred).** Iframe plus a small `postMessage` listener. The tool posts its
height on every view change and the parent resizes the frame, so the page grows and shrinks with
the content. Paste into a Zoho HTML / Custom Code widget.

**B — no-script fallback.** Some Zoho Sites widgets strip `<script>` tags. If A renders but never
resizes, the listener was stripped: switch to B, which is a fixed 1600px frame that scrolls
internally. The tool works identically; you lose only the page-grows-with-content behaviour.

**C — link-out.** If the widget blocks iframes altogether, a styled button opening the Netlify
site in a new tab. Everything works; you lose the in-page experience.

Replace `YOUR-SITE.netlify.app` in **both** the `src` and the `ORIGIN` constant. They must match
exactly or the height listener silently does nothing.

**Test A on a staging page before publishing.** Zoho's HTML widget behaviour differs between page
types, and finding out on the live page is avoidable.

Notes on the sandbox attributes, since each one is load-bearing:

- `allow-scripts` — obvious.
- `allow-same-origin` — IndexedDB persistence needs it. Drop it and every refresh loses the session.
- `allow-downloads` — **the one people forget.** Without it the Excel and HTML exports fail
  silently. The button clicks, nothing happens, no console error.
- `allow-modals` — the resume-session and start-again confirmations.

A further Zoho consideration: your site loads its own scripts on every page. Because the tool runs
inside an iframe on a different origin, those scripts cannot reach the uploaded chart of accounts —
the same-origin policy is what makes the privacy claim true rather than merely asserted. Keep the
tool on the Netlify origin; do not be tempted to inline it into a Zoho page later.

## Phase 5 — Before it goes live

Work through this in order. Each has caught something in testing.

1. **Cross-foot.** Upload a chart of accounts whose net you already know. The banner must read
   "Cross-foot verified" and the figure must equal profit for the period in the published accounts.
   The engine reproduces the input exactly; it cannot make a wrong input right.
2. **Sign flip.** Upload the same file with every sign inverted. Detection must report "trial
   balance, high confidence" and produce the identical result. This is the single most likely
   silent failure in production.
3. **Both exports.** Click Download Excel and Download HTML in the deployed iframe, not just
   locally. This is where a missing `allow-downloads` surfaces.
4. **Offline.** Disconnect the network and reload. The page must render — no CDN, no external font.
5. **Mobile.** The tables scroll horizontally; check the wizard is usable at 380px.
6. **A real annual report.** Run a published set of accounts where you already know the IFRS 18
   answer, and confirm the engine agrees. Disagreements are missing rules, not bugs.
7. **`vega-audit`** over the exported HTML before you publish any client-facing version.

---

## What I would still change before this is client-facing

**The MPM reconciliations are illustrative and labelled as such.** They add back whatever the rule
engine tagged, not the entity's own published definition. That is honest but not useful for a real
engagement. The fix is a screen where the client enters their actual adjusting items — real work,
maybe a day.

**No prior-period comparative.** The mandatory transition reconciliation under IFRS 18.C1–C9 needs
the comparative period re-tagged. The tool handles one period. Adding a second upload slot is
straightforward and would make the output genuinely transition-ready rather than diagnostic.

**Balance sheet accounts are passed through, not categorised.** IFRS 18's categorisation is a P&L
matter, so this is correct, but a client who uploads a full trial balance will see them sit inert
in the revised chart. Consider filtering them out at the upload step with a note, rather than
carrying them.

**Analytics.** There is deliberately none. If you want to know how many people reach the results
step, add a privacy-preserving counter that fires on step transitions only and never touches the
financial data. Do not reach for a general-purpose analytics script: on a page where a CFO is
uploading an unpublished P&L, a third-party script in the same document is exactly the thing your
privacy claim promises is not there.
