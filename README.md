# Signal Desk: AI Campaign Planner (concept)

**Live demo:** https://samiksha721.github.io/Signal-Desk-Campaign-Builder/

A product prototype exploring how data signals can act as *verifiers* in an AI-assisted campaign planner for martech.

A marketer picks a campaign objective and switches on data layers (**location**, **transaction**, **behavioral**). The planner recomputes the audience funnel, channel mix, send timing, expected revenue, cost to build, cost to run and a confidence score. An LLM then writes brand-checked subject lines and a plain-language report for the teams who act on the plan.

## Where AI is used, and where it deliberately isn't

| Part of the page | Produced by | Why |
|---|---|---|
| Subject lines | **LLM** (Claude Haiku 4.5), checked by brand rules | Language generation |
| Planner report and next steps | **LLM**, with a numeric grounding check | Explaining a plan in plain language |
| Funnel, revenue, costs, ROAS | Plain code | Must be exactly right and repeatable |
| Channels, timing, segment | Rules | Predictable and auditable |
| Calculation trace | Templates from the calculated numbers | Shows exactly how each figure was built |

Rule of thumb: the LLM handles language and explanation, code does the math, and rules enforce what must never be wrong.

## How the AI layer works

```
Browser (GitHub Pages)  ──POST──▶  Cloudflare Worker (worker.js)  ──▶  Claude API
   no API key                       holds the key, validates input,
                                    checks the model's output
```

- **`POST /subjects`**: the Worker looks up the fictional brand's voice, banned phrases and the campaign's real offer, asks the model for three lines, then **blocks** any line that invents a discount, uses a banned phrase, mentions a discount the campaign doesn't have, uses all-caps or emoji, or exceeds 70 characters. Lines over 50 characters are flagged as likely to be cut off on mobile.
- **`POST /report`**: the Worker rebuilds a fact sheet from validated numbers and allow-listed values, and asks the model for a short report for the **campaign management** and **activation** teams. It then runs a **grounding check**: every figure in the report has to appear in the fact sheet, and anything that doesn't is shown as a warning on the page.

### Guardrails
- API key lives only in the Worker as a secret. The page never sees it.
- Only origins listed in `ALLOWED_ORIGINS` can call the Worker.
- Inputs are validated against allow-lists, so no free text from the browser reaches the prompt.
- Model output is treated as untrusted: escaped before display, and checked before it's shown.
- Token caps per request, a best-effort rate limit, and a monthly spend limit in the Claude Console as the real cost backstop.
- AI lines and reports are tied to the exact plan they describe. If the plan changes, they're marked out of date.
- Nothing is sent to customers. Every plan still needs human approval.

## What's simulated

No real customer or company data is used. **Harbor & Pine is a fictional brand.** Funnel rates, lift multipliers and costs are **illustrative assumptions** chosen to show the decision architecture, not to predict real results.

| Signal | Effect in the model |
|---|---|
| Transaction data | Refines the pool to proven buyers (×0.85 reach), conversion ×1.55, AOV ×1.28 |
| Behavioral data | Open rate ×1.42, click rate ×1.50, unlocks send-time optimization |
| Location data | Conversion ×1.12, adds SMS as a proximity channel |

Lift multipliers are relative changes that stack multiplicatively. In production they'd be measured against a holdout group, not assumed.

## Turn on the AI features

The page works without this. It shows rule-based subject lines and explains that AI isn't connected.

1. **Get an API key.** In the Claude Console, create an API key and **set a monthly spend limit**.
2. **Create the Worker.** In Cloudflare (free plan), go to Workers & Pages, create a Worker from the "Hello World" template, name it `signal-desk-api`, and deploy it.
3. **Paste the code.** Open the Worker's code editor, replace everything with `worker.js`, and deploy.
4. **Add settings.** In the Worker's Settings → Variables and Secrets:
   - `ANTHROPIC_API_KEY`: type **Secret**, value is your key
   - `ALLOWED_ORIGINS`: `https://samiksha721.github.io` (no path, no trailing slash)
5. **Check it.** Open the Worker's URL in a browser. You should see `"keyConfigured": true`.
6. **Connect the page.** In `index.html`, set `const API_BASE = "https://signal-desk-api.<your-subdomain>.workers.dev";` and commit.

Never put the API key in `index.html` or anywhere in this repo.

## Known gaps a production version would need

- **Wrong objective:** the page shows ROAS, which rewards reaching people who would have bought anyway. The intended north star is **incremental margin per campaign**, measured against a holdout group.
- **False precision:** point estimates should become confidence intervals.
- **Privacy and consent:** fusing location, transaction and behavioral data is highly re-identifying. It needs consent, purpose limitation and regional rules (GDPR, CCPA, TCPA, CAN-SPAM, India's DPDP Act).
- **Disparate impact:** targeting logic needs fairness audits.
- **Send fatigue:** frequency caps should apply across all campaigns and channels.
- **Grounding check scope:** it verifies figures above 10. Smaller percentages are not yet checked.

## Roadmap

- [x] LLM subject lines through a serverless proxy, with brand and offer checks
- [x] LLM planner report with next steps for campaign management and activation, plus a numeric grounding check
- [ ] Holdout-percentage control and a gross-margin input, with incremental margin as the headline metric
- [ ] Agent mode: mock data tools the agent calls itself, shown step by step in the trace
- [ ] Before/after comparison mode

## Files

- `index.html`: the whole planner (no build step)
- `worker.js`: the Cloudflare Worker that calls the model
