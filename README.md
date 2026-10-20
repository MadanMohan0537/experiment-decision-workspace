# Experiment Decision Workspace

**A browser workspace for keeping a running record of product experiments: declare the hypothesis and minimum useful effect, enter observed counts and guardrails, get a calculated Hold / Ship / Iterate / Rollback recommendation, and log the human decision. Saved decisions become a searchable record of what was learned.**

The GitHub repository was renamed from *ai-product-experimentation-platform*. The rename drops "platform" because the app does not ingest live events or control feature flags. `package.json` `name`, the HTML document title, and the Worker name produced by the build are still `ai-product-experimentation-platform`.

## Problem and who it's for

Teams run many experiments, but the reasoning behind each decision (the hypothesis, the guardrails, why it shipped or didn't) lives in scattered docs and spreadsheets and is forgotten by the next planning cycle. Product managers and growth leads need one place to plan an experiment, read it out consistently, and keep the decision and its evidence together.

## Approach

1. **Plan:** hypothesis, audience, primary conversion metric, baseline rate and relative minimum detectable effect (MDE). The app shows the approximate per-arm sample size (50/50 split, two-sided α = 5%, 80% power) from `sampleSize` in `lib/experiment-engine.ts`.
2. **Record results:** enter control and treatment visitors and conversions, plus guardrails in their own units with a preferred direction and an allowed regression. A guardrail breach is a raw difference against that allowance, not a second statistical test.
3. **Read out:** rates, absolute and relative lift, a normal-approximation 95% interval for the difference (clamped to [-1, 1], omitted when the sample is sparse), a two-sided p-value, and a sample-ratio check. The calculated recommendation is Hold, Ship, Iterate, or Rollback. **Ship** requires a non-sparse sample, both arms at the planned size, no sample-ratio mismatch (1% chi-square threshold), a two-sided p-value under 5%, an absolute lift at least as large as the predeclared relative MDE, and at least one guardrail that is within its allowance. A positive primary result with no guardrails stays **Hold**. A breached guardrail, or a significant negative primary difference, is **Rollback**.
4. **Decide:** record your own decision (Ship, Hold, Iterate, or Rollback) and a planned rollout percentage from 0 to 100. Neither one releases anything. The percentage is saved planning state, not a Cloudflare Workers traffic split.
5. **Learn:** search saved names, hypotheses, metrics, audiences, and decisions, and export or import the workspace as validated JSON (`version: 1`, at most 200 experiments, import files up to 1 MB).
6. **Optional planning help:** "Suggest a measurement plan" posts the experiment name, hypothesis, and audience to `/api/ai/design`. With `DEEPSEEK_API_KEY`, that route calls DeepSeek. If only `ANTHROPIC_API_KEY` is set, it calls Anthropic. DeepSeek is used when both keys are set. Without a key, the server builds a deterministic outline and labels the provider `local-outline`, so a template is never presented as AI output.

## What exists today

- A Next.js 16.2.6 / React 19.2.6 app, built with Vinext 0.0.50 and Vite 8.0.13. The UI is `app/page.tsx`. Local dev and the production build use the Cloudflare Vite plugin. The Worker source entry is `worker/index.ts`; `npm run build` emits `dist/server/index.js`.
- The statistics engine (`lib/experiment-engine.ts`) throws on impossible plans, invalid counts, incomplete guardrails, and malformed imports. Sparse samples (fewer than 5 successes or failures in either arm) do not throw: the readout returns **Hold** with a null p-value and a null interval.
- **11 engine tests** in `tests/experiment-engine.test.mjs` (`npm run test:engine`). They cover impossible sample-size targets, a positive result that ships with a guardrail, a positive result that holds without guardrails, guardrail override to Rollback, low sample and sparse outcomes, sample-ratio mismatch, a negative result, a zero difference (Iterate), invalid counts and guardrails, empty counts ("Awaiting results"), and import validation. On Node.js v22.23.3, all 11 passed. The file imports `lib/experiment-engine.ts` directly. Node.js enables that type stripping by default in 22.18.0 and later. Node.js 20 cannot load the import. `package.json` declares `engines.node` of `>=22.13.0`.
- `npm test` runs those engine tests, `npm run build` (`scripts/build-verified.sh`), and `tests/rendered-html.test.mjs` (2 tests: the built worker returns HTML with the development preview meta tag, and `/api/ai/design` returns `local-outline` when no provider key is set). That command passed in this environment. `npm run lint` runs ESLint and reported no findings.
- Workspace rows stay in this browser's `localStorage` under `variant-experiments-v1` (at most 200 experiments). Observed counts are not sent to a model. The optional planning request sends the experiment name, hypothesis, and audience to this app's `/api/ai/design` route, which calls DeepSeek or Anthropic only when the matching key is configured.
- No Wrangler config is committed. No GitHub Actions workflow is present, and the GitHub deployments API for this repository returned no deployment records. This README does not claim a live URL.

## How success would be measured

| Measure | Status |
|---|---|
| Experiments saved with predeclared baseline, MDE and guardrails | Not yet measured |
| Readouts where the human decision matches the calculated recommendation (and why not, when it differs) | Not yet measured |
| Reuse: past decisions found via search during new planning | Not yet measured |

## Limitations

- Counts are typed in by hand. There is no event ingestion and no feature-flag integration. The rollout slider is a plan only.
- The primary metric is a binary conversion rate with two arms and one fixed-horizon look. Repeated peeking is not corrected for. Guardrails are entered numbers, not hypothesis tests.
- The workspace lives in one browser's `localStorage`. Export before clearing browser data or switching devices.
- Model-generated plans are proposals to review, not validated designs. The page shows the hypothesis, primary metric, and guardrail names from that proposal.
- The document title is still "AI Product Experimentation Platform", and its meta description still says the app monitors experiments and makes rollout decisions. The page header in `app/page.tsx` says "Experiment decision workspace".
- Starter-template files remain in the tree. The running app does not import `app/chatgpt-auth.ts`, `db/`, or `examples/d1/`. `db/schema.ts` exports nothing, and `drizzle/meta/_journal.json` has no migrations. The Sites build plugin still copies `drizzle/` into `dist/.openai` when that directory exists.
- There is no LICENSE file yet.

## Quick start

`package.json` requires Node.js 22.13.0 or later. Run the engine tests on Node.js 22.18.0 or later so type stripping is on by default. API keys are optional.

```bash
git clone https://github.com/MadanMohan0537/experiment-decision-workspace.git
cd experiment-decision-workspace
npm install
npm run test:engine
cp .env.example .env.local     # optional: DEEPSEEK_API_KEY or ANTHROPIC_API_KEY
npm run dev
```

`npm run dev` starts Vite. With this config it serves http://localhost:5173/.

`npm test` runs the engine tests, the verified Vinext build, and `tests/rendered-html.test.mjs`. `npm run lint` runs ESLint.

## Deployment

Cloudflare Workers is the deploy target, using `npx wrangler deploy`. This is not a versioned release: do not use `wrangler versions upload`, a percentage or gradual Worker rollout, or a git tag as the deploy step. The in-app rollout percentage is not that mechanism.

No Wrangler config is committed, and this repository does not record a live workers.dev URL. `npm run build` writes `dist/server/wrangler.json`. Deploy with that generated file so the Worker name stays `ai-product-experimentation-platform`:

```bash
npm run build
npx wrangler deploy --config dist/server/wrangler.json
```

That generated config, from the build run here, sets:

- **Worker name:** `ai-product-experimentation-platform` (same as `package.json` `name`). Keep this name so an existing workers.dev hostname for this Worker would not change.
- **Entry:** `main` is `index.js`, the built file `dist/server/index.js`. Dev uses the source entry `worker/index.ts` through the Cloudflare Vite plugin. This app is TypeScript, so it does not need Python Workers or a language port.
- **Bindings:** no D1, R2, KV, service, or secret bindings. `.openai/hosting.json` sets `d1` and `r2` to null. Static assets are `../client` relative to the config file. The compatibility flag is `nodejs_compat`.
- **Secrets:** none are required to boot. `DEEPSEEK_API_KEY` and `ANTHROPIC_API_KEY` are optional and are not declared in the generated config. Counts and decisions are not deployed with the Worker; they stay in the browser.

`npx wrangler deploy` from the repository root, without `--config`, has no Wrangler file to read.

## Which experimentation repo should I read?

| Repo | Use it for |
|---|---|
| **experiment-decision-workspace** (this one) | Keeping experiments, readouts and human decisions together over time, in the browser |
| [ab-test-design-and-readout](https://github.com/MadanMohan0537/ab-test-design-and-readout) | Design preflight and a validity-first readout (Python API) |
| [bayesian-ab-test-advisor](https://github.com/MadanMohan0537/bayesian-ab-test-advisor) | Bayesian analysis with expected loss (Python) |

## License

No license file is included yet.
