# Exception Resolution Workbench

A human-in-command AI workbench for reviewing and resolving flagged AP (accounts payable) transactions — price/quantity/tax mismatches, duplicate invoices, missing POs, and ambiguous cases — with transparent, evidence-based confidence scoring that decides when the system can safely auto-resolve versus when it must escalate to a human.

---

## 1. How to present this to a recruiter

Don't say: *"I built an AI chatbot for transactions."*

Say:

> **"I built a human-in-the-loop AI exception resolution workbench that lets a reviewer investigate flagged transactions, get source-grounded explanations and resolution recommendations, and safely auto-resolve high-confidence cases while escalating uncertain ones to a human — with every decision logged to an audit trail."**

Then walk them through the workflow:

```
Transaction
    │
    ▼
Exception detected (deterministic rules, not AI)
    │
    ▼
AI Explanation  ──────────────► grounded only in transaction fields
    │
    ▼
AI Recommendation (structured JSON: action + reason)
    │
    ▼
Confidence Evaluation ─────────► evidence-based scoring (not random, not model-guessed)
    │
    ├── ≥ threshold ──► Auto-Resolve ──► logged
    │
    └── < threshold ──► Human Review ──► Approve / Reject ──► logged
```

This single sentence covers five things interviewers actually screen for:
1. **AI engineering** (structured, grounded LLM calls — not free-text chat)
2. **Backend/business logic** (deterministic exception detection, confidence math)
3. **Frontend engineering** (real dashboard, not a wireframe)
4. **Responsible AI / safety** (fail-safe escalation, no blind automation)
5. **Auditability** (every decision — AI or human — is traceable)

---

## 2. What's actually built

| Layer | What it does |
|---|---|
| **Detection** | 28 mock transactions generated with deterministic rule-based exception types (price mismatch, quantity mismatch, tax mismatch, duplicate invoice, missing PO, and one deliberately ambiguous case) |
| **Confidence engine** | Transparent point-based scoring: exact matching evidence (+0.40), clear single exception type (+0.25), required fields present (+0.20), no unresolved ambiguity (+0.15). Never `Math.random()`. The full breakdown is shown to the reviewer, not hidden. |
| **AI layer** | Two grounded calls per exception: **Explain** (why was this flagged, using only the given fields) and **Suggest Resolution** (structured JSON: `recommended_action` + `reason`). A third grounded call powers the chat panel. |
| **Threshold logic** | Configurable slider (default 90%). `confidence ≥ threshold → auto-resolve`. `confidence < threshold → status = REVIEW`, reviewer must Approve or Reject. |
| **Audit trail** | Every state transition — created, AI explained, AI recommended, auto-resolved, sent to review, approved, rejected — is timestamped per exception. |
| **Persistence** | State (exceptions + threshold) is saved so it survives a reload, with graceful in-memory fallback if the storage backend is briefly unavailable. |

### Why the confidence score is defensible

The doc this was scoped from specifically calls out a common shortcut: generating a fake confidence number and hoping nobody checks. This build avoids that by computing confidence from **evidence quality**, shown as four labeled bars in the UI:

```
Exact matching evidence        ████████░░  0.40 / 0.40
Clear exception type           ████████░░  0.25 / 0.25
Required fields present        ████████░░  0.20 / 0.20
No unresolved ambiguity        ████████░░  0.15 / 0.15
                                            ────────────
                                             1.00 total
```

The LLM is *told* this score and asked to justify a recommendation consistent with it — it never sets or overrides the number. That separation (deterministic safety logic vs. LLM reasoning) is the single most important design decision in the project, and the one worth explaining first in an interview.

---

## 3. Architecture note (what's simulated vs. real)

This is a **single-file client-side app**, not the full Node/Express/MongoDB stack described in the original scoping doc. That was a deliberate trade-off to get you a working, demoable product fast. What's real vs. simulated:

- ✅ **Real**: exception detection logic, confidence engine, threshold enforcement, audit trail, state management, UI/UX — all genuine, not mocked.
- ✅ **Real AI calls**: Explain / Suggest Resolution / Chat call Claude directly with structured prompts and JSON parsing — not canned responses.
- ⚠️ **Simulated**: there's no real database or backend API — "MongoDB" is `window.storage` (a key-value store), and there's no Express REST layer. If you want the literal architecture from the scoping doc (React + Express + MongoDB, with `GET /api/exceptions`, `POST /api/exceptions/:id/resolve`, etc.) for submission credibility, that's a separate build — ask and it can be scaffolded.

---

## 4. Running it after download

**Simplest option — just open it:**
Double-click `exception-resolution-workbench.html`, or drag it into a browser tab. The dashboard, queue, filters, confidence engine, and audit trail all work immediately with zero setup — there is no build step, no `npm install`, no server required for those parts.

**Recommended option — serve it locally** (avoids occasional `file://` quirks with some browsers' local storage handling):
```bash
cd folder-containing-the-file
python3 -m http.server 8000
# then open http://localhost:8000/exception-resolution-workbench.html
```
or, if you have Node:
```bash
npx serve .
```

### Important: the AI buttons (Explain / Suggest Resolution / Chat) will NOT work outside Claude.ai

Inside Claude.ai's artifact preview, API calls to Claude are transparently authenticated for you. Once the file is downloaded and opened locally or hosted elsewhere, those same calls have no credentials behind them and will fail (you'll see a "couldn't generate" alert). Everything else in the app — queue, filters, confidence scoring, audit trail, threshold logic — is unaffected and works fully offline.

To make the AI features work outside Claude.ai, see the hosting section below.

---

## 5. Hosting it

### Option A — Static hosting (fastest, but AI buttons stay disabled)
Good for showing the dashboard/UX/confidence-engine work in a portfolio link.

- **Netlify**: go to app.netlify.com → "Add new site" → "Deploy manually" → drag the `.html` file in. Live URL in seconds.
- **Vercel**: `npx vercel` in the folder, or drag-and-drop via vercel.com/new.
- **GitHub Pages**: push the file to a repo, enable Pages in repo settings, pointing at the branch/root. Rename the file `index.html` for a clean root URL.

### Option B — Add a tiny backend proxy so AI calls work when hosted (recommended for a real demo)
You need one small server endpoint that holds your real Anthropic API key and forwards requests, so the key never sits in browser-visible code. Minimal shape:

```js
// server.js (Node + Express, ~15 lines)
import express from "express";
const app = express();
app.use(express.json());

app.post("/api/claude", async (req, res) => {
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": process.env.ANTHROPIC_API_KEY,
      "anthropic-version": "2023-06-01"
    },
    body: JSON.stringify(req.body)
  });
  res.json(await r.json());
});

app.listen(3000);
```
Then in the HTML file, change the `fetch("https://api.anthropic.com/v1/messages", ...)` call to `fetch("/api/claude", ...)` (three occurrences: explain, recommend, chat). Deploy the server to Render, Railway, Fly.io, or a small VPS, set `ANTHROPIC_API_KEY` as an environment variable there, and host the HTML from the same server as a static file.

If you want, I can make this edit and hand you both files (server + updated frontend) ready to deploy — just say so.

---

## 6. Test scenarios worth demoing

| Scenario | What to show |
|---|---|
| High-confidence case | Open `EX-1004` (duplicate invoice) → Suggest Resolution → watch it auto-resolve with confidence breakdown visible |
| Low-confidence case | Open the `AMBIGUOUS` exception → Suggest Resolution → watch it route to Review instead of guessing |
| Human override | On a REVIEW item, click Reject → confirm status and audit entry update |
| Threshold sensitivity | Drag the threshold slider down to 60% → re-run Suggest Resolution on a borderline case → show it now auto-resolves — demonstrates the logic is genuinely threshold-driven, not hardcoded |
| Grounded chat | Ask "why was this flagged?" and "can this be auto-resolved?" in the chat panel — answers should cite the actual numbers, not generic text |
