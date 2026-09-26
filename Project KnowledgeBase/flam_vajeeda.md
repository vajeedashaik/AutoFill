# flam_vajeeda (Adaptive Layout Engine) — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\flam_vajeeda`
**GitHub origin:** `https://github.com/vajeedashaik/flam_vajeeda.git`
**Type:** TypeScript + React + Vite demo app around a pure-TypeScript constraint-based layout engine. Interview/assessment submission for **Flam** (an ad-experience platform). npm package name `adaptive-layout-engine`, v0.1.0, private. Not deployed ("Not yet deployed" in README; local preview at http://localhost:4173).
**Sibling projects:** the `flam demo` folder holds four other independent attempts at the same brief (`adaptive-layout-engine`, `adaptive-layout-engine-1`, `Adaptive-layout-engine-2`, `FlamaAi`). This repo is described as "The Main Submission" and was compared against them in `ANALYSIS.md` and `Adaptive-Resizing-Logic.md`.
**State when read:** uncommitted changes to `PHASE8-AUDIT.md`, `README.md`, `candidates.ts`, `candidates.test.ts`, `sample-data.ts`, `theme.css`. Latest commit: "add screenshot to documentation for improved visual reference".

## 1. What this project is

One declarative ad spec (content + intent, never hand-authored width/height/position) is resolved into a genuinely different, correct layout for each target **surface**: phone portrait, phone landscape, broadcast lower-third, retail kiosk, printed QR panel, or an unseen surface typed in live. Not CSS breakpoints or a lookup table. Every resolution runs a real pipeline:

1. **Experience Graph**: derives proximity/exclusion edges between elements from their *roles* (never ids).
2. **Context Engine**: turns surface constraints (tap size, viewing distance, attention window, aspect ratio, audio, motion) into qualitative flags.
3. **Candidate generation**: 5 spatial strategies through one shared placement engine.
4. **Deterministic scoring**: 8 weighted sub-scores plus a hard-fail rule that zeroes any candidate with overlap, clipping, a dropped must-keep element, or priority inversion.
5. **Explainable resolution**: a `DecisionTrace` records why each element was shrunk, grown or dropped and why each losing strategy lost.

Headline claims: 144 Vitest tests, `tsc --noEmit` strict with no `any`, 200 randomized adversarial surfaces per Stress Lab run with **0 failed every run**, self-healing on simultaneous bad input (broken images, absurd text, undersized surface).

The design bet (from `artifact.md`, the original project plan): model an ad as a *semantic graph of relationships* resolved by a deterministic, scored, explainable process. No per-surface hardcoding and no opaque ML. It targets the narrower problem of adapting one rich experience to wildly different contexts without redesigning each version by hand.

**Grading rubric it was built against (referenced throughout README/ARCHITECTURE):** Constraint resolution algorithm 35% · Layout correctness across surfaces 25% · TypeScript & architecture 20% · Example application 10% · Code quality 10%.

## 2. Tech stack

| Layer | Tech |
|---|---|
| Language | TypeScript 5.6, strict + `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noUnusedLocals/Parameters`, `noFallthroughCasesInSwitch` |
| UI | React 18.3 (only two runtime deps: `react`, `react-dom`) |
| Build | Vite 5.4 + `@vitejs/plugin-react` |
| Tests | Vitest 2.1 (Node environment) |
| Styling | Hand-written `src/styles/theme.css` (glassmorphism + claymorphism, Nykaa-branded: hot-pink accent, white cards, light only, `prefers-reduced-motion` aware) |
| Requires | Node ≥ 18 |

Scripts: `npm run dev` (Vite, :5173) · `npm run test` (144 tests) · `npm run build` (`tsc --noEmit && vite build`) · `npm run preview` (:4173) · `npm run typecheck` · `npm run test:watch`.

The project was verified with the chrome-devtools and Playwright MCP servers (live measurement of real `getBoundingClientRect()`, screenshots, Stress Lab runs). `.playwright-mcp/` and `audit-single.png` are leftovers from that.

## 3. Repository layout

```
flam_vajeeda/
├── src/
│   ├── core/                     pure engine, no React/DOM dependency
│   │   ├── spec.ts               AdSpec types + defineAd() validated factory
│   │   ├── surfaces.ts           SurfaceProfile types + defineSurface() validated factory
│   │   ├── sample-data.ts        1 sample ad (DIOR product) + 5 sample surfaces
│   │   ├── graph.ts              Experience Graph: buildGraph(), getRelatedNodes()
│   │   ├── context.ts            Context Engine: resolveContext()
│   │   ├── candidates.ts         5 strategies + shared placeElementsInOrder() engine
│   │   ├── scoring.ts            scoreCandidate(): 8 sub-scores + hard-fail rule
│   │   ├── resolver.ts           resolveLayout(): generate → score → pick → trace
│   │   ├── trace.ts              DecisionTrace, buildTrace(), explainLoss()
│   │   ├── text-measure.ts       Canvas measureText() with Node linear fallback
│   │   ├── naive-resolver.ts     deliberately dumb baseline for Naive-vs-Smart
│   │   ├── stress-lab.ts         generateRandomSurfaces, runStressTest, categorizeStressDetail
│   │   ├── self-healing-scenario.ts  adversarial spec + tiny surface
│   │   ├── render-dom.ts         deliberate stub (renderer-agnostic seam)
│   │   ├── spec.invalid-example.ts.txt  documented compile-error demo (excluded from build)
│   │   └── __tests__/            11 suites: candidates, context, graph, resolver, scoring,
│   │                             self-healing, spec, stress-lab, surfaces, text-measure, trace
│   ├── components/
│   │   ├── SurfaceStage.tsx      THE one ResolvedLayout → pixels renderer (React DOM)
│   │   ├── DeviceFrame.tsx       phone / TV / clean chassis wrapper (presentation only)
│   │   ├── DebugOverlay.tsx      safe-area outline + per-element boxes
│   │   ├── LayoutDebugger.tsx / LayoutCounterfactuals.tsx / LayoutHealthCheck.tsx   3 explainability panels
│   │   ├── StressLab.tsx / MultiSurfaceView.tsx / SelfHealingDemo.tsx
│   │   ├── NaiveVsSmart.tsx / DegradationSlider.tsx / UnknownSurfaceInput.tsx
│   │   └── useCountUp.ts         presentation-only number animation hook
│   ├── styles/theme.css
│   ├── App.tsx                   top nav + view switching, surface picker
│   └── main.tsx
├── public/products/dior-backstage-rosy-glow.png   real 490×648 product photo
├── docs/screenshot.png
├── dist/                          production build output
├── README.md  ARCHITECTURE.md  PHASE8-AUDIT.md  ANALYSIS.md  Adaptive-Resizing-Logic.md
├── UI-Enhancement.md  artifact.md  PHASE2-MANUAL-STEPS.md
├── phase1.md … phase7.md          the prompts each build phase was generated from
└── package.json  tsconfig.json  vite.config.ts  vitest.config.ts  index.html
```

Rule enforced by tests: `resolver.ts`, `candidates.ts` and `scoring.ts` contain **zero surface ids and zero ad-element ids** (`resolver.test.ts` does a source-level regex check). The UI is presentation-only; nothing in `src/core` imports styling, and presentation must never influence resolver geometry.

## 4. Data model

### AdSpec (`spec.ts`, built via `defineAd`)
Closed unions (a typo like `role: "primry"` is a compile error, TS2820):
- `ElementRole`: `primary | hero | action | branding | secondary`
- `ElementType`: `text | image | button`
- `importance`: `critical | should-survive | nice-to-have`
- `interaction`: `clickable | static`
- `visibility`: `always | degradable | decorative-only`

Element fields: `id`, `type`, `role`, `priority` (positive; **lower = more critical**), `importance?`, `interaction?`, `visibility?`, `minSize?`, `preferredSize?`, `brandRules?: { locked?, minSize? }`, `text?`, `fontSize?`, `src?`.
`defineAd` throws named `AdSpecValidationError` (valid role, no duplicate ids, positive finite priority, ≥1 element).

### SurfaceProfile (`surfaces.ts`, built via `defineSurface`)
`width`, `height`, `safeArea?`, `minTapTarget?`, `minTextSize?`, `touchOnly?`, `viewingDistance?` (`"near" | "far" | number cm`), `interaction?` (`none | touch | remote`), `attentionWindow?` (seconds), `motion?`, `audio?`.
`defineSurface` throws `SurfaceValidationError`; cross-field rule: `touchOnly` requires a positive `minTapTarget`. The Unknown-surface form shows this error verbatim (a non-numeric width names `NaN`, not `null`).

### Sample ad (current `sample-data.ts`): DIOR Backstage Rosy Glow Stick, 012 Rosewood, styled as a Nykaa PDP ad

| id | type / role | priority | visibility | notes |
|---|---|---|---|---|
| `headline` | text / primary | 1 | always | "Buildable Rosy Glow For Cheeks & Lips", 24px, min 180×40, pref 420×96 |
| `cta` | button / action | 2 | always | "ADD TO BAG", 15px, clickable, min 120×44, pref 200×56 |
| `product-image` | image / hero | 3 | always | real photo, min 73×96, pref 363×480 (matches the photo's ~0.756 aspect) |
| `price` | text / secondary | 4 | degradable | "₹2,900 · Shade 012 Rosewood", min 64×24, pref 140×48 |
| `logo` | text / branding | 5 | decorative-only | "DIOR", brand-locked, min 24×24, pref 96×32 |

History note: the product image was promoted to `visibility: "always"` and priority 5→3 in "§7.8" after feedback that the photo is *the highlight* and must survive while price/logo get sacrificed.

### Five sample surfaces

| id | size | key traits |
|---|---|---|
| `mobilePortrait` | 390×844 | touch, tapTarget 44, near, attention 3s, safeArea top59/bottom34 |
| `mobileLandscape` | 844×390 | touch, tapTarget 44, attention 2s, safeArea left/right 59 |
| `broadcastLowerThird` | 1920×320 | far viewing, audio, minText 32, attention 6s, no touch |
| `retailKiosk` | 1080×1920 | touch, tapTarget 60, viewing 80cm, attention 25s |
| `printQRPanel` | 620×874 | no interaction/motion/audio, safeArea 36 all round, attention 40s |

The UI also has a pre-shrunk `retailKioskShrunk` (70×70 in the demo; commits mention 150×150 earlier) for the degradation demo.

## 5. How the engine works

### Pipeline
`AdSpec → buildGraph → ExperienceGraph` · `SurfaceProfile → resolveContext → Context` → `generateCandidates` (5) → `scoreCandidate` each → highest `overall` wins (ties go to the earliest = `vertical-stack`) → `ResolvedLayout` (`elements[]` with `x/y/width/height/visible/role`) + `DecisionTrace`.

### Experience Graph (`graph.ts`)
`GraphNode extends AdElement` with an `edges` array. Edge types: `proximity | alignment | exclusion`. Rules keyed on role only:
- **Rule P**: every `secondary` ↔ every `action` gets a *proximity* edge (price belongs beside CTA).
- **Rule X**: every `hero` ↔ every `branding` gets an *exclusion* edge.
- `alignment` is typed but never derived.
Edges are now consumed by the `adjacencyFit` sub-score (added in Phase 7). Placement itself still doesn't read them.

### Context Engine (`context.ts`), six flags with named thresholds
- `aspectRatioClass`: `≤ 0.8` tall · `≥ 1.25` wide · else square
- `isTouchInteractive`: `touchOnly` or `interaction === "touch"`
- `isFarViewing`: `"far"` or numeric ≥ 100 cm
- `attentionBudget`: `≤ 3s` short · `≥ 15s` long · else medium (default medium)
- `hasAudio`, `allowsMotion`: default false
`hasAudio`, `allowsMotion` and `minTextSize` are populated/validated but **never consumed** downstream (found in `ANALYSIS.md`, not listed in the README limitations).

### Five strategies (`candidates.ts`)
| strategy | geometry |
|---|---|
| `vertical-stack` | one column, top→bottom. Safe default and the tie-break winner |
| `horizontal-split` | one row, left→right |
| `grid` | `ceil(√n)` columns, each element centred in its cell |
| `overlay-safe-margins` | four corners, capped at 4 elements (always drops the 5th) |
| `emergency-fit` | last resort. Relaxes minimums to a tiny floor so something renders when the other four hard-fail |

All go through one shared `placeElementsInOrder(requests, box, layoutFn, growAxes)`. Order is `priority` asc then `id` asc.

**Cascade (per element, in priority order):** ask the strategy for a slot → shrink to fit (`min(preferred, slot)`) → placed only if not below `min`, inside the box and non-overlapping. Otherwise a cascade flag is set and this element **and every lower-priority one are dropped** (strictly monotone). Shrink and drop notes feed the trace.

**Sizing (`toRequest`):**
- Text/button with `text` is sized from real `measureText()` (`fontSize`, line height ×1.3). Width is `max(measured, min.width, preferredSize.width)`, so the "DIOR" logo doesn't collapse.
- Far viewing: text/button preferred ×1.3.
- Touch + clickable: preferred ×1.15.
- Clickable with `minTapTarget`: preferred raised to at least `minTapTarget` on both axes (this fix moved robustness ~65% → ~87%).
- `min` is never scaled.

**Grow into slack:** each strategy declares which axes are safe to grow (`GrowAxes`). `vertical-stack` grows width only, `horizontal-split` height only, `grid` both, `overlay-safe-margins` none. Cap `GROWTH_CAP_FACTOR = 1.4` × the element's own preferred; hero role `HERO_GROWTH_CAP_FACTOR = 1.9`. `brandRules.locked` elements never grow. Clickable elements may grow in width but not height (height growth had produced a squat "blob" CTA on `mobileLandscape`).

**Hero handling (`heroScale`):** for the hero role a single scale factor drives both axes, so the resolved box's aspect ratio always equals the photo's (~0.756). Result: zero void and zero vertical crop under `object-fit: cover`, measured across 200 seeded surfaces plus the 5 real ones.

**Centring:** `centerAlongAxis` centres a cascading block so leftover space becomes even margins rather than one big gap. `grid` centres each element in its cell (`centerInCells`). `horizontal-split` and `emergency-fit` rows also centre on the free axis (`centerFreeAxis`). `vertical-stack`'s free axis (width) stays left-aligned on purpose.

**Emergency-fit (final form):**
- `visibility:"always"` elements relax to `EMERGENCY_MIN_SIZE = 24×16`.
- The hero relaxes to `EMERGENCY_HIGHLIGHT_SIZE` ≈ 27×36 (portrait, ≈0.75).
- Degradable/decorative elements relax to `EMERGENCY_SECONDARY_SIZE = 16×10`, smaller than the always-floor by design, and only claim genuine leftover space, so a fine-print price can never dwarf the message.
- Wins only when the other four have all hard-failed to 0.
- On a mathematically unsatisfiable surface (smaller than 24×16) it never throws: all five candidates score 0 and zero elements are visible.

### Scoring (`scoring.ts`), weights currently in code
```
constraintViolations 0.27   priorityPreservation 0.22   compositionCohesion 0.16
tapTargetCompliance  0.11   adjacencyFit         0.08   contextFit          0.07
visualBalance        0.06   renderCost           0.03        (sum = 1.00)
```
(ARCHITECTURE.md §4 still quotes the older 6-weight table 0.32/0.28/0.14/0.10/0.10/0.06. That section is stale on this point.)

| sub-score | idea |
|---|---|
| `constraintViolations` | 100 − 25 per soft violation (outside safe area, below `minSize`, locked-but-resized, clickable below `minTapTarget`) |
| `priorityPreservation` | % of priority-weighted content still visible |
| `tapTargetCompliance` | % of visible clickables meeting `minTapTarget` on both axes |
| `contextFit` | starts at 60. Shape bonus (wide → horizontal-split +20, grid +10, vertical −15; tall mirror; square → grid/overlay +10). Short attention and far viewing reward fewer visible elements. Touch penalises overlay-safe-margins −10 |
| `visualBalance` | 0.65 × centroid-centred + 0.35 × area evenness. Provably indifferent to cluster-vs-spread |
| `renderCost` | fewer/larger elements are cheaper (tie-breaker) |
| `adjacencyFit` | Euclidean centre distance of each proximity-edge pair, normalised by box diagonal. Pairs with a dropped member are skipped (neutral 100) |
| `compositionCohesion` | sum of element areas ÷ bounding-box footprint of all visible elements. Kills the "scattered corners with a dead void" arrangement without penalising legitimate full-bleed banners |

**Hard-fail → `overall = 0`:** no visible elements; any overlap; any element out of surface bounds; a `visibility:"always"` element dropped; priority inversion. `scoreCandidate` is pure (a determinism test asserts identical output twice).

`overlay-safe-margins` no longer wins on any of the 5 real surfaces (regression test).

### Trace and explainability (`trace.ts`)
`buildTrace` produces per-candidate scores, the winner, and per-element notes for the winner only, for example `logo: wanted 96×32, placed at 80×32 → shrunk 17%`, `headline: wanted 411×96, grew to 411×134 (+40%) — extra room available` and `price: dropped — priority 4, no room left once every higher-priority element was placed`. `explainLoss()` finds the sub-score with the widest gap between loser and winner and names it with the point delta. Adding a new sub-score only needed one table row there.

## 6. Demo app (views, from the top nav)

1. **Single surface** (default): surface picker (5 samples + shrunk kiosk + anything loaded from Stress Lab), Device frame (Auto/Clean/Phone/TV, chosen by aspect ratio, TV pillarboxes non-landscape content), Debug overlay toggle, context-flag header. Below it: the **Degradation slider** (live re-resolve as width/height shrink, with a **Fluid Stress Test** sin/cos sweep), then the **Layout Debugger**, **Layout Counterfactuals** (click a loser for the generated explanation) and **Layout Health Check** (pass/warn/fail on the sub-scores).
2. **Side-by-side**: all 5 surfaces resolved at once in device frames.
3. **Stress Lab**: 200 randomized surfaces (100–3840 × 120–2400, plus a 25% chance of an extreme like 320×50, 100×600, 3840×400, 120×2000), tiered pass/degraded/failed, a Conclusion panel bucketing degraded entries by root cause, click-through into Single surface.
4. **Self-Healing**: long headline + invalid hero `src` + empty logo `src` + 240×260 touch panel, with an optional longer German CTA (EN/DE toggle). Shows naive vs recovered, a "re-optimizing" beat, and the debugger. Recovers via `grid` with 4/4 visible and the CTA kept.
5. **Naive vs Smart**: uniform-scale, overlap-allowed baseline vs the real resolver.
6. **Unknown surface**: form → real `defineSurface()` → same pipeline. Errors are shown verbatim, and an impossibly small surface is explained rather than showing a bare "0/5".

## 7. Stress Lab methodology and results
- `generateRandomSurfaces(count, rng)`: valid profiles via `defineSurface` (touchOnly always gets a `minTapTarget`); seeded `mulberry32` available for reproducibility, the UI uses `Math.random`.
- `runStressTest`: for each surface, real `resolveContext → resolveLayout`, then `findInvariantViolation()`, a bounds+overlap check written **independently of scoring.ts** as a real cross-check.
- Tiers: **failed** = invariant actually broken; **degraded** = score < `DEGRADED_SCORE_THRESHOLD` (70); **passed** otherwise.
- Latest documented runs: 163/37/0 (81.5%) after the Phase 8 audit fix; 168/32/0 after §7; **174/26/0 (87.0%)** in the last live browser run. Typical band is 80–87%. **`failed` is 0 in every run.**
- All degraded entries are categorised "valid but sparse": no "nothing fits" and no "always-element dropped" cases remain, and every degraded surface now shows all 5 elements (small ones).
- Robustness history: ~50% → ~65% (`contextFit`) → ~87% (minTapTarget sizing fix) → ~90% (`emergency-fit`) → ~84% (`adjacencyFit`) → ~82.5% (`compositionCohesion` + placement fixes) → 81.5% (audit) → ~87% (hero AR + centring). The robustness % is a *quality* bar, not a correctness bar.

## 8. Development history (the "phases")
Built by driving Claude Code with hand-written phase prompts (`phase1.md`–`phase7.md`); a human reviewed, tested and gated each phase.
1. **Phase 1**: Vite/React/TS setup, `spec.ts`, `surfaces.ts`, `sample-data.ts`, validation and tests.
2. **Phase 2**: Experience Graph, Context Engine, simple greedy resolver, minimal React render (`PHASE2-MANUAL-STEPS.md`: no manual steps needed).
3. **Phase 3**: candidate generation + deterministic scoring, priority degradation, surface-picker UI, `DecisionTrace`.
4. **Phase 4**: the three explainability panels.
5. **Phase 5**: Stress Lab, Self-Healing, Side-by-side, Naive-vs-Smart, Degradation slider, Unknown-surface, text measurement.
6. **Phase 6**: README + ARCHITECTURE docs + final regression sweep.
7. **Phase 7**: `adjacencyFit` (graph edges into scoring), fix corner-spreading, the unsatisfiable-surface test (later joined by `compositionCohesion`).
8. **Phase 8 (`PHASE8-AUDIT.md`, no prompt file)**: a strict "panel judge" audit of the live app.

Later work (from the git log): UI theming (Nykaa), device frames, debug overlay, Fluid Stress Test, `contextFit`, `emergency-fit`, `compositionCohesion`, and hero sizing.

### Phase 8 audit findings (all measured live)
| # | Finding | Status |
|---|---|---|
| 1 | Critical: `emergency-fit` let price/logo balloon to 76% of a 320×50 banner while the image/headline/CTA were dots | Fixed (then refined in §7 to a tiny secondary floor so all 5 show) |
| 2 | The "highlight" had no size distinction at the emergency floor | Fixed (`EMERGENCY_HIGHLIGHT_SIZE`, image ≈ 69–72% of area) |
| 3 | Debugger label could read "grew to … (+-92%)" (per-axis verdict vs area %) | Fixed, now area-driven, with a property test |
| 4 | On `grid`-winning wide surfaces the image's lead over the headline is thin (39.8% vs 29%) because `grid` isn't role-aware | Documented, not fixed |
| 5 | Decorative "nykaa" corner badge can collide with real content | Documented, not fixed (breaks presentation/resolver separation) |
| 6 | Hero photo void / vertical crop → `heroScale`, `centerInCells`, `centerFreeAxis` | Fixed |
| 7 | `mobileLandscape` now resolves to `horizontal-split` by a 1-point margin (84 vs 83) and drops price/logo (3/5) | Residual trade-off, documented |

Image share of ad area per real surface after the fixes: portrait 71.5%, landscape 39.8%, broadcast 45.7%, kiosk 82.4%, print 75.7%.

## 9. Bonus-point status (README's own table)
Implemented: semantic graph (4.1), importance tiers (4.2), candidate generation + scoring (4.3), context-aware adaptation (4.3-context), declarative constraints (4.4), degradation engine (4.5), Debugger (4.7), Counterfactuals (4.7b), Health Check (4.7c), perf-aware scoring (4.9), Stress Lab (4.10), self-healing (4.13), interactive-vs-passive strategy (4.14), side-by-side (4.16), language-aware adaptation (4.17, EN/DE), degradation slider (4.18), naive-vs-smart (4.19), unknown-surface input (4.20), text-measurement-aware layout.
Partial: brand-safe adaptation (4.6). Only `locked` and brand `minSize` are modelled.
Skipped: Canvas renderer (4.21; `render-dom.ts` is a stub, so renderer-agnosticism is by construction only) and the optional AI semantic parser (by design, roles/priorities are authored by hand).

## 10. Known limitations and gaps
- `contextFit` bonuses and scale factors (+20/−15, 1.3×, 1.15×) and the sub-score weights are hand-chosen, not calibrated.
- 5 fixed strategies. There is no search, packing or refinement, and `emergency-fit` is a floor, not a smarter layout.
- The robustness % is a quality bar (score ≥ 70), not correctness.
- Text measurement: Canvas `measureText` in the browser, linear approximation (`length × fontSize × 0.52`) in Node tests. This can make the browser and tests disagree on close ties.
- `alignment` edges are never derived. Placement doesn't read graph edges (only scoring does).
- One renderer (`SurfaceStage`). `overlay-safe-margins` caps at 4 elements.
- One hard-coded product; no catalog or CMS.
- `fitFontSize` is a single-pass approximation (cosmetic).
- `minTextSize`, `hasAudio`, `allowsMotion` are validated but never used in decisions (from `ANALYSIS.md`).
- `findInvariantViolation` re-checks bounds/overlap only, not dropped-always or priority inversion.
- `resolveLayout` has no explicit `valid`/`violations` flag on its result.
- `constraintViolations` is a flat −25 per violation, so showing more tiny content on emergency surfaces can slightly lower the score.
- Not deployed; the "time spent" line in the README is still a placeholder comment to fill in.
- **Stale docs:** README/ARCHITECTURE partly lag the code (weights table, "4 strategies" wording in places, older sample-ad description, `ANALYSIS.md` counting 74 `it` blocks against the 144 claimed, since loops generate tests). Trust `sample-data.ts`, `scoring.ts` and `candidates.ts` over the prose where they differ.

## 11. The supporting `.md` files
| File | Purpose |
|---|---|
| `README.md` | Submission README: what/why, setup, demo walkthrough, feature→rubric table, limitations, AI disclosure, bonus table |
| `ARCHITECTURE.md` | Full technical design (§1 flow, §2 graph, §3 context, §4 candidates+scoring incl. §4c grow, §4d emergency, §4e adjacency, §4g cohesion, §5 degradation, §6 TS design, §7 extensibility, §8 stress, §9 self-healing, §10 limits, §11 presentation chrome) |
| `PHASE8-AUDIT.md` | Adversarial live audit with before/after measurements and addenda §7 and §8 |
| `ANALYSIS.md` | Deep comparison against the four `flam demo` variants, with prioritised improvements |
| `Adaptive-Resizing-Logic.md` | How each of the 4 sibling projects decides layout, and what was (and deliberately wasn't) ported |
| `UI-Enhancement.md` | "Master prompt" for porting three UI features (device frames, debug overlay, fluid stress test) from a reference implementation |
| `artifact.md` | Original project plan (vision, architecture pillars A/B/…, positioning against Flam's platform) |
| `phase1.md`–`phase7.md` | The build-phase prompts |
| `PHASE2-MANUAL-STEPS.md` | Optional verification steps for Phase 2 |

## 12. How to run
```bash
cd flam_vajeeda
npm install
npm run dev        # http://localhost:5173
npm run test       # 144 tests, 11 suites
npm run build      # tsc --noEmit + vite build → dist/
npm run preview    # http://localhost:4173
```

## 13. Suggested next steps (from `ANALYSIS.md` and the audit)
1. Give `minTextSize` (and `hasAudio`/`allowsMotion`) a real consumer, mirroring `minTapTarget`, or list them as limitations.
2. Extend `findInvariantViolation` to also re-derive dropped-always and priority-inversion, and split "degraded" accordingly.
3. Add `valid`/`violations` to `ResolveResult`.
4. Make `grid` role-aware (bigger hero cell) to fix the thin image margin on wide surfaces.
5. Turn `spec.invalid-example.ts.txt` into an enforced `@ts-expect-error` typecheck fixture.
6. Deploy (Vercel) so the "resolve an unseen surface live" demo has a link.
7. Lower priority: a Canvas/SVG second renderer, property-based testing with shrinking, and a recalibrated `constraintViolations` penalty for the emergency floor.
